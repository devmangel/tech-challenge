## **2.4 Manejo de Metadata y Estado**

### **Separación de Responsabilidades entre DBs**

| **Data Type** | **Storage** | **Structure** | **Justificación** |
|---------------|------------|---------------|-------------------|
| **Metadata Inmutable** | Aurora PostgreSQL | Relacional normalizada | Complex queries, relationships, ACID compliance |
| **Estado de Chunks** | DynamoDB | JSON flexible + arrays | High write throughput, schema flexibility, TTL |

### **Estructura de Metadata por Fase**

#### **Metadata Inicial (PostgreSQL)**
```sql
-- Tabla: document_batches
document_batches (
  batch_id UUID PRIMARY KEY,
  project_id UUID NOT NULL,
  user_id UUID NOT NULL,
  file_name VARCHAR(255) NOT NULL,
  file_hash SHA256 NOT NULL, -- Para deduplicación
  file_size BIGINT NOT NULL,
  total_chunks INTEGER NOT NULL,
  chunk_size_mb INTEGER NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  status batch_status_enum DEFAULT 'INITIALIZED',
  UNIQUE(project_id, file_hash) -- Evitar duplicados
);
```

#### **Estado de Progreso (DynamoDB)**
```json
{
  "batchId": "batch-123",
  "projectId": "proj-456",
  "chunksState": {
    "1": {
      "status": "completed",
      "sha256": "7d865e959b2466918c9863afca942d0fb89d7c9ac0c99bafc3749504ded97730",
      "etag": "abc123def456",
      "uploadedAt": "2024-01-15T10:30:00Z",
      "sequenceNumber": 1
    },
    "2": {
      "status": "failed", 
      "failureReason": "sha256_mismatch",
      "retryCount": 2,
      "lastAttemptAt": "2024-01-15T10:35:00Z"
    },
    "3": {
      "status": "pending",
      "presignedUrl": "https://s3.amazonaws.com/...",
      "urlExpiresAt": "2024-01-15T10:45:00Z"
    }
  },
  "summary": {
    "totalChunks": 100,
    "completedChunks": 1,
    "failedChunks": 1,
    "pendingChunks": 98,
    "progressPercentage": 1
  },
  "ttl": 1705312800, // 7 días desde creación
  "lastUpdated": "2024-01-15T10:30:00Z"
}
```

## **2.5 Estrategia de Timeout y Recovery**

### **Timeout Management Híbrido**

| **Componente** | **Responsabilidad** | **Trigger** | **Acción** |
|----------------|-------------------|-------------|-----------|
| **DynamoDB TTL** | Auto-expire chunks | 30 minutos per chunk | Automatic cleanup |
| **Lambda Scheduled** | Detect stale batches | Cada 5 minutos | Mark as "TIMEOUT", notify client |
| **Upload Service** | Batch abandonment | 24 horas sin actividad | Mark as "ABANDONED" |

### **Recovery API Strategy**

#### **Recovery State Response**
```typescript
interface RecoveryState {
  batchId: string;
  canResume: boolean;
  totalChunks: number;
  completedChunks: number[];
  failedChunks: ChunkFailure[];
  pendingChunks: number[];
  newPresignedUrls: PreSignedUrl[];
  expirationTime: Date;
  estimatedTimeRemaining?: string;
}

interface ChunkFailure {
  chunkId: number;
  failureReason: 'timeout' | 'sha256_mismatch' | 's3_error';
  retryCount: number;
  canRetry: boolean;
}
```

### **Idempotencia y Deduplicación**

#### **Estrategia de Fingerprinting**
```yaml
Deduplication Flow:
1. Cliente calcula SHA-256 del archivo completo
2. POST /documents/validate {fileHash, projectId, fileName}
3. Server verifica: SELECT COUNT(*) FROM document_batches WHERE project_id = ? AND file_hash = ?
4. Response cases:
   - DUPLICATE: Archivo idéntico ya existe
   - PROCEED: Archivo único, continuar
   - CONFLICT: Mismo nombre pero diferente hash (versioning)

Chunk Integrity (Trust but Verify):
1. Cliente sube chunk a S3 → obtiene ETag (MD5 del contenido)
2. PATCH /chunks/{id} {etag, status: 'completed'}
3. Server valida: ETag presence = proof of successful S3 upload
4. Para chunks failed: regenerar pre-signed URL y retry
```

### **Fault Tolerance Patterns**

| **Failure Scenario** | **Detection** | **Recovery Action** | **User Experience** |
|---------------------|---------------|-------------------|------------------|
| **Chunk Upload Fail** | S3 HTTP 5xx/timeout | Cliente retry 5x exponential backoff | Transparent retry |
| **SHA-256 Mismatch** | Server validation | Pause batch, request re-upload | Error notification |
| **Network Disconnect** | Client-side timeout | Resume from last successful chunk | Seamless recovery |
| **Batch Abandonment** | 24h no activity | Auto-cleanup + notification | Manual re-initiation |
| **Server Unavailable** | Health check failure | Circuit breaker + fallback | Graceful degradation |
