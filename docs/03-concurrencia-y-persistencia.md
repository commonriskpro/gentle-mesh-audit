# Frente 3: Concurrencia, Scheduler & Persistencia SQLite WAL

## Resumen Ejecutivo del Frente

Gentle-Mesh utiliza una arquitectura de persistencia híbrida basada en SQLite con Write-Ahead Logging (WAL) para metadatos y certificados, combinada con archivos de append-only JSON Lines (.jsonl) para el streaming de eventos de tareas y auditoría. La concurrencia está coordinada mediante BranchLockManager, TerritoryScheduler y TaskManager. La auditoría identificó fallas severas en la gestión de conexiones SQLite (ausencia de SetMaxOpenConns(1) y doble pool sin mutex compartido sobre gentle-mesh.db), liberación prematura de locks de rama que produce ejecuciones paralelas destructivas, inversión de prioridad FIFO y pérdida de eventos SSE terminales.

### Métricas del Frente
- **Total Hallazgos**: 12
- **Distribución de Severidad**: 2 Críticos (CRITICAL), 5 Altos (HIGH), 4 Medios (MEDIUM), 1 Bajos (LOW)

---

## Invariantes de Arquitectura & Seguridad Rotos

1. **Single-Writer SQLite Invariant**: SQLite en modo WAL soporta múltiples lectores concurrentes pero estrictamente un único escritor a la vez. En Go, database/sql requiere db.SetMaxOpenConns(1) o sincronización estricta mediante un mutex a nivel de aplicación para prevenir SQLITE_BUSY (database is locked).
2. **Sincronía de Liberación de Locks de Recursos**: El lock de exclusividad sobre una rama de git jamás debe liberarse hasta que el subproceso del runner haya finalizado por completo y confirmado su estado de salida.
3. **Invarianza FIFO en Planificación de Tareas**: TerritoryScheduler debe encolar y despachar tareas respetando estrictamente el orden de llegada FIFO para ramas idénticas, sin permitir que tareas recién recibidas se adelanten a tareas pendientes.
4. **Entrega Garantizada de Eventos Terminales**: Los canales de streaming de Server-Sent Events (SSE) deben garantizar que los eventos de cierre (EventCompletion o EventError) sean entregados a los suscriptores antes de destruir el canal o marcar la tarea como desvinculada.

---

## Tabla Resumen de Hallazgos

