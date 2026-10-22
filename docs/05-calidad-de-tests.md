# Frente 5: Calidad de Tests, Gates de Seguridad & Verificación en Vivo

## Resumen Ejecutivo del Frente

La auditoría no se limitó a una inspección estática del código; incluyó la ejecución real y exhaustiva de la suite de pruebas automatizadas, el detector de carreras (go test -race), los scripts de gates de seguridad de CI y el monitoreo del consumo de recursos del sistema operativo. Se descubrieron fallas reproducibles de alta severidad: saturación masiva de sockets de red en estado TIME_WAIT que agota la totalidad de puertos efímeros del sistema operativo (16,236 sockets) provocando errores EADDRNOTAVAIL, condiciones de carrera detectadas por el compilador en TaskManager.FailTask, sincronización frágil mediante time.Sleep de decenas de segundos en tests HTTP, y scripts de validación de CI que utilizan expresiones regulares eludibles mediante saltos de línea.

### Métricas del Frente
- **Total Hallazgos**: 4
- **Distribución de Severidad**: 2 Críticos (CRITICAL), 1 Medios (MEDIUM), 1 Bajos (LOW)

---

## Invariantes de Arquitectura & Seguridad Rotos

1. **Determinismo y Aislamiento de Recursos en Tests**: Los tests automatizados no deben depender de retardos temporales arbitrarios (time.Sleep) ni saturar recursos finitos del host (descriptores de archivos, puertos efímeros).
2. **Cero Tolerancia a Condiciones de Carrera**: El código debe pasar la totalidad de las suites bajo 'go test -race' sin advertencias de colisión de lectura/escritura en canales o estructuras de datos compartidas.
3. **Robustez en Gates de Seguridad Automatizados**: Los análisis estáticos de seguridad ejecutados en CI no deben basarse en expresiones regulares línea por línea vulnerables a formatos alternativos, sino en analizadores sintácticos reales (go/ast o semgrep).

---

## Tabla Resumen de Hallazgos

| ID | Severidad | Título | Archivo / Componente Principal |
|---|---|---|---|
| [F5-01](#f5-01) | **CRITICAL** 🔴 | Agotamiento de puertos efímeros del SO (16,236 sockets en TIME_WAIT) en TestServer_SetSigner_ConcurrentRaceFree | `integration/agent/server_mesh_test.go:565-587` |
| [F5-02](#f5-02) | **CRITICAL** 🔴 | Condición de carrera real reproducible en TaskManager.FailTask bajo go test -race | `pkg/server/task/manager.go:371-402` |
| [F5-03](#f5-03) | **MEDIUM** 🟡 | Sincronización por time.Sleep (>30s acumulados) en tests HTTP y 0% de tests unitarios en pkg/server/webhook | `pkg/server/http/retry_terminates_test.go` |
| [F5-04](#f5-04) | **LOW** 🔵 | Expresiones regulares en scripts/security-gates.sh vulnerables a llamadas multilínea o variables | `scripts/security-gates.sh:18-24` |

---

## Análisis Detallado por Hallazgo

### <a id="f5-01"></a>[F5-01] Agotamiento de puertos efímeros del SO (16,236 sockets en TIME_WAIT) en TestServer_SetSigner_ConcurrentRaceFree

- **Severidad**: **CRITICAL** 🔴
- **Ubicación en Código**: `integration/agent/server_mesh_test.go:565-587`

#### ⚠️ Invariante Roto y Causa Raíz
Verificado en vivo: 4 goroutines llaman a client.Get(ts.URL + "/health") sin pausa durante 2 segundos y cierran resp.Body sin drenarlo, agotando los 16,384 puertos efímeros en macOS y rompiendo TestEndpointsAbsentInProduction y TestTLS_* con EADDRNOTAVAIL.

#### 🛠️ Remediación Arquitectónica Sistémica
Drenar resp.Body con io.Copy(io.Discard, resp.Body) y agregar un throttle o probar SetSigner en memoria.

#### 🔍 Evidencia de Código
```go
resp, err := client.Get(ts.URL + "/health")
if err == nil { resp.Body.Close() }
```

---

### <a id="f5-02"></a>[F5-02] Condición de carrera real reproducible en TaskManager.FailTask bajo go test -race

- **Severidad**: **CRITICAL** 🔴
- **Ubicación en Código**: `pkg/server/task/manager.go:371-402 | pkg/server/http/silent_node_test.go:400-445`

#### ⚠️ Invariante Roto y Causa Raíz
Verificado en vivo al correr go test -race ./pkg/server/http: TestSilentNodeConcurrentEOFAndSweepSingleTransitionAndEvent falla con expected 1 EventError, got 0 porque FailTask suelta task.mu antes de EmitEvent y otra goroutine cierra los canales de suscriptores en TransitionTo(Failed).

#### 🛠️ Remediación Arquitectónica Sistémica
Hacer atómica la asignación de error, emisión de EventError y transición a Failed bajo el mismo lock.

#### 🔍 Evidencia de Código
```go
task.mu.Unlock() // G2 entra aquí y cierra canales en TransitionTo antes de que G1 haga EmitEvent
```

---

### <a id="f5-03"></a>[F5-03] Sincronización por time.Sleep (>30s acumulados) en tests HTTP y 0% de tests unitarios en pkg/server/webhook

- **Severidad**: **MEDIUM** 🟡
- **Ubicación en Código**: `pkg/server/http/retry_terminates_test.go | pkg/server/webhook/`

#### ⚠️ Invariante Roto y Causa Raíz
Los tests de pkg/server/http usan decenas de time.Sleep fijos (500ms-2500ms) y pkg/server/webhook no tiene ningún archivo _test.go.

#### 🛠️ Remediación Arquitectónica Sistémica
Usar canales/tickers deterministas y agregar tests unitarios para pkg/server/webhook.

#### 🔍 Evidencia de Código
```go
// pkg/server/webhook: ? [no test files]
```

---

### <a id="f5-04"></a>[F5-04] Expresiones regulares en scripts/security-gates.sh vulnerables a llamadas multilínea o variables

- **Severidad**: **LOW** 🔵
- **Ubicación en Código**: `scripts/security-gates.sh:18-24`

#### ⚠️ Invariante Roto y Causa Raíz
Los gates S6 y S1-Literal usan grep línea por línea buscando literales exactos, eludibles con llamadas multilínea o variables.

#### 🛠️ Remediación Arquitectónica Sistémica
Complementar con un analizador basado en go/ast.

#### 🔍 Evidencia de Código
```go
SHELL_PATTERN='exec\.Command.*"sh".*"-c"'
```

---

## Hoja de Ruta Específica del Frente

- **P0 (Inmediato)**: Resolver la condición de carrera en TaskManager.FailTask que produce fallas intermitentes en TestSilentNodeConcurrentEOFAndSweepSingleTransitionAndEvent.
- **P1 (Corto Plazo)**: Modificar TestServer_SetSigner_ConcurrentRaceFree en integration/agent/ agregando rate-limiting o reutilización de conexiones en el cliente HTTP para erradicar la saturación de puertos efímeros (TIME_WAIT).
- **P2 (Estructural)**: Reemplazar los time.Sleep acumulados (>30s) en pkg/server/http/ por mecanismos reactivos basados en canales y sync.WaitGroup. Agregar cobertura de tests unitarios para pkg/server/webhook/ (actualmente 0%). Migrar scripts/security-gates.sh a verificaciones basadas en AST.

