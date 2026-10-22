# Gentle-Mesh — Reporte de Auditoría Integral de Seguridad y Arquitectura

> **Auditoría Exhaustiva de 5 Frentes: Criptografía & PKI, Sandboxing & Red, Concurrencia SQLite WAL, Arquitectura de Módulos y Verificación en Vivo.**

## 📊 Ficha Técnica del Repositorio Auditado

- **Proyecto**: [gentle-mesh](https://github.com/Rafaeldelinares/gentle-mesh)
- **Commit Auditado**: `c3e900a` (Merge PR #143)
- **Stack**: Go `1.26.10` (`CGO_ENABLED=0`) · SQLite puro Go (`modernc.org/sqlite v1.59.0`)
- **Volumen**: 182 archivos Go · 24,449 LOC Producción · 47,603 LOC Tests (72,052 LOC totales, ratio 1.95x)
- **Veredicto Global**: **`NEEDS_CORRECTION`**

---

## 🎯 Resumen Ejecutivo de Hallazgos

Se identificaron un total de **50 hallazgos verificados directamente en código fuente**:

| Severidad | Cantidad | Impacto Principal |
| :--- | :---: | :--- |
| 🔴 **CRITICAL** | **12** | Escape de sandbox hacia git hooks del host, Path/Symlink Traversal con lectura arbitraria del host, SSRF sin restricciones, PKI fail-open tras rotación, doble liberación de branch locks y colisión `SQLITE_BUSY`. |
| 🟠 **HIGH** | **19** | Condiciones de carrera en transición de estados, pérdida silenciosa de eventos SSE bajo backpressure, OOM remoto por lecturas HTTP sin límite, derivación de `context.Background()` y degradación de failover multi-nodo. |
| 🟡 **MEDIUM** | **15** | Fuga de goroutines/canales en queries interactivas, CORS wildcard `*.ts.net`, contradicción en estados terminales de contratos y caché fantasma en federación. |
| 🔵 **LOW** | **4** | Restricciones RFC 5480 en ECDSA, doble escritura redundante en SQLite y regex frágiles en gates de seguridad. |

---

## ⚡ Resultados de Verificación en Vivo (Concurrencia & Linters)

1. `go vet ./...` ➔ **PASS** (compilación limpia en 182 archivos).
2. `bash scripts/security-gates.sh --test` ➔ **PASS** (pasa gates S1, S6 y DB-Artifact).
3. `go test -race ./pkg/server/http` ➔ ❌ **FAIL (Exit 1 · 34.1s)**:
   - Falla reproducible en `TestSilentNodeConcurrentEOFAndSweepSingleTransitionAndEvent`:
     ```text
     silent_node_test.go:435: expected exactly 1 EventError event, got 0
     FAIL github.com/gentleman-programming/gentle-mesh/pkg/server/http 34.110s
     ```
   - **Causa Raíz**: Ventana de carrera en `TaskManager.FailTask` (`pkg/server/task/manager.go:371-402`). Suelta el mutex tras setear `task.Error` pero antes de `EmitEvent`. Una segunda goroutine concurrente entra, ve `task.Error != nil`, y llama directamente a `TransitionTo(Failed)`, la cual cierra los canales de suscriptores antes de que la primera goroutine emita el evento.
4. `go test -race ./integration/agent/...` ➔ ❌ **FAIL (EADDRNOTAVAIL)**:
   - `TestServer_SetSigner_ConcurrentRaceFree` satura **16,236 sockets en TIME_WAIT** en macOS en sólo 2 segundos debido a un bucle no regulado de 4 goroutines que cierran `resp.Body` sin drenarlo (`io.Copy(io.Discard, resp.Body)`), agotando el pool de puertos efímeros del sistema operativo (16,384) y quebrando en cascada los tests TLS subsiguientes.

---

## 🧭 Hoja de Ruta de Remediación Priorizada

### Prioridad 0 (Inmediata · Riesgos Críticos)
1. **Cerrar Escape de Sandbox & Traversal** (`pkg/server/runner/gentleshell.go`, `pkg/settlement/evaluate.go`, `pkg/server/http/handlers.go`):
   - Montar `repoCache` y `~/.gentle-shell` en modo solo lectura (`--ro-bind`) en Bubblewrap.
   - Pasar incondicionalmente `-c core.hooksPath=/dev/null` en todas las llamadas `git` ejecutadas en el host por el worker.
   - Forzar validación de contención física con `filepath.EvalSymlinks` y `filepath.Rel` tanto en `absPath` como en `resolveWorkspacePath`.
2. **Reparar Fail-Open en PKI y Bloquear Secuestro de Identidad** (`pkg/server/store/certstore.go`, `pkg/server/http/handlers.go`):
   - Migrar la clave primaria de `node_certs` a `serial TEXT PRIMARY KEY` para que la rotación nunca purgue seriales históricos.
   - En `IsCertRevoked`, tratar certificados ausentes como fail-closed o mantener tabla inmutable `revoked_serials`.
   - Validar en `/v1/certs/enroll` que el `NodeID` no exista ni esté revocado previamente salvo rotación autenticada.
3. **Unificar Conexiones SQLite y Corregir Locks** (`pkg/server/store/sqlite.go`, `cmd/gentle-mesh/main.go`, `pkg/server/http/handlers.go`):
   - Compartir una única instancia `*sql.DB` con `SetMaxOpenConns(1)` y DSN PRAGMAs (`busy_timeout=5000&journal_mode=WAL`).
   - Eliminar `ReleaseLock` y `Drain()` de `handleTaskCancel`: el lock de rama solo debe liberarse de forma simétrica cuando el runner efectivamente termina en `TerritoryScheduler.complete(mt)`.
   - Hacer atómica la transición y emisión de error en `FailTask` bajo un único lock.

### Prioridad 1 (Corto Plazo · Robustez y Red)
4. **Mitigar SSRF y Denegación de Servicio en Webhooks & Federación** (`pkg/server/webhook/dispatcher.go`, `pkg/server/federation/manager.go`):
   - Restringir destinos salientes con `net.Dialer.Control` bloqueando loopback (127.0.0.1), redes privadas (RFC 1918) y metadata cloud (169.254.169.254).
   - Implementar pool de workers acotado para webhooks y envolver lecturas HTTP en `io.LimitReader` / `MaxBytesReader`.
   - Eliminar el fallback de CSPRNG a `time.Now().UnixNano()` en `generateSecureToken` y IDs de recibos.

### Prioridad 2 (Estructural & Mantenibilidad)
5. **Desacoplar Inversión DAG y Descomponer Monolitos** (`integration/agent/`, `cmd/gentle-mesh/main.go`):
   - Extraer los 3,241 LOC del servidor RFC-002 desde `integration/agent/` hacia `pkg/settlement/transport`.
   - Eliminar la dependencia de `pkg/shell/mesh.go` hacia `integration/agent`.
   - Descomponer `main.go` (2,462 LOC) en subcomandos modulares.

---

## 📂 Estructura de este Repositorio

```text
.
├── README.md                           # Este documento ejecutivo
├── index.html                          # Dashboard interactivo autónomo (abrir en navegador o GitHub Pages)
├── findings.json                       # Dataset completo estructurado con los 50 hallazgos
└── docs/                               # Informes técnicos detallados por frente
    ├── 01-criptografia-y-pki.md        # Frente 1: Cripto, JCS, Ed25519, mTLS y PKI (8 hallazgos)
    ├── 02-aislamiento-y-red.md         # Frente 2: Sandboxing, Runner, SSRF y Límites HTTP (19 hallazgos)
    ├── 03-concurrencia-y-persistencia.md# Frente 3: Concurrencia, SQLite WAL y Estados (12 hallazgos)
    ├── 04-arquitectura-y-deuda.md      # Frente 4: Arquitectura, DAG, Monolitos y Ciclo de Vida (7 hallazgos)
    └── 05-calidad-de-tests.md          # Frente 5: Tests en Vivo, Race Detector y Gates (4 hallazgos)
```

## 🖥️ Cómo visualizar el Dashboard Interactivo

El archivo `index.html` es completamente autónomo y no requiere servidores externos ni conexión a internet:
1. **Localmente**: Abrir `index.html` en cualquier navegador web moderno (doble clic o `open index.html`).
2. **Vía GitHub Pages**: Al habilitar GitHub Pages en este repositorio (rama `main`, raíz `/`), el dashboard estará disponible directamente en la web con filtros por severidad, búsqueda en tiempo real y código fuente.
