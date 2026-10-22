# Frente 1: Criptografía, PKI & Gestión de Identidad

## Resumen Ejecutivo del Frente

El subsistema de criptografía y PKI de Gentle-Mesh constituye el núcleo de autenticación y confianza de la malla distribuida. Gobierna la autenticación mTLS entre nodos coordinadores y ejecutores, la emisión y revocación de certificados X.509, la validación de identidades soberanas, la firma digital Ed25519 de contratos (CognitiveTaskEnvelope) y el encadenamiento criptográfico no repudiable de recibos (ReceiptChain) bajo la especificación RFC 8785 (JCS). Se descubrieron vulnerabilidades críticas que comprometen el aislamiento de identidades y permiten la reactivación ilícita de nodos revocados.

### Métricas del Frente
- **Total Hallazgos**: 8
- **Distribución de Severidad**: 2 Críticos (CRITICAL), 2 Altos (HIGH), 3 Medios (MEDIUM), 1 Bajos (LOW)

---

## Invariantes de Arquitectura & Seguridad Rotos

1. **Fail-Closed en Revocación mTLS**: La ausencia de un registro de certificado en la base de datos debe impedir el acceso de manera fail-closed. Tratar registros ausentes como certificados válidos destruye el modelo de revocación.
2. **Inmutabilidad del Historial de Certificados**: La rotación de certificados de un nodo debe conservar el número de serie anterior en una lista inmutable de revocados, jamás sobreescribir la fila primaria destruyendo la traza del serial anterior.
3. **Cadena de Custodia Criptográfica No Repudiable**: La firma del emisor en los recibos debe comprometer tanto la firma del ejecutor como la estampa de tiempo de aceptación; de lo contrario, la firma es repudiable y susceptible a ataques de replay.
4. **Normalización Determinista Canónica**: Toda entidad con identificadores de transporte (como URLs de repositorios) debe normalizarse antes de hashearse y antes de firmarse, eliminando divergencias entre cálculo de SHA-256 y firma Ed25519.

---

## Tabla Resumen de Hallazgos

