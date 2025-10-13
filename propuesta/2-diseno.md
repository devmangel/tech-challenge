## **2.1 Arquitectura de Alto Nivel**

La arquitectura propuesta implementa una separación clara de responsabilidades mediante capas especializadas, permitiendo escalamiento independiente y mantenimiento optimizado:

[Architecture Diagram](https://miro.com/app/board/uXjVJ7C2YKA=/?focusWidget=3458764643882840802)

**Capas del Sistema:**

- **API Layer**: Entry points y routing
- **Application Layer**: Lógica de negocio y procesamiento
- **Storage Layer**: Persistencia optimizada por tipo de dato
- **Monitoring Layer**: Observabilidad y eventos

**Flujo de Tráfico Progresivo:**

- **Fase 1**: 90% Legacy Monolith | 10% Upload Service
- **Fase 2**: 50% Legacy Monolith | 50% Upload Service  
- **Fase 3**: 0% Legacy Monolith | 100% Upload Service

## **2.2 Componentes Core**

| **Componente** | **Responsabilidad** | **Tecnología** | **Justificación** |
|----------------|-------------------|----------------|-------------------|
| **API Gateway** | • Rate limiting (100 req/s per projects)<br>• Authentication/Authorization<br>• Request routing y versioning | AWS API Gateway | • Managed service reduce overhead<br>• Auto-scaling nativo<br>|
| **Upload Service** | • Gestión de uploads masivos<br>• Chunking strategy (5-10MB chunks)<br>• Pre-signed URL generation<br>• Deduplication logic | NestJS on ECS Fargate<br>2 vCPU, 4GB RAM<br>Auto-scaling 1-8 tasks | • Framework modular ideal para DDD<br>• TypeScript para type safety<br>• Fargate elimina gestión de infraestructura<br>• Right-sized para 5K uploads/día baseline |
| **Processing Service** | • Validación de archivos<br>• Generación de previews<br>• Extracción de metadata CAD/BIM<br>• Compresión inteligente | Node.js + Sharp/DWG libs<br>Auto-scaling 1-12 tasks | • Procesamiento paralelo eficiente<br>• Librerías especializadas CAD<br>• Scale-to-zero en baja demanda<br>• Optimizado para 5K files/día + peaks |
| **Sync Service** | • Detección de cambios<br>• Delta sync optimization<br>• Conflict resolution<br>• Version control | Event-driven architecture<br>EventBridge + Lambda | • Real-time sync capabilities<br>• Serverless para costo-eficiencia<br>• Event sourcing para audit trail |
| **Legacy Monolith** | • APIs existentes (70%)<br>• Business logic actual<br>• Integración gradual | Express.js on EC2<br> instances | • Sistema actual estable<br>• Migración incremental reduce riesgo<br>• Deprecación progresiva planificada |

## **2.3 Flujo de Datos Principal**

### **Upload Flow**

```
1. Cliente → POST /documents/validate 
   └─ Validación de duplicados por fileHash + projectId
   └─ Response: "DUPLICATE" | "PROCEED" | "CONFLICT"

2. Si PROCEED → POST /batch-upload
   └─ Upload Service calcula chunking strategy
   └─ Crea registro en PostgreSQL (metadata inmutable)  
   └─ Crea registro en DynamoDB (estado chunks mutable)
   └─ Genera pre-signed URLs (15min TTL)
   └─ Retorna: {batchId, presignedUrls[], chunkingStrategy}

3. Cliente → S3 Direct Upload (paralelo por chunk)
   └─ Cliente retry S3 upload: 5 intentos con exponential backoff
   └─ Por cada chunk exitoso: PATCH /uploads/{batchId}/chunks/{chunkId}
   └─ Upload Service valida ETag presence (proof of successful S3 upload)

4. S3 Multipart Complete → EventBridge → Processing Service
   └─ Trigger solo cuando batch 100% completo
   └─ Processing time: <30s (imágenes) | <5min (CAD)
   └─ Update final en PostgreSQL + DynamoDB cleanup

5. Recovery Flow (GET /uploads/{batchId}/recovery)
   └─ Retorna chunks completados vs pendientes
   └─ Regenera pre-signed URLs automáticamente
   └─ Cliente resume desde último chunk exitoso
```
