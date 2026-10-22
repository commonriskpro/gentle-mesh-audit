# Frente 2: Aislamiento, Sandboxing & Red HTTP

## Resumen Ejecutivo del Frente

El frente de aislamiento abarca la ejecución de procesos dentro del sandbox Bubblewrap (bwrap), la interfaz HTTP de los coordinadores y workers, el despachador de webhooks salientes, la sincronización de federación multi-cluster y el servicio de archivos del espacio de trabajo. Este frente presenta la mayor concentración de vulnerabilidades críticas y altas (5 Críticos y 9 Altos), incluyendo escapes totales de sandbox al host por montajes con permisos de escritura (repoCache y ~/.gentle-shell), ejecución arbitraria de comandos en el host sin aislamiento en el subsistema de Settlement, lectura arbitraria de archivos del host mediante symlinks y Path Traversal, y SSRF hacia redes privadas y metadatos de nube.

### Métricas del Frente
- **Total Hallazgos**: 19
- **Distribución de Severidad**: 5 Críticos (CRITICAL), 9 Altos (HIGH), 5 Medios (MEDIUM)

---

## Invariantes de Arquitectura & Seguridad Rotos

1. **Confinamiento Estricto de Sandbox (Zero Host Escape)**: Ningún directorio del host con hooks ejecutables (como repositorios git bare) ni directorios de configuración de usuario (~/.gentle-shell) debe montarse en modo lectura-escritura (--bind). Todas las invocaciones de git en el host deben anular core.hooksPath.
2. **Validación de Contención Física (Anti-Symlink / Path Traversal)**: Toda ruta de archivo servida o evaluada dentro de un workspace debe validarse tras la resolución de symlinks reales con filepath.EvalSymlinks y comprobar contención absoluta con filepath.Rel.
3. **Aislamiento de Entorno del Host**: Procesos que evalúan condiciones o aserciones de settlement jamás deben heredar os.Environ() del host ni ejecutarse directamente en el shell del demonio.
4. **Filtrado Estricto de Tráfico Saliente (Anti-SSRF)**: Cualquier cliente HTTP que realice peticiones basadas en URLs provistas por usuarios (Webhooks, Federación) debe restringir a nivel de socket (net.Dialer.Control) el acceso a loopback, RFC 1918, link-local y endpoints de metadatos cloud.

---

## Tabla Resumen de Hallazgos

