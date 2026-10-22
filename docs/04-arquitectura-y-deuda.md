# Frente 4: Arquitectura, Monolitos & Deuda Técnica

## Resumen Ejecutivo del Frente

Este frente analiza la topología modular, el acoplamiento entre capas, la adherencia a la jerarquía de dependencias (DAG), la higiene en el manejo de errores y la presencia de antipatrones arquitectónicos en Gentle-Mesh. El análisis descubrió un caso extremo de split-brain arquitectónico (3,241 LOC de servidor de producción ubicadas dentro del directorio de tests integration/agent/ e importadas por código de producción), dos godfiles monolíticos masivos (cmd/gentle-mesh/main.go con 2,462 LOC y pkg/server/http/handlers.go con 1,067 LOC), errores de base de datos silenciados sistemáticamente con _ = junto a la desactivación de linters en CI, y un peligroso fallback criptográfico a time.Now().UnixNano().

### Métricas del Frente
- **Total Hallazgos**: 7
- **Distribución de Severidad**: 2 Críticos (CRITICAL), 4 Altos (HIGH), 1 Medios (MEDIUM)

---

## Invariantes de Arquitectura & Seguridad Rotos

1. **Directed Acyclic Graph (DAG) e Inversión de Dependencias**: El código de producción en pkg/ jamás debe depender de paquetes ubicados en tests o integration/. Los módulos deben ordenarse en una jerarquía estricta sin referencias circulares ni dependencias espurias.
2. **Single Responsibility Principle (Límites de Monolitos)**: Los puntos de entrada (main.go) y los controladores HTTP deben limitarse a orquestar y despachar, delegando lógica de negocio y persistencia a submódulos cohesivos de menos de 500-800 LOC.
3. **Fail-Closed en Manejo de Errores**: Las operaciones de persistencia crítica jamás deben silenciarse con _ =. Toda falla en la base de datos o en la serialización debe propagarse y abortar la transacción de forma segura.
4. **Pureza Criptográfica de Entropía**: Generadores de identificadores únicos y claves jamás deben recurrir a estampas de tiempo o números pseudoaleatorios predecibles si crypto/rand falla; deben fallar de inmediato (panic/abort).

---

## Tabla Resumen de Hallazgos