| ID | Severidad | Título | Archivo / Componente Principal |
|---|---|---|---|
| [F1-01](#f1-01) | **CRITICAL** 🔴 | Rotación de certificados borra seriales históricos y falla abierto (fail-open), rehabilitando certificados revocados | `pkg/server/store/certstore.go:142-167, 273-289` |
| [F1-02](#f1-02) | **CRITICAL** 🔴 | Inscripción CSR (/v1/certs/enroll) permite secuestro de identidad (NodeID takeover) y revive nodos revocados | `pkg/server/http/handlers.go:160-222` |
| [F1-03](#f1-03) | **HIGH** 🟠 | Contrafirma del Emisor limpia ExecutorSignature y EmitterAcceptanceAt, rompiendo la cadena de custodia | `pkg/receipt/chain.go:825-840` |
| [F1-04](#f1-04) | **HIGH** 🟠 | Divergencia de normalización de repositorio entre ComputeEnvelopeHash y SignEnvelope | `pkg/envelope/validate.go:297-316` |
| [F1-05](#f1-05) | **MEDIUM** 🟡 | Contradicción y bloqueo en estados terminales de CognitiveTaskEnvelope | `pkg/envelope/types.go:264-275` |
| [F1-06](#f1-06) | **MEDIUM** 🟡 | Falta restricción UNIQUE sobre receipt_id en el esquema SQLite de ChainStore | `pkg/receipt/chain.go:76-96, 465-475, 540-565` |
| [F1-07](#f1-07) | **MEDIUM** 🟡 | Desincronización de caché de revocación en memoria frente a SQLite | `pkg/server/http/server.go:285-292` |
| [F1-08](#f1-08) | **LOW** 🔵 | Certificados ECDSA P-256 incluyen KeyEncipherment (prohibido por RFC 5480) y CA omite MaxPathLenZero | `pkg/pki/pki.go:108-118, 253-255, 635-637` |

---

## Análisis Detallado por Hallazgo

### <a id="f1-01"></a>[F1-01] Rotación de certificados borra seriales históricos y falla abierto (fail-open), rehabilitando certificados revocados

- **Severidad**: **CRITICAL** 🔴
- **Ubicación en Código**: `pkg/server/store/certstore.go:142-167, 273-289 | pkg/server/http/server.go:265-283`

#### ⚠️ Invariante Roto y Causa Raíz
La tabla node_certs define node_id TEXT PRIMARY KEY. Cuando IssueCert rota o re-emite un certificado para un nodo existente, ejecuta ON CONFLICT(node_id) DO UPDATE SET serial = excluded.serial, revoked = 0, borrando el número de serie anterior de SQLite. Luego, en IsCertRevoked, cuando un certificado viejo se conecta por mTLS y su serial ya no existe en la tabla, if err == sql.ErrNoRows { return false, nil } lo trata como NO revocado.

#### 🛠️ Remediación Arquitectónica Sistémica
Cambiar la clave primaria de node_certs a serial TEXT PRIMARY KEY (o mantener una tabla inmutable revoked_serials). Al emitir un nuevo certificado para un node_id, marcar automáticamente todos los seriales previos de ese nodo como revocados.

#### 🔍 Evidencia de Código
```go
if err == sql.ErrNoRows {
    return false, nil // FAIL-OPEN: serial borrado tras rotación pasa como válido
}
```

---

### <a id="f1-02"></a>[F1-02] Inscripción CSR (/v1/certs/enroll) permite secuestro de identidad (NodeID takeover) y revive nodos revocados

- **Severidad**: **CRITICAL** 🔴
- **Ubicación en Código**: `pkg/server/http/handlers.go:160-222 | pkg/server/store/certstore.go:148-160`

#### ⚠️ Invariante Roto y Causa Raíz
En handleCertsEnroll, el cliente envía cualquier req.NodeID junto a un token de inscripción válido (por ejemplo, un token multi-uso). El handler valida el token y la firma del CSR, pero nunca verifica si req.NodeID ya existe o si fue revocado previamente. Al llamar a IssueCert, el UPSERT pisa el certificado legítimo y resetea revoked = 0, revoked_at = NULL.

#### 🛠️ Remediación Arquitectónica Sistémica
Vincular los tokens de inscripción a un AllowedNodeID opcional y rechazar en handleCertsEnroll cualquier intento de inscribir un node_id que ya posea un certificado activo o esté marcado como revocado, salvo rotación autenticada por administrador.

#### 🔍 Evidencia de Código
```go
ON CONFLICT(node_id) DO UPDATE SET serial = excluded.serial, cert_pem = excluded.cert_pem, revoked = 0, revoked_at = NULL
```

---

### <a id="f1-03"></a>[F1-03] Contrafirma del Emisor limpia ExecutorSignature y EmitterAcceptanceAt, rompiendo la cadena de custodia

- **Severidad**: **HIGH** 🟠
- **Ubicación en Código**: `pkg/receipt/chain.go:825-840 | pkg/receipt/signer.go:57-118`

#### ⚠️ Invariante Roto y Causa Raíz
En ComputeEmitterReceiptHash, antes de serializar con JCS (RFC 8785), el código limpia explícitamente cleared.ExecutorSignature = "" y cleared.EmitterAcceptanceAt = nil. Por lo tanto, la firma Ed25519 de aceptación o disputa del emisor no queda ligada criptográficamente a la firma del ejecutor ni al instante en que se tomó la decisión.

#### 🛠️ Remediación Arquitectónica Sistémica
Mantener r.ExecutorSignature y r.EmitterAcceptanceAt dentro del struct canónico que firma el emisor en ComputeEmitterReceiptHash (solo limpiar EmitterSignature).

#### 🔍 Evidencia de Código
```go
cleared := *r
cleared.ExecutorSignature = ""
cleared.EmitterAcceptanceAt = nil
```

---

### <a id="f1-04"></a>[F1-04] Divergencia de normalización de repositorio entre ComputeEnvelopeHash y SignEnvelope

- **Severidad**: **HIGH** 🟠
- **Ubicación en Código**: `pkg/envelope/validate.go:297-316 | pkg/envelope/signer.go:26-38, 60-75`

#### ⚠️ Invariante Roto y Causa Raíz
ComputeEnvelopeHash recorta sufijos / y .git de env.Territory.Repository sobre una copia temporal antes de calcular el hash SHA-256. En cambio, SignEnvelope y VerifyEnvelopeSignature serializan el struct original sin normalizar. Si un envelope se crea con barra final y un transporte intermedio lo normaliza, el hash coincide pero la verificación de firma Ed25519 falla.

#### 🛠️ Remediación Arquitectónica Sistémica
Normalizar env.Territory.Repository directamente en Validate() o en un helper canónico compartido tanto por ComputeEnvelopeHash como por SignEnvelope/VerifyEnvelopeSignature.

#### 🔍 Evidencia de Código
```go
repo := strings.TrimSuffix(strings.TrimSuffix(env.Territory.Repository, "/"), ".git") // Solo en Hash, ausente en SignEnvelope
```

---

### <a id="f1-05"></a>[F1-05] Contradicción y bloqueo en estados terminales de CognitiveTaskEnvelope

- **Severidad**: **MEDIUM** 🟡
- **Ubicación en Código**: `pkg/envelope/types.go:264-275 | pkg/envelope/contract_state.go:14-38`

#### ⚠️ Invariante Roto y Causa Raíz
1) ContractStatusSettled.IsTerminal() devuelve true en types.go, pero allowedTransitions permite transicionar desde SETTLED hacia ACCEPTED_FINAL y DISPUTED. 2) El estado PROPOSED solo permite ir a ACCEPTED o REJECTED; si el ejecutor muere en el pre-flight handshake, no puede transicionar a EXPIRED ni ABANDONED. 3) REMEDIATING no permite ir a SETTLEMENT_TIMEOUT.

#### 🛠️ Remediación Arquitectónica Sistémica
Hacer que ContractStatusSettled.IsTerminal() devuelva false y agregar las transiciones hacia EXPIRED/ABANDONED desde PROPOSED y hacia SETTLEMENT_TIMEOUT desde REMEDIATING.

#### 🔍 Evidencia de Código
```go
case ContractStatusSettled, ContractStatusAcceptedFinal: return true // Contradice contract_state.go
```

---

### <a id="f1-06"></a>[F1-06] Falta restricción UNIQUE sobre receipt_id en el esquema SQLite de ChainStore

- **Severidad**: **MEDIUM** 🟡
- **Ubicación en Código**: `pkg/receipt/chain.go:76-96, 465-475, 540-565`

#### ⚠️ Invariante Roto y Causa Raíz
La tabla receipts define receipt_id TEXT NOT NULL sin PRIMARY KEY ni UNIQUE. InjectReceipt usa INSERT OR REPLACE, que sin constraint UNIQUE sobre receipt_id actúa como INSERT común, permitiendo duplicar receipt_id y haciendo que GetReceipt/UpdateReceipt afecten filas ambiguas.

#### 🛠️ Remediación Arquitectónica Sistémica
Añadir CREATE UNIQUE INDEX IF NOT EXISTS idx_receipts_receipt_id ON receipts(receipt_id); en InitSchema.

#### 🔍 Evidencia de Código
```go
CREATE TABLE IF NOT EXISTS receipts ( receipt_id TEXT NOT NULL, ... )
```

---

### <a id="f1-07"></a>[F1-07] Desincronización de caché de revocación en memoria frente a SQLite

- **Severidad**: **MEDIUM** 🟡
- **Ubicación en Código**: `pkg/server/http/server.go:285-292 | pkg/server/http/middleware.go:384-402`

#### ⚠️ Invariante Roto y Causa Raíz
Server.IsCertRevoked consulta únicamente s.revocationCache en RAM. Si se revoca un certificado mediante CLI o en otra instancia contra la misma base SQLite, el servidor en ejecución jamás consulta s.certStore y sigue aceptando conexiones mTLS del certificado revocado hasta reiniciarse.

#### 🛠️ Remediación Arquitectónica Sistémica
Consultar s.certStore.IsCertRevoked(ctx, serial) cuando la caché en memoria devuelva false o recargar periódicamente desde SQLite.

#### 🔍 Evidencia de Código
```go
return s.revocationCache.IsRevoked(serial) // Nunca consulta s.certStore si hay miss
```

---

### <a id="f1-08"></a>[F1-08] Certificados ECDSA P-256 incluyen KeyEncipherment (prohibido por RFC 5480) y CA omite MaxPathLenZero

- **Severidad**: **LOW** 🔵
- **Ubicación en Código**: `pkg/pki/pki.go:108-118, 253-255, 635-637`

#### ⚠️ Invariante Roto y Causa Raíz
GenerateNodeCert y SignCSR configuran KeyUsageKeyEncipherment en llaves ECDSA P-256 (RFC 5480 §3 lo prohíbe para ECDSA). Además, GenerateCA define MaxPathLen: 0 sin MaxPathLenZero: true, omitiendo pathLenConstraint en ASN.1.

#### 🛠️ Remediación Arquitectónica Sistémica
Usar únicamente x509.KeyUsageDigitalSignature para certificados de nodo ECDSA y agregar MaxPathLenZero: true en GenerateCA.

#### 🔍 Evidencia de Código
```go
KeyUsage: x509.KeyUsageDigitalSignature | x509.KeyUsageKeyEncipherment
```

---

## Hoja de Ruta Específica del Frente

- **P0 (Inmediato)**: Migrar la PK de node_certs a serial TEXT PRIMARY KEY o crear la tabla revoked_serials. Forzar IsCertRevoked a fallar cerrado ante sql.ErrNoRows. Prohibir en /v1/certs/enroll la re-inscripción de NodeIDs existentes sin token administrativo.
- **P1 (Corto Plazo)**: Preservar ExecutorSignature y EmitterAcceptanceAt en el cálculo del hash JCS para contrafirma del emisor. Compartir el pipeline de normalización de repositorios entre hash y firma Ed25519. Sincronizar la caché de revocación en memoria con SQLite.
- **P2 (Estructural)**: Resolver contradicciones en los estados terminales de CognitiveTaskEnvelope (contract_state.go vs types.go). Añadir constraint UNIQUE sobre receipts(receipt_id). Alinear los flags de certificados ECDSA a RFC 5480 y configurar MaxPathLenZero en CAs.