| ID | Severidad | Título | Archivo / Componente Principal |
|---|---|---|---|
| [F3-01](#f3-01) | **CRITICAL** 🔴 | Pool SQLite sin SetMaxOpenConns(1), PRAGMA busy_timeout no heredado y doble pool compitiendo por gentle-mesh.db | `pkg/server/store/sqlite.go:30-58` |
| [F3-02](#f3-02) | **CRITICAL** 🔴 | Doble liberación de lock de rama y ejecución solapada al cancelar tareas (POST /v1/tasks/{id}/cancel) | `pkg/server/http/handlers.go:633-646` |
| [F3-03](#f3-03) | **HIGH** 🟠 | Bypass de la cola FIFO e inversión de prioridad en TerritoryScheduler.Schedule | `pkg/server/http/scheduler.go:198-223` |
| [F3-04](#f3-04) | **HIGH** 🟠 | Reconciliación rota en Crash Recovery: sobrescribe tareas completadas en .jsonl como FAILED | `pkg/server/task/manager.go:94-116, 660-675` |
| [F3-05](#f3-05) | **HIGH** 🟠 | Corrupción permanente de logs .jsonl tras escritura parcial en un crash | `pkg/server/task/logger.go:33-40, 110-135` |
| [F3-06](#f3-06) | **HIGH** 🟠 | Pérdida silenciosa de eventos terminales (EventCompletion / EventError) en clientes SSE lentos | `pkg/server/task/task.go:212-225, 395-408, 640-650` |
| [F3-07](#f3-07) | **HIGH** 🟠 | Tareas huérfanas eternas cuando un worker muere antes de emitir su primer evento SSE | `pkg/server/runner/mesh.go:128-142` |
| [F3-08](#f3-08) | **MEDIUM** 🟡 | Inconsistencia case-sensitive en BranchLockManager vs detección de conflictos territoriales | `pkg/server/registry/lock.go:23-30` |
| [F3-09](#f3-09) | **MEDIUM** 🟡 | Creación innecesaria de tarea y archivo .jsonl antes de verificar Idempotency-Key | `pkg/server/http/handlers.go:434-456` |
| [F3-10](#f3-10) | **MEDIUM** 🟡 | Canales pendingQueries quedan abiertos al terminar o cancelar una tarea | `pkg/server/task/task.go:415-465` |
| [F3-11](#f3-11) | **MEDIUM** 🟡 | TaskManager.Close() descarta actualizaciones finales en SQLite durante el apagado | `pkg/server/task/manager.go:506-538, 705-720` |
| [F3-12](#f3-12) | **LOW** 🔵 | Doble escritura redundante en syncTaskToStore y falta de cs.mu en UpdateReceipt | `pkg/server/task/manager.go:742-745` |

---

## Análisis Detallado por Hallazgo

### <a id="f3-01"></a>[F3-01] Pool SQLite sin SetMaxOpenConns(1), PRAGMA busy_timeout no heredado y doble pool compitiendo por gentle-mesh.db

- **Severidad**: **CRITICAL** 🔴
- **Ubicación en Código**: `pkg/server/store/sqlite.go:30-58 | cmd/gentle-mesh/main.go:183-205`

#### ⚠️ Invariante Roto y Causa Raíz
1) PRAGMA busy_timeout=5000 solo se ejecuta en una conexión del pool; nuevas conexiones nacen con busy_timeout=0. 2) Se omitió db.SetMaxOpenConns(1). 3) main.go:183 abre un segundo sql.Open sobre el mismo gentle-mesh.db sin WAL ni writeMu compartido.

#### 🛠️ Remediación Arquitectónica Sistémica
Pasar PRAGMAs en el DSN, aplicar db.SetMaxOpenConns(1) y compartir un único *sql.DB entre todos los stores.

#### 🔍 Evidencia de Código
```go
rawDB, rawErr := sql.Open("sqlite", dbPath) // Segundo pool sin WAL ni writeMu
```

---

### <a id="f3-02"></a>[F3-02] Doble liberación de lock de rama y ejecución solapada al cancelar tareas (POST /v1/tasks/{id}/cancel)

- **Severidad**: **CRITICAL** 🔴
- **Ubicación en Código**: `pkg/server/http/handlers.go:633-646 | pkg/server/http/scheduler.go:338-348, 426-434`

#### ⚠️ Invariante Roto y Causa Raíz
handleTaskCancel llama inmediatamente a ReleaseLock y s.scheduler.Drain() mientras el runner cancelado sigue corriendo, despachando la siguiente tarea sobre la misma rama en paralelo.

#### 🛠️ Remediación Arquitectónica Sistémica
Eliminar ReleaseLock y Drain() de handlers.go; solo TerritoryScheduler.complete(mt) debe liberar el lock al terminar el runner.

#### 🔍 Evidencia de Código
```go
_ = s.registry.Locks().ReleaseLock(t.Request.GitRepo, t.Request.GitBranch, id)
s.scheduler.Drain()
```

---

### <a id="f3-03"></a>[F3-03] Bypass de la cola FIFO e inversión de prioridad en TerritoryScheduler.Schedule

- **Severidad**: **HIGH** 🟠
- **Ubicación en Código**: `pkg/server/http/scheduler.go:198-223`

#### ⚠️ Invariante Roto y Causa Raíz
Schedule(mt) solo verifica conflictos contra RunningManifest() e ignora s.queue, permitiendo que tareas nuevas salten por delante de tareas ya encoladas.

#### 🛠️ Remediación Arquitectónica Sistémica
Verificar si s.queue contiene tareas pendientes antes de permitir el despacho directo.

#### 🔍 Evidencia de Código
```go
conflict := s.findConflictLocked(mt) // Solo mira RunningManifest(), ignora s.queue
```

---

### <a id="f3-04"></a>[F3-04] Reconciliación rota en Crash Recovery: sobrescribe tareas completadas en .jsonl como FAILED

- **Severidad**: **HIGH** 🟠
- **Ubicación en Código**: `pkg/server/task/manager.go:94-116, 660-675`

#### ⚠️ Invariante Roto y Causa Raíz
rehydrateTask lee el archivo .jsonl solo para restaurar eventSeq, ignorando si el último evento ya fue EventCompletion. Si cayó antes de actualizar SQLite, la marca como FAILED.

#### 🛠️ Remediación Arquitectónica Sistémica
Inspeccionar el último evento del .jsonl durante rehydrateTask para reconciliar el estado terminal en SQLite.

#### 🔍 Evidencia de Código
```go
if t.Status == protocol.TaskStatusRunning { _ = m.FailTask(t.TaskID, OrphanedTaskCode, OrphanedTaskMessage) }
```

---

### <a id="f3-05"></a>[F3-05] Corrupción permanente de logs .jsonl tras escritura parcial en un crash

- **Severidad**: **HIGH** 🟠
- **Ubicación en Código**: `pkg/server/task/logger.go:33-40, 110-135`

#### ⚠️ Invariante Roto y Causa Raíz
NewJSONLLogger abre con os.O_APPEND sin truncar una última línea parcial sin salto de línea tras un crash, concatenando el siguiente JSON en la misma línea rota.

#### 🛠️ Remediación Arquitectónica Sistémica
Truncar hasta el último salto de línea válido al abrir NewJSONLLogger.

#### 🔍 Evidencia de Código
```go
f, err := os.OpenFile(path, os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0o644)
```

---

### <a id="f3-06"></a>[F3-06] Pérdida silenciosa de eventos terminales (EventCompletion / EventError) en clientes SSE lentos

- **Severidad**: **HIGH** 🟠
- **Ubicación en Código**: `pkg/server/task/task.go:212-225, 395-408, 640-650`

#### ⚠️ Invariante Roto y Causa Raíz
Si el buffer de 64 eventos se llena, el select no bloqueante descarta incluso EventCompletion/EventError e inmediatamente cierra el canal sub.

#### 🛠️ Remediación Arquitectónica Sistémica
Garantizar entrega obligatoria para eventos terminales (EventStatus, EventCompletion, EventError).

#### 🔍 Evidencia de Código
```go
select { case sub <- *evt: default: } // Descarta evento terminal antes de close(sub)
```

---

### <a id="f3-07"></a>[F3-07] Tareas huérfanas eternas cuando un worker muere antes de emitir su primer evento SSE

- **Severidad**: **HIGH** 🟠
- **Ubicación en Código**: `pkg/server/runner/mesh.go:128-142 | pkg/server/task/manager.go:472-500`

#### ⚠️ Invariante Roto y Causa Raíz
nodeAttributionSink asigna t.NodeID recién en el primer evento SSE. Si el worker muere antes del primer evento, FailTasksOfNode no encuentra t.NodeID.

#### 🛠️ Remediación Arquitectónica Sistémica
Atribuir t.NodeID inmediatamente al seleccionar el nodo antes de iniciar la llamada de red.

#### 🔍 Evidencia de Código
```go
s.once.Do(func() { s.attributor.AttributeNode(s.nodeID) })
```

---

### <a id="f3-08"></a>[F3-08] Inconsistencia case-sensitive en BranchLockManager vs detección de conflictos territoriales

- **Severidad**: **MEDIUM** 🟡
- **Ubicación en Código**: `pkg/server/registry/lock.go:23-30 | pkg/protocol/federation.go:174-182`

#### ⚠️ Invariante Roto y Causa Raíz
ClashesWith usa strings.EqualFold pero makeBranchKey en lock.go es case-sensitive.

#### 🛠️ Remediación Arquitectónica Sistémica
Normalizar la rama en makeBranchKey.

#### 🔍 Evidencia de Código
```go
branch: strings.TrimSpace(branch)
```

---

### <a id="f3-09"></a>[F3-09] Creación innecesaria de tarea y archivo .jsonl antes de verificar Idempotency-Key

- **Severidad**: **MEDIUM** 🟡
- **Ubicación en Código**: `pkg/server/http/handlers.go:434-456 | pkg/server/registry/idempotency.go:19-58`

#### ⚠️ Invariante Roto y Causa Raíz
handleTaskCreate llama a CreateTask antes de RecordOrGet en Idempotency y la caché vive solo en RAM.

#### 🛠️ Remediación Arquitectónica Sistémica
Verificar Idempotency-Key antes de CreateTask y persistir llaves en SQLite.

#### 🔍 Evidencia de Código
```go
t, err := s.taskManager.CreateTask(req)
```

---

### <a id="f3-10"></a>[F3-10] Canales pendingQueries quedan abiertos al terminar o cancelar una tarea

- **Severidad**: **MEDIUM** 🟡
- **Ubicación en Código**: `pkg/server/task/task.go:415-465`

#### ⚠️ Invariante Roto y Causa Raíz
TransitionTo cierra t.subscribers pero olvida cerrar t.pendingQueries.

#### 🛠️ Remediación Arquitectónica Sistémica
Cerrar y limpiar t.pendingQueries al entrar en estado terminal.

#### 🔍 Evidencia de Código
```go
// TransitionTo no limpia t.pendingQueries
```

---

### <a id="f3-11"></a>[F3-11] TaskManager.Close() descarta actualizaciones finales en SQLite durante el apagado

- **Severidad**: **MEDIUM** 🟡
- **Ubicación en Código**: `pkg/server/task/manager.go:506-538, 705-720`

#### ⚠️ Invariante Roto y Causa Raíz
Close() marca m.closed = true antes de m.runnerWg.Wait(), haciendo que syncTaskToStore aborte el guardado final.

#### 🛠️ Remediación Arquitectónica Sistémica
Esperar a m.runnerWg.Wait() antes de marcar m.closed = true.

#### 🔍 Evidencia de Código
```go
m.closed = true
m.mu.Unlock()
m.runnerWg.Wait()
```

---

### <a id="f3-12"></a>[F3-12] Doble escritura redundante en syncTaskToStore y falta de cs.mu en UpdateReceipt

- **Severidad**: **LOW** 🔵
- **Ubicación en Código**: `pkg/server/task/manager.go:742-745 | pkg/receipt/chain.go:534-555`

#### ⚠️ Invariante Roto y Causa Raíz
syncTaskToStore ejecuta UpdateStatus seguido de SaveTask (que actualiza las mismas columnas), y UpdateReceipt no toma cs.mu.

#### 🛠️ Remediación Arquitectónica Sistémica
Eliminar UpdateStatus redundante y proteger UpdateReceipt con cs.mu.

#### 🔍 Evidencia de Código
```go
_ = st.UpdateStatus(ctx, ...)
_ = st.SaveTask(ctx, record)
```

---

## Hoja de Ruta Específica del Frente

- **P0 (Inmediato)**: Configurar db.SetMaxOpenConns(1) y PRAGMA busy_timeout=5000 en el DSN SQLite, unificando el acceso a gentle-mesh.db bajo un único *sql.DB compartido. Eliminar ReleaseLock() y scheduler.Drain() de handleTaskCancel para que la liberación del lock sea potestad exclusiva de complete(mt).
- **P1 (Corto Plazo)**: Reescribir TerritoryScheduler.Schedule para despachar tareas secuencialmente respetando la cola FIFO. Corregir la reconciliación en crash recovery para no marcar tareas completadas en .jsonl como FAILED. Usar buffers y timeouts deterministas para evitar pérdida de EventCompletion/EventError en clientes SSE.
- **P2 (Estructural)**: Asegurar que escrituras parciales en .jsonl se sincronicen atómicamente con fsync. Normalizar nombres de rama case-insensitive en BranchLockManager. Cerrar proactivamente canales pendingQueries al cancelar tareas. Forzar a TaskManager.Close() a esperar el flush de actualizaciones pendientes a SQLite.