| ID | Severidad | Título | Archivo / Componente Principal |
|---|---|---|---|
| [F4-01](#f4-01) | **CRITICAL** 🔴 | Split-Brain arquitectónico e inversión DAG: 3,241 LOC de servidor de producción dentro de integration/agent/ | `integration/agent/server.go:1-1552` |
| [F4-02](#f4-02) | **CRITICAL** 🔴 | Fallback criptográfico a time.Now().UnixNano() cuando falla crypto/rand.Read | `cmd/gentle-mesh/main.go:2223-2228` |
| [F4-03](#f4-03) | **HIGH** 🟠 | Monolitos Godfile: cmd/gentle-mesh/main.go (2,462 LOC) y pkg/server/http/handlers.go (1,067 LOC) | `cmd/gentle-mesh/main.go:1-2462` |
| [F4-04](#f4-04) | **HIGH** 🟠 | Errores de persistencia silenciados con _ = y regla G104 deshabilitada en CI | `pkg/server/task/manager.go:346, 579, 707-708` |
| [F4-05](#f4-05) | **HIGH** 🟠 | Fuga de conexión HTTP en runWorker por defer resp.Body.Close() antes de loop infinito | `cmd/gentle-mesh/main.go:964` |
| [F4-06](#f4-06) | **HIGH** 🟠 | Condición de carrera peligrosa en killProcessTree: envía SIGKILL tras 3s a un PID que el SO pudo reciclar | `pkg/server/runner/gentleshell.go:768-774` |
| [F4-07](#f4-07) | **MEDIUM** 🟡 | Handlers RFC-002 usan context.Background(), Server.Shutdown olvida webhookDispatcher.Stop() y time.After en loops | `integration/agent/server.go:533, 956+` |

---

## Análisis Detallado por Hallazgo

### <a id="f4-01"></a>[F4-01] Split-Brain arquitectónico e inversión DAG: 3,241 LOC de servidor de producción dentro de integration/agent/

- **Severidad**: **CRITICAL** 🔴
- **Ubicación en Código**: `integration/agent/server.go:1-1552 | pkg/shell/mesh.go:8 | cmd/test-harness/main.go:21`

#### ⚠️ Invariante Roto y Causa Raíz
El motor HTTP completo del protocolo RFC-002 vive en integration/agent/ y pkg/shell/mesh.go importa integration/agent, invirtiendo el DAG de paquetes.

#### 🛠️ Remediación Arquitectónica Sistémica
Extraer el transporte RFC-002 hacia pkg/ y unificar los servidores HTTP.

#### 🔍 Evidencia de Código
```go
import "github.com/gentleman-programming/gentle-mesh/integration/agent" // en pkg/shell/mesh.go
```

---

### <a id="f4-02"></a>[F4-02] Fallback criptográfico a time.Now().UnixNano() cuando falla crypto/rand.Read

- **Severidad**: **CRITICAL** 🔴
- **Ubicación en Código**: `cmd/gentle-mesh/main.go:2223-2228 | pkg/receipt/append.go:208-213 | pkg/shell/shell.go:317-319`

#### ⚠️ Invariante Roto y Causa Raíz
En generateSecureToken, newReceiptID y newContractID, si crypto/rand.Read falla, cae en un fallback usando time.Now().UnixNano() en vez de fallar cerrado.

#### 🛠️ Remediación Arquitectónica Sistémica
Eliminar todo fallback basado en reloj; devolver error o abortar si falla el CSPRNG.

#### 🔍 Evidencia de Código
```go
b[i] = charset[time.Now().UnixNano()%int64(len(charset))]
```

---

### <a id="f4-03"></a>[F4-03] Monolitos Godfile: cmd/gentle-mesh/main.go (2,462 LOC) y pkg/server/http/handlers.go (1,067 LOC)

- **Severidad**: **HIGH** 🟠
- **Ubicación en Código**: `cmd/gentle-mesh/main.go:1-2462 | pkg/server/http/handlers.go:1-1067`

#### ⚠️ Invariante Roto y Causa Raíz
main.go concentra 7 responsabilidades ortogonales en 2,462 líneas y handlers.go concentra todos los endpoints HTTP.

#### 🛠️ Remediación Arquitectónica Sistémica
Descomponer main.go por subcomando y handlers.go por dominio.

#### 🔍 Evidencia de Código
```go
// cmd/gentle-mesh/main.go: 2,462 LOC
```

---

### <a id="f4-04"></a>[F4-04] Errores de persistencia silenciados con _ = y regla G104 deshabilitada en CI

- **Severidad**: **HIGH** 🟠
- **Ubicación en Código**: `pkg/server/task/manager.go:346, 579, 707-708 | .github/workflows/ci.yml:68`

#### ⚠️ Invariante Roto y Causa Raíz
Las escrituras en SQLite y JSONL descartan errores con _ = y se excluyó G104 en gosec.

#### 🛠️ Remediación Arquitectónica Sistémica
Propagar/loguear errores de persistencia y rehabilitar G104 en CI.

#### 🔍 Evidencia de Código
```go
_ = task.logger.WriteEvent(*evt)
```

---

### <a id="f4-05"></a>[F4-05] Fuga de conexión HTTP en runWorker por defer resp.Body.Close() antes de loop infinito

- **Severidad**: **HIGH** 🟠
- **Ubicación en Código**: `cmd/gentle-mesh/main.go:964`

#### ⚠️ Invariante Roto y Causa Raíz
En runWorker, la línea 964 ejecuta defer resp.Body.Close() justo antes del loop infinito for {} de heartbeats, dejando la conexión abierta toda la vida del proceso.

#### 🛠️ Remediación Arquitectónica Sistémica
Cerrar resp.Body.Close() explícitamente antes de entrar al loop for.

#### 🔍 Evidencia de Código
```go
defer resp.Body.Close() // Antes del loop infinito en runWorker
```

---

### <a id="f4-06"></a>[F4-06] Condición de carrera peligrosa en killProcessTree: envía SIGKILL tras 3s a un PID que el SO pudo reciclar

- **Severidad**: **HIGH** 🟠
- **Ubicación en Código**: `pkg/server/runner/gentleshell.go:768-774`

#### ⚠️ Invariante Roto y Causa Raíz
Lanza una goroutine que duerme 3s y envía SIGKILL al PID numérico sin cancelar si cmd.Wait() ya recolectó el proceso.

#### 🛠️ Remediación Arquitectónica Sistémica
Usar cmd.WaitDelay o cancelar la goroutine cuando cmd.Wait() retorne.

#### 🔍 Evidencia de Código
```go
time.Sleep(sandboxSIGKILLGrace)
_ = proc.Signal(syscall.SIGKILL)
```

---

### <a id="f4-07"></a>[F4-07] Handlers RFC-002 usan context.Background(), Server.Shutdown olvida webhookDispatcher.Stop() y time.After en loops

- **Severidad**: **MEDIUM** 🟡
- **Ubicación en Código**: `integration/agent/server.go:533, 956+ | pkg/server/http/server.go:632-643 | pkg/server/runner/simulated.go:197`

#### ⚠️ Invariante Roto y Causa Raíz
Los handlers usan context.Background() ignorando r.Context(), Shutdown() no detiene webhookDispatcher y simulated.go usa time.After en loops.

#### 🛠️ Remediación Arquitectónica Sistémica
Derivar de r.Context(), llamar a webhookDispatcher.Stop() y reutilizar time.NewTimer.

#### 🔍 Evidencia de Código
```go
ctx, cancel := context.WithTimeout(context.Background(), s.config.EvalTimeout)
```

---

## Hoja de Ruta Específica del Frente

- **P0 (Inmediato)**: Eliminar el fallback criptográfico a time.Now().UnixNano() en main.go, append.go y shell.go, forzando un error fatal explícito si crypto/rand.Read falla.
- **P1 (Corto Plazo)**: Corregir los errores de persistencia silenciados con _ = en TaskManager y reactivar la regla G104 del linter en .github/workflows/ci.yml. Eliminar la fuga de sockets en runWorker moviendo resp.Body.Close() dentro del bucle de polling. Reemplazar time.After en bucles por time.NewTimer con stop/drain.
- **P2 (Estructural)**: Extraer el servidor RFC-002 de integration/agent/server.go hacia un paquete legítimo en pkg/server/mesh/ y eliminar la dependencia invertida. Descomponer cmd/gentle-mesh/main.go (2,462 LOC) en subcomandos modulares (cmd/gentle-mesh/worker, server, enroll, etc.). Refactorizar killProcessTree para evitar el envío de señales a PIDs reciclados.