| ID | Severidad | Título | Archivo / Componente Principal |
|---|---|---|---|
| [F2-01](#f2-01) | **CRITICAL** 🔴 | Path Traversal arbitrario y lectura/stat de archivos del host en Evaluator.absPath | `pkg/settlement/evaluate.go:259-264, 125, 143, 207-210` |
| [F2-02](#f2-02) | **CRITICAL** 🔴 | Ejecución de comandos de Settlement en el host sin sandbox y filtración de os.Environ() | `pkg/settlement/evaluate.go:66-72, 266-285` |
| [F2-03](#f2-03) | **CRITICAL** 🔴 | Escape de sandbox y ejecución en el host por montaje Read-Write del bare RepoCache en GentleShellRunner | `pkg/server/runner/gentleshell.go:283-294, 594-620` |
| [F2-04](#f2-04) | **CRITICAL** 🔴 | Server-Side Request Forgery (SSRF) crítico en el despachador de Webhooks | `pkg/server/webhook/dispatcher.go:82-105` |
| [F2-05](#f2-05) | **CRITICAL** 🔴 | Lectura arbitraria de archivos del host vía Symlink Traversal en /v1/workspace/file | `pkg/server/http/handlers.go:683-700, 763-800` |
| [F2-06](#f2-06) | **HIGH** 🟠 | Directorio persistente ~/.gentle-shell del host montado en Read-Write dentro del sandbox | `pkg/server/runner/gentleshell.go:238` |
| [F2-07](#f2-07) | **HIGH** 🟠 | Acoplamiento duro a Linux bwrap sin chequeo preflight ni soporte de ejecución real en macOS | `pkg/server/runner/gentleshell.go:214-239, 322-356` |
| [F2-08](#f2-08) | **HIGH** 🟠 | Timeout de inactividad en PiRunner neutralizado por bloqueo síncrono en scanner.Scan() | `pkg/server/runner/pi.go:183-205` |
| [F2-09](#f2-09) | **HIGH** 🟠 | Tormenta ilimitada de goroutines y OOM remoto por io.ReadAll sin límite en Webhooks | `pkg/server/webhook/dispatcher.go:74-76, 108-111` |
| [F2-10](#f2-10) | **HIGH** 🟠 | SSRF saliente y OOM por JSON sin límite en sincronización de Federación (SyncPeer) | `pkg/server/federation/manager.go:82-120, 203-240` |
| [F2-11](#f2-11) | **HIGH** 🟠 | Falta límite de tamaño (MaxBytesReader) en POST /v1/tasks/{id}/checkpoint | `pkg/server/http/handlers.go:1002-1025` |
| [F2-12](#f2-12) | **HIGH** 🟠 | Bypass de Rate Limiter mediante X-Forwarded-For falsificable y fuga infinita de memoria en mapa de clientes | `pkg/server/http/middleware.go:40-70, 83-93` |
| [F2-13](#f2-13) | **HIGH** 🟠 | Worker daemon carece de HostMiddleware/CORSMiddleware y permite bind externo sin autenticación | `pkg/server/worker/server.go:60-120` |
| [F2-14](#f2-14) | **HIGH** 🟠 | Cola interna del Worker responde HTTP 200 prematuro y rompe el failover multi-nodo en MeshRunner | `pkg/server/worker/server.go:183-255` |
| [F2-15](#f2-15) | **MEDIUM** 🟡 | Tokenizer SplitCommand descarta argumentos vacíos, falla con tabs/newlines y silencia comillas abiertas | `pkg/settlement/evaluate.go:301-331` |
| [F2-16](#f2-16) | **MEDIUM** 🟡 | Fuga de goroutines y conexiones HTTP sin drenar en consultas interactivas de RemoteRunner | `pkg/server/runner/remote.go:205-236` |
| [F2-17](#f2-17) | **MEDIUM** 🟡 | Caché estancada en Federación genera conflictos territoriales fantasma permanentes | `pkg/server/federation/manager.go:312-385` |
| [F2-18](#f2-18) | **MEDIUM** 🟡 | CORS permite cualquier subdominio global *.ts.net de Tailscale | `pkg/server/http/middleware.go:132-140` |
| [F2-19](#f2-19) | **MEDIUM** 🟡 | Pérdida permanente de respuesta en /v1/reply del Worker por borrado prematuro del mapa | `pkg/server/worker/server.go:279-307` |

---

## Análisis Detallado por Hallazgo

### <a id="f2-01"></a>[F2-01] Path Traversal arbitrario y lectura/stat de archivos del host en Evaluator.absPath

- **Severidad**: **CRITICAL** 🔴
- **Ubicación en Código**: `pkg/settlement/evaluate.go:259-264, 125, 143, 207-210`

#### ⚠️ Invariante Roto y Causa Raíz
Evaluator.absPath(p) devuelve p directamente si filepath.IsAbs(p) es true, o hace filepath.Join(e.workingDir, p) sin verificar si ../ escapa del workspace. En evalFileModified, evalFileHashEquals y evalGitCleanWorktree, un contrato puede inspeccionar hashes o existencia de /etc/shadow, ~/.ssh/id_rsa o cualquier directorio del host.

#### 🛠️ Remediación Arquitectónica Sistémica
Rechazar rutas absolutas y validar contención física con filepath.EvalSymlinks y filepath.Rel(e.workingDir, cleanPath), abortando si empieza con ..

#### 🔍 Evidencia de Código
```go
func (e *Evaluator) absPath(p string) string {
    if filepath.IsAbs(p) { return p }
    return filepath.Join(e.workingDir, p)
}
```

---

### <a id="f2-02"></a>[F2-02] Ejecución de comandos de Settlement en el host sin sandbox y filtración de os.Environ()

- **Severidad**: **CRITICAL** 🔴
- **Ubicación en Código**: `pkg/settlement/evaluate.go:66-72, 266-285`

#### ⚠️ Invariante Roto y Causa Raíz
A diferencia de las precondiciones que pasan por ValidatePreconditionCommand, las aserciones de settlement (AssertionCommandExitCode, AssertionCommandOutputContains, AssertionNoRegression) ejecutan comandos directamente en el host vía exec.CommandContext. Además, NewEvaluator inicializa e.env = os.Environ(), exponiendo todos los secretos y credenciales del host.

#### 🛠️ Remediación Arquitectónica Sistémica
Ejecutar las aserciones de comando dentro del sandbox aislado del runner (bwrap/contenedor) y reemplazar os.Environ() por una lista blanca mínima de entorno.

#### 🔍 Evidencia de Código
```go
env: os.Environ(), // Filtra todas las variables y secretos del proceso host
```

---

### <a id="f2-03"></a>[F2-03] Escape de sandbox y ejecución en el host por montaje Read-Write del bare RepoCache en GentleShellRunner

- **Severidad**: **CRITICAL** 🔴
- **Ubicación en Código**: `pkg/server/runner/gentleshell.go:283-294, 594-620`

#### ⚠️ Invariante Roto y Causa Raíz
En buildDefaultSandboxMounts, el repositorio bare cache del host (repoCache) se monta con --bind (lectura y escritura) dentro del sandbox de Bubblewrap. Una tarea dentro del sandbox puede escribir un script en repoCache/hooks/post-checkout. Cuando el demonio del worker ejecuta git -C cachePath worktree remove/prune FUERA del sandbox, git dispara los hooks en el host sin aislamiento.

#### 🛠️ Remediación Arquitectónica Sistémica
Montar repoCache en solo lectura (--ro-bind) y pasar siempre -c core.hooksPath=/dev/null en todas las invocaciones git del host.

#### 🔍 Evidencia de Código
```go
if repoCache != "" {
    out = append(out, "--bind", repoCache, repoCache) // RW permite plantar git hooks
}
```

---

### <a id="f2-04"></a>[F2-04] Server-Side Request Forgery (SSRF) crítico en el despachador de Webhooks

- **Severidad**: **CRITICAL** 🔴
- **Ubicación en Código**: `pkg/server/webhook/dispatcher.go:82-105 | pkg/server/http/handlers.go:896-930`

#### ⚠️ Invariante Roto y Causa Raíz
handleCreateWebhook acepta cualquier req.URL sin validar esquema ni IP de destino. El dispatcher realiza peticiones HTTP POST hacia wh.URL, permitiendo atacar servicios en loopback (127.0.0.1), redes internas VPC (10.0.0.0/8) o metadata cloud (169.254.169.254).

#### 🛠️ Remediación Arquitectónica Sistémica
Validar esquema http/https y configurar un net.Dialer.Control que bloquee conexiones a rangos loopback, RFC 1918, link-local y metadata cloud.

#### 🔍 Evidencia de Código
```go
req, err := http.NewRequest(http.MethodPost, wh.URL, bytes.NewReader(payload))
resp, err := d.client.Do(req)
```

---

### <a id="f2-05"></a>[F2-05] Lectura arbitraria de archivos del host vía Symlink Traversal en /v1/workspace/file

- **Severidad**: **CRITICAL** 🔴
- **Ubicación en Código**: `pkg/server/http/handlers.go:683-700, 763-800`

#### ⚠️ Invariante Roto y Causa Raíz
resolveWorkspacePath limpia la ruta de forma puramente léxica (filepath.Clean y filepath.Rel), pero jamás evalúa enlaces simbólicos con filepath.EvalSymlinks. Si un repositorio o agente crea un symlink dentro del workspace apuntando a /etc/shadow o claves del host, os.ReadFile(fullPath) sigue el symlink y devuelve el archivo por HTTP 200.

#### 🛠️ Remediación Arquitectónica Sistémica
Resolver la ruta física real mediante filepath.EvalSymlinks(fullPath) y verificar con filepath.Rel(s.workspaceRoot, realPath) que resida dentro del workspace.

#### 🔍 Evidencia de Código
```go
fullPath = filepath.Join(s.workspaceRoot, filepath.Clean("/"+subPath))
rel, err := filepath.Rel(s.workspaceRoot, fullPath) // Sin EvalSymlinks
```

---

### <a id="f2-06"></a>[F2-06] Directorio persistente ~/.gentle-shell del host montado en Read-Write dentro del sandbox

- **Severidad**: **HIGH** 🟠
- **Ubicación en Código**: `pkg/server/runner/gentleshell.go:238`

#### ⚠️ Invariante Roto y Causa Raíz
defaultSystemMounts() monta ~/.gentle-shell con --bind (lectura-escritura) en todos los contenedores bwrap, permitiendo contaminar de forma persistente todas las ejecuciones futuras del worker.

#### 🛠️ Remediación Arquitectónica Sistémica
Montar ~/.gentle-shell en modo solo lectura (--ro-bind) y usar un directorio efímero por tarea.

#### 🔍 Evidencia de Código
```go
"--bind", filepath.Join(home, ".gentle-shell"), filepath.Join(home, ".gentle-shell")
```

---

### <a id="f2-07"></a>[F2-07] Acoplamiento duro a Linux bwrap sin chequeo preflight ni soporte de ejecución real en macOS

- **Severidad**: **HIGH** 🟠
- **Ubicación en Código**: `pkg/server/runner/gentleshell.go:214-239, 322-356 | cmd/gentle-mesh/main.go:897`

#### ⚠️ Invariante Roto y Causa Raíz
GentleShellRunner codifica rutas específicas de Linux (/lib64, /proc, --unshare-pid) y no verifica en el arranque si bwrap existe o si el SO es Linux. En macOS falla recién cuando recibe una tarea (GENTLESHELL_START_ERROR).

#### 🛠️ Remediación Arquitectónica Sistémica
Ejecutar validación preflight (runtime.GOOS y exec.LookPath) al inicializar el runner.

#### 🔍 Evidencia de Código
```go
// Falla en ejecución de tarea en vez de validar en el arranque del daemon
```

---

### <a id="f2-08"></a>[F2-08] Timeout de inactividad en PiRunner neutralizado por bloqueo síncrono en scanner.Scan()

- **Severidad**: **HIGH** 🟠
- **Ubicación en Código**: `pkg/server/runner/pi.go:183-205`

#### ⚠️ Invariante Roto y Causa Raíz
En piStream.consume, la verificación time.Since(lastOutput) > inactivityTimeout está dentro de for scanner.Scan(). Si el subproceso se cuelga en silencio, scanner.Scan() bloquea indefinidamente y el timeout jamás se evalúa.

#### 🛠️ Remediación Arquitectónica Sistémica
Bombear líneas desde una goroutine lectora hacia un canal y hacer select contra un time.NewTimer.

#### 🔍 Evidencia de Código
```go
for scanner.Scan() { // Si el hijo se cuelga en silencio, bloquea acá para siempre
```

---

### <a id="f2-09"></a>[F2-09] Tormenta ilimitada de goroutines y OOM remoto por io.ReadAll sin límite en Webhooks

- **Severidad**: **HIGH** 🟠
- **Ubicación en Código**: `pkg/server/webhook/dispatcher.go:74-76, 108-111`

#### ⚠️ Invariante Roto y Causa Raíz
sendToWebhooks lanza go d.sendWebhook sin límite de concurrencia y cuando StatusCode >= 400 ejecuta io.ReadAll(resp.Body) sin io.LimitReader, permitiendo agotar toda la RAM del coordinador.

#### 🛠️ Remediación Arquitectónica Sistémica
Usar un pool acotado de workers y envolver la lectura en io.LimitReader(resp.Body, 4096).

#### 🔍 Evidencia de Código
```go
if resp.StatusCode >= 400 { body, _ := io.ReadAll(resp.Body) }
```

---

### <a id="f2-10"></a>[F2-10] SSRF saliente y OOM por JSON sin límite en sincronización de Federación (SyncPeer)

- **Severidad**: **HIGH** 🟠
- **Ubicación en Código**: `pkg/server/federation/manager.go:82-120, 203-240`

#### ⚠️ Invariante Roto y Causa Raíz
RegisterPeer acepta cualquier Endpoint sin validar IP ni esquema, y SyncPeer decodifica resp.Body con json.NewDecoder sin io.LimitReader.

#### 🛠️ Remediación Arquitectónica Sistémica
Validar URLs e IPs de peers federados y limitar el body con io.LimitReader(resp.Body, 1<<20).

#### 🔍 Evidencia de Código
```go
if err := json.NewDecoder(resp.Body).Decode(&manifest); err != nil
```

---

### <a id="f2-11"></a>[F2-11] Falta límite de tamaño (MaxBytesReader) en POST /v1/tasks/{id}/checkpoint

- **Severidad**: **HIGH** 🟠
- **Ubicación en Código**: `pkg/server/http/handlers.go:1002-1025`

#### ⚠️ Invariante Roto y Causa Raíz
handleTaskCheckpoint decodifica directamente de r.Body sin stdhttp.MaxBytesReader, permitiendo enviar payloads arbitrariamente grandes.

#### 🛠️ Remediación Arquitectónica Sistémica
Envolver r.Body con stdhttp.MaxBytesReader(w, r.Body, 256<<10).

#### 🔍 Evidencia de Código
```go
if err := json.NewDecoder(r.Body).Decode(&req); err != nil
```

---

### <a id="f2-12"></a>[F2-12] Bypass de Rate Limiter mediante X-Forwarded-For falsificable y fuga infinita de memoria en mapa de clientes

- **Severidad**: **HIGH** 🟠
- **Ubicación en Código**: `pkg/server/http/middleware.go:40-70, 83-93`

#### ⚠️ Invariante Roto y Causa Raíz
RateLimitMiddleware confía en X-Forwarded-For sin verificar proxy confiable y nunca expira las entradas de RateLimiter.clients.

#### 🛠️ Remediación Arquitectónica Sistémica
Usar r.RemoteAddr por defecto e implementar expulsión TTL/LRU sobre el mapa de clientes.

#### 🔍 Evidencia de Código
```go
ip := r.Header.Get("X-Forwarded-For")
```

---

### <a id="f2-13"></a>[F2-13] Worker daemon carece de HostMiddleware/CORSMiddleware y permite bind externo sin autenticación

- **Severidad**: **HIGH** 🟠
- **Ubicación en Código**: `pkg/server/worker/server.go:60-120`

#### ⚠️ Invariante Roto y Causa Raíz
Si cfg.BearerTokens está vacío, el worker expone /v1/execute sin autenticación ni validateExposure(), y carece de HostMiddleware/CORSMiddleware.

#### 🛠️ Remediación Arquitectónica Sistémica
Exigir autenticación fuera de loopback y agregar HostMiddleware y CORSMiddleware.

#### 🔍 Evidencia de Código
```go
// pkg/server/worker/server.go omite validateExposure() y HostMiddleware
```

---

### <a id="f2-14"></a>[F2-14] Cola interna del Worker responde HTTP 200 prematuro y rompe el failover multi-nodo en MeshRunner

- **Severidad**: **HIGH** 🟠
- **Ubicación en Código**: `pkg/server/worker/server.go:183-255 | pkg/server/runner/mesh.go:215-245`

#### ⚠️ Invariante Roto y Causa Raíz
Cuando un worker está lleno, encola y envía HTTP 200 OK. Si luego expira QueueTimeout, MeshRunner ve que ya hubo HTTP 200 (!isPreAcceptanceTransportError) y aborta sin hacer failover a otros nodos libres.

#### 🛠️ Remediación Arquitectónica Sistémica
Devolver HTTP 429 Too Many Requests cuando los slots activos estén llenos.

#### 🔍 Evidencia de Código
```go
if !isPreAcceptanceTransportError(runErr) { return runErr, nil }
```

---

### <a id="f2-15"></a>[F2-15] Tokenizer SplitCommand descarta argumentos vacíos, falla con tabs/newlines y silencia comillas abiertas

- **Severidad**: **MEDIUM** 🟡
- **Ubicación en Código**: `pkg/settlement/evaluate.go:301-331`

#### ⚠️ Invariante Roto y Causa Raíz
SplitCommand ignora tabs/newlines, descarta tokens vacíos entre comillas alterando posiciones y no reporta error si una comilla queda abierta.

#### 🛠️ Remediación Arquitectónica Sistémica
Reemplazar por un lexer POSIX determinista que preserve argumentos vacíos entrecomillados y valide comillas.

#### 🔍 Evidencia de Código
```go
if t := strings.TrimSpace(current.String()); t != "" { tokens = append(tokens, t) }
```

---

### <a id="f2-16"></a>[F2-16] Fuga de goroutines y conexiones HTTP sin drenar en consultas interactivas de RemoteRunner

- **Severidad**: **MEDIUM** 🟡
- **Ubicación en Código**: `pkg/server/runner/remote.go:205-236`

#### ⚠️ Invariante Roto y Causa Raíz
Cada EventQuery lanza una goroutine sin límite y cierra resp.Body sin io.Copy(io.Discard, resp.Body), rompiendo keep-alive.

#### 🛠️ Remediación Arquitectónica Sistémica
Acotar queries concurrentes con semáforo y drenar resp.Body antes de cerrar.

#### 🔍 Evidencia de Código
```go
if resp, err := r.client.Do(replyHTTP); err == nil { resp.Body.Close() }
```

---

### <a id="f2-17"></a>[F2-17] Caché estancada en Federación genera conflictos territoriales fantasma permanentes

- **Severidad**: **MEDIUM** 🟡
- **Ubicación en Código**: `pkg/server/federation/manager.go:312-385`

#### ⚠️ Invariante Roto y Causa Raíz
FindConflict evalúa peer.cachedManifest sin verificar si el peer está Offline ni si el manifiesto expiró por TTL.

#### 🛠️ Remediación Arquitectónica Sistémica
Agregar TTL a cachedManifest e ignorar manifiestos de peers marcados como PeerStatusOffline.

#### 🔍 Evidencia de Código
```go
if peer.cachedManifest == nil { continue } // Sin chequeo de TTL ni PeerStatusOffline
```

---

### <a id="f2-18"></a>[F2-18] CORS permite cualquier subdominio global *.ts.net de Tailscale

- **Severidad**: **MEDIUM** 🟡
- **Ubicación en Código**: `pkg/server/http/middleware.go:132-140`

#### ⚠️ Invariante Roto y Causa Raíz
originAllowed acepta cualquier origen terminado en .ts.net, incluyendo tailnets de terceros.

#### 🛠️ Remediación Arquitectónica Sistémica
Restringir al nombre específico de la tailnet del cluster o exigir orígenes explícitos.

#### 🔍 Evidencia de Código
```go
if strings.HasSuffix(hostname, ".ts.net") { return true }
```

---

### <a id="f2-19"></a>[F2-19] Pérdida permanente de respuesta en /v1/reply del Worker por borrado prematuro del mapa

- **Severidad**: **MEDIUM** 🟡
- **Ubicación en Código**: `pkg/server/worker/server.go:279-307`

#### ⚠️ Invariante Roto y Causa Raíz
handleReply ejecuta delete(s.queries, req.QueryID) antes del select que envía al canal. Si cae en default (409), el reintento da 404.

#### 🛠️ Remediación Arquitectónica Sistémica
Borrar req.QueryID únicamente tras confirmar el envío al canal.

#### 🔍 Evidencia de Código
```go
if ok { delete(s.queries, req.QueryID) }
```

---

## Hoja de Ruta Específica del Frente

- **P0 (Inmediato)**: Corregir absPath y resolveWorkspacePath aplicando filepath.EvalSymlinks. Montar repoCache y ~/.gentle-shell como --ro-bind en bwrap y forzar -c core.hooksPath=/dev/null en todas las llamadas de git en el host. Reemplazar os.Environ() por lista blanca limpia en Evaluator.
- **P1 (Corto Plazo)**: Implementar dialer seguro con bloqueo de SSRF en Webhooks y Federación. Envolver todas las lecturas de bodies HTTP entrantes y salientes en io.LimitReader y http.MaxBytesReader. Desacoplar el timeout de inactividad de PiRunner del bloqueo síncrono de scanner.Scan().
- **P2 (Estructural)**: Implementar preflight check de sistema operativo y dependencias (bwrap) al arrancar. Añadir validación de IP real e invalidación de caché TTL en el mapa de RateLimiter. Incorporar HostMiddleware y CORSMiddleware en el servidor worker. Resolver el bypass de failover multi-nodo en MeshRunner devolviendo HTTP 429 ante colas llenas.

