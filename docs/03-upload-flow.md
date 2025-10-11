# Upload Flow - Deep Dive

This section provides a detailed walkthrough of the file upload workflow, which is the core solution to BuildPeer's scalability challenge.

---

## Upload Workflow Overview

The proposed architecture decouples the API from file transfer, enabling massive scalability. Here's the high-level flow:

1. **Client initiates upload** → API generates presigned URL(s)
2. **Client uploads directly to S3** (API is bypassed for file transfer)
3. **S3 triggers event** → EventBridge routes to processing queue
4. **Lambda processes file** (thumbnail, scan, metadata extraction)
5. **Status updated in database** → Client notified via WebSocket/polling

**Key insight**: The API never touches the file data, eliminating the bottleneck.

---

## Detailed Sequence Diagram

```
┌──────────┐  ┌─────────┐  ┌──────┐  ┌─────────────┐  ┌────────┐  ┌─────────┐  ┌──────────┐
│  Mobile  │  │   API   │  │  S3  │  │ EventBridge │  │  SQS   │  │ Lambda  │  │    DB    │
│  Client  │  │(NestJS) │  │      │  │             │  │        │  │         │  │(RDS+Redis)│
└────┬─────┘  └────┬────┘  └───┬──┘  └──────┬──────┘  └───┬────┘  └────┬────┘  └─────┬────┘
     │             │            │            │             │            │             │
     │                                                                                │
     │ 1. POST /api/uploads/initiate                                                 │
     │    { projectId, files: [                                                      │
     │      { filename, contentType, size },                                         │
     │      { filename, contentType, size }                                          │
     │    ]}                                                                          │
     ├────────────►│                                                                  │
     │             │                                                                  │
     │             │ 2. Validate:                                                     │
     │             │    - User authenticated (JWT)                                    │
     │             │    - User has access to projectId                                │
     │             │    - Files meet constraints (size, type)                         │
     │             │                                                                  │
     │             │ 3. Create upload records                                         │
     │             │    (status: PENDING)                                             │
     │             ├─────────────────────────────────────────────────────────────────►│
     │             │                                                                  │
     │             │                                                      uploads table│
     │             │                                                      {            │
     │             │                                                        id,        │
     │             │                                                        tenant_id, │
     │             │                                                        project_id,│
     │             │                                                        user_id,   │
     │             │                                                        filename,  │
     │             │                                                        s3_key,    │
     │             │                                                        status: PENDING│
     │             │                                                      }            │
     │             │                                                                  │
     │             │ 4. Generate presigned URL(s)                                     │
     │             │    For each file:                                                │
     │             │    - If size > 5MB: multipart upload                             │
     │             │    - If size ≤ 5MB: single PUT URL                               │
     │             ├────────────►│                                                    │
     │             │              │                                                    │
     │             │◄────────────┤                                                    │
     │             │  Presigned URLs (valid 15 min)                                   │
     │             │                                                                  │
     │             │ 5. Return response                                               │
     │◄────────────┤                                                                  │
     │  Response:  │                                                                  │
     │  {          │                                                                  │
     │    uploads: [                                                                  │
     │      {                                                                         │
     │        uploadId: "uuid",                                                       │
     │        url: "https://s3.amazonaws.com/...",                                    │
     │        method: "PUT",                                                          │
     │        expiresAt: "2024-10-10T14:30:00Z"                                       │
     │      }                                                                         │
     │    ]                                                                           │
     │  }          │                                                                  │
     │             │                                                                  │
     │ 6. Client uploads file(s) directly to S3                                      │
     │    (API is bypassed - no load on API servers)                                 │
     │                                                                                │
     │    PUT https://s3-accelerate.amazonaws.com/bucket/tenant-X/project-Y/file.jpg │
     │    Headers:                                                                    │
     │      Content-Type: image/jpeg                                                  │
     │      Authorization: <presigned signature>                                      │
     ├───────────────────────►│                                                       │
     │                        │                                                       │
     │                        │ 7. S3 validates signature                             │
     │                        │    Accepts upload                                     │
     │                        │                                                       │
     │◄───────────────────────┤                                                       │
     │  200 OK                │                                                       │
     │  ETag: "abc123..."     │                                                       │
     │                        │                                                       │
     │                        │ 8. S3 Event: ObjectCreated                            │
     │                        ├───────────────►│                                      │
     │                        │                │                                      │
     │                        │                │ 9. EventBridge filters & routes      │
     │                        │                │    Rule: s3:ObjectCreated:*          │
     │                        │                │          + bucket = buildpeer-uploads│
     │                        │                ├──────────────►│                      │
     │                        │                │               │                      │
     │                        │                │               │ 10. Message enqueued │
     │                        │                │               │     {                │
     │                        │                │               │       s3Key,         │
     │                        │                │               │       bucket,        │
     │                        │                │               │       size,          │
     │                        │                │               │       timestamp      │
     │                        │                │               │     }                │
     │                        │                │               │                      │
     │                        │                │               │ 11. Lambda polls queue│
     │                        │                │               ├──────────►│          │
     │                        │                │               │           │          │
     │                        │                │               │           │ 12. Process file│
     │                        │                │               │           │          │
     │                        │                │               │           │ a) Download from S3│
     │                        │◄──────────────────────────────────────────┤          │
     │                        │                │               │           │          │
     │                        │                │               │           │ b) Generate thumbnail│
     │                        │                │               │           │    (if image)│
     │                        │                │               │           │          │
     │                        │                │               │           │ c) Virus scan│
     │                        │                │               │           │    (ClamAV)│
     │                        │                │               │           │          │
     │                        │                │               │           │ d) Extract metadata│
     │                        │                │               │           │    (EXIF, file info)│
     │                        │                │               │           │          │
     │                        │                │               │           │ e) Upload processed files│
     │                        │◄──────────────────────────────────────────┤          │
     │                        │  (thumbnails)  │               │           │          │
     │                        │                │               │           │          │
     │                        │                │               │           │ 13. Update DB│
     │                        │                │               │           ├─────────►│
     │                        │                │               │           │          │
     │                        │                │               │           │    UPDATE uploads│
     │                        │                │               │           │    SET status = 'COMPLETED',│
     │                        │                │               │           │        processed_at = NOW(),│
     │                        │                │               │           │        metadata = {...}│
     │                        │                │               │           │    WHERE s3_key = '...'│
     │                        │                │               │           │          │
     │                        │                │               │           │          │
     │                        │                │               │           │ 14. Cache invalidation│
     │                        │                │               │           ├─────────►│
     │                        │                │               │           │    Redis DEL upload:status:{id}│
     │                        │                │               │           │          │
     │                        │                │               │           │          │
     │                        │                │               │ 15. Delete from queue│
     │                        │                │               │◄──────────┤          │
     │                        │                │               │  (success)│          │
     │                        │                │               │           │          │
     │                        │                │               │           │ 16. Publish SNS notification│
     │                        │                │               │           │          │
     │                        │                │               │           │    SNS Topic: upload-completed│
     │                        │                │               │           │    {      │
     │                        │                │               │           │      uploadId,│
     │                        │                │               │           │      tenantId,│
     │                        │                │               │           │      status: 'COMPLETED'│
     │                        │                │               │           │    }      │
     │                        │                │               │           │          │
     │ 17. Client receives notification                                               │
     │     (via WebSocket / polling GET /uploads/{id}/status)                        │
     │◄──────────────────────────────────────────────────────────────────────────────┤
     │  {                                                                             │
     │    uploadId: "uuid",                                                           │
     │    status: "COMPLETED",                                                        │
     │    progress: 100,                                                              │
     │    thumbnailUrl: "https://...",                                                │
     │    processedAt: "2024-10-10T14:32:15Z"                                         │
     │  }                                                                             │
     │                                                                                │
```

---

## Step-by-Step Explanation

### Step 1-2: Client Initiates Upload

**Client request:**
```http
POST /api/uploads/initiate
Content-Type: application/json
Authorization: Bearer <JWT>
X-Tenant-ID: <tenant-uuid>

{
  "projectId": "project-uuid",
  "files": [
    {
      "filename": "plan-arquitectonico.pdf",
      "contentType": "application/pdf",
      "size": 15728640  // 15 MB
    },
    {
      "filename": "foto-obra-1.jpg",
      "contentType": "image/jpeg",
      "size": 3145728  // 3 MB
    }
  ]
}
```

**API validation:**
- JWT authentication valid?
- User belongs to tenant specified in header?
- User has access to projectId?
- File sizes within limits? (max 100 MB per file)
- File types allowed? (whitelist: jpg, png, pdf, dwg, etc.)

### Step 3: Create Upload Records in Database

API creates records in `uploads` table with status `PENDING`:

```sql
INSERT INTO uploads (
  id, tenant_id, project_id, user_id,
  filename, s3_key, content_type, size,
  status, created_at
) VALUES
  ('upload-uuid-1', 'tenant-X', 'project-Y', 'user-Z',
   'plan-arquitectonico.pdf',
   'tenant-X/project-Y/uploads/1728567890-plan-arquitectonico.pdf',
   'application/pdf', 15728640,
   'PENDING', NOW()
  ),
  ('upload-uuid-2', 'tenant-X', 'project-Y', 'user-Z',
   'foto-obra-1.jpg',
   'tenant-X/project-Y/uploads/1728567890-foto-obra-1.jpg',
   'image/jpeg', 3145728,
   'PENDING', NOW()
  );
```

**Why create records before upload?**
- Tracking: We have a record of intent even if upload fails
- Status: Client can poll status immediately
- Security: Presigned URL is only valid for files we expect

### Step 4: Generate Presigned URLs

For **small files (≤5 MB)**: Single PUT URL

```javascript
const url = await s3.getSignedUrl('putObject', {
  Bucket: 'buildpeer-uploads',
  Key: 's3Key',
  ContentType: 'image/jpeg',
  Expires: 900, // 15 minutes
});
```

For **large files (>5 MB)**: Multipart upload URLs

```javascript
// Initiate multipart upload
const multipart = await s3.createMultipartUpload({
  Bucket: 'buildpeer-uploads',
  Key: 's3Key',
  ContentType: 'application/pdf',
});

// Generate URL for each part (5MB chunks)
const partSize = 5 * 1024 * 1024;
const numParts = Math.ceil(fileSize / partSize);
const parts = [];

for (let i = 1; i <= numParts; i++) {
  const partUrl = await s3.getSignedUrl('uploadPart', {
    Bucket: 'buildpeer-uploads',
    Key: s3Key,
    PartNumber: i,
    UploadId: multipart.UploadId,
    Expires: 900,
  });
  parts.push({ partNumber: i, url: partUrl });
}
```

**Why multipart?**
- **Resumable**: If network fails, only re-upload failed parts
- **Parallel**: Client can upload multiple parts simultaneously
- **Required**: Files >5GB must use multipart (AWS limit)

### Step 5: API Response

```json
{
  "uploads": [
    {
      "uploadId": "upload-uuid-1",
      "filename": "plan-arquitectonico.pdf",
      "size": 15728640,
      "method": "MULTIPART",
      "s3UploadId": "multipart-xyz",
      "parts": [
        { "partNumber": 1, "url": "https://s3-accelerate.amazonaws.com/..." },
        { "partNumber": 2, "url": "https://s3-accelerate.amazonaws.com/..." },
        { "partNumber": 3, "url": "https://s3-accelerate.amazonaws.com/..." }
      ],
      "expiresAt": "2024-10-10T14:30:00Z"
    },
    {
      "uploadId": "upload-uuid-2",
      "filename": "foto-obra-1.jpg",
      "size": 3145728,
      "method": "PUT",
      "url": "https://s3-accelerate.amazonaws.com/buildpeer-uploads/...",
      "expiresAt": "2024-10-10T14:30:00Z"
    }
  ]
}
```

**API operation time**: <50ms (lightweight, just URL generation)

### Step 6: Client Uploads to S3

Client uses presigned URLs to upload **directly to S3**:

```javascript
// For single PUT
await fetch(presignedUrl, {
  method: 'PUT',
  body: fileBlob,
  headers: {
    'Content-Type': 'image/jpeg',
  },
});

// For multipart
for (const part of parts) {
  const chunk = fileBlob.slice(
    (part.partNumber - 1) * partSize,
    part.partNumber * partSize
  );

  const response = await fetch(part.url, {
    method: 'PUT',
    body: chunk,
  });

  const etag = response.headers.get('ETag');
  uploadedParts.push({ PartNumber: part.partNumber, ETag: etag });
}

// Complete multipart upload
await fetch(`/api/uploads/${uploadId}/complete`, {
  method: 'POST',
  body: JSON.stringify({ parts: uploadedParts }),
});
```

**Key benefits:**
- ✅ API servers don't handle file data (no bandwidth consumed)
- ✅ S3 Transfer Acceleration routes through nearest CloudFront edge
- ✅ Client can retry individual chunks if network fails
- ✅ Parallel uploads (all files simultaneously)

### Step 7-9: S3 Event Trigger

When upload completes, S3 emits `s3:ObjectCreated:*` event:

```json
{
  "Records": [{
    "eventName": "ObjectCreated:Put",
    "s3": {
      "bucket": { "name": "buildpeer-uploads" },
      "object": {
        "key": "tenant-X/project-Y/uploads/1728567890-foto-obra-1.jpg",
        "size": 3145728,
        "eTag": "abc123..."
      }
    }
  }]
}
```

EventBridge filters and routes to SQS queue:

**EventBridge Rule:**
```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": { "name": ["buildpeer-uploads"] }
  }
}
```

**Why SQS between EventBridge and Lambda?**
- **Buffering**: If Lambda is at concurrency limit, messages wait in queue
- **Retries**: SQS automatically retries failed Lambda invocations (3 times)
- **DLQ**: Failed messages go to Dead Letter Queue for investigation
- **Backpressure**: Prevents overwhelming downstream systems during spikes

### Step 10-12: Lambda Processing

Lambda function is triggered by SQS messages (batch size: 10 messages):

**Image Processor Lambda:**
```javascript
async function processImage(s3Key) {
  // 1. Download from S3
  const image = await s3.getObject({ Bucket, Key: s3Key });

  // 2. Generate thumbnails
  const thumbnail200 = await sharp(image.Body)
    .resize(200, 200)
    .jpeg({ quality: 80 })
    .toBuffer();

  const thumbnail800 = await sharp(image.Body)
    .resize(800, 800)
    .jpeg({ quality: 80 })
    .toBuffer();

  // 3. Upload thumbnails
  await s3.putObject({
    Bucket,
    Key: s3Key.replace('/uploads/', '/thumbnails/200x200/'),
    Body: thumbnail200,
  });

  // 4. Extract EXIF metadata
  const metadata = await sharp(image.Body).metadata();
  const exif = await exifReader(image.Body);

  return {
    width: metadata.width,
    height: metadata.height,
    gps: exif.gps, // GPS coordinates from phone
    timestamp: exif.timestamp,
  };
}
```

**Document Processor Lambda:**
```javascript
async function processDocument(s3Key) {
  // 1. Download PDF
  const pdf = await s3.getObject({ Bucket, Key: s3Key });

  // 2. Generate preview (first page as image)
  const preview = await pdfToImage(pdf.Body, { page: 1 });

  // 3. Upload preview
  await s3.putObject({
    Bucket,
    Key: s3Key.replace('/uploads/', '/previews/'),
    Body: preview,
  });

  // 4. Extract text (optional OCR)
  const text = await extractPdfText(pdf.Body);

  return { previewUrl, pageCount, textContent: text };
}
```

**Security Scanner Lambda:**
```javascript
async function scanFile(s3Key) {
  const file = await s3.getObject({ Bucket, Key: s3Key });

  // ClamAV virus scan
  const scanResult = await clamav.scan(file.Body);

  if (scanResult.isInfected) {
    // Quarantine file
    await s3.putObjectTagging({
      Bucket,
      Key: s3Key,
      Tagging: { TagSet: [{ Key: 'quarantine', Value: 'true' }] },
    });

    throw new Error(`Virus detected: ${scanResult.virus}`);
  }

  return { clean: true };
}
```

**Processing time:**
- Small images (2-3 MB): ~5 seconds
- Large PDFs (15-50 MB): ~15-30 seconds
- Virus scan: ~10-20 seconds

### Step 13-14: Update Database

After processing, Lambda updates upload record:

```sql
UPDATE uploads
SET
  status = 'COMPLETED',
  processed_at = NOW(),
  metadata = '{
    "width": 4032,
    "height": 3024,
    "thumbnailUrl": "https://...",
    "gps": { "lat": 25.6866, "lng": -100.3161 }
  }'::jsonb
WHERE s3_key = 'tenant-X/project-Y/uploads/1728567890-foto-obra-1.jpg';
```

Redis cache invalidated:
```javascript
await redis.del(`upload:status:${uploadId}`);
```

**Why invalidate cache?**
- Client may be polling `GET /uploads/{id}/status` every 5 seconds
- Cache prevents database overload
- After status changes, cache must be cleared to show new status

### Step 15-16: Notification

Lambda publishes to SNS topic:

```javascript
await sns.publish({
  TopicArn: 'arn:aws:sns:us-east-1:123456:upload-completed',
  Message: JSON.stringify({
    uploadId,
    tenantId,
    projectId,
    status: 'COMPLETED',
    thumbnailUrl,
  }),
});
```

SNS fans out to subscribers:
- **WebSocket API**: Pushes real-time update to connected clients
- **Mobile push**: Firebase Cloud Messaging (FCM) notification
- **Email**: Digest email (once per hour, batched)

### Step 17: Client Receives Update

**Option A: WebSocket (real-time)**
```javascript
socket.on('upload:completed', (data) => {
  console.log(`Upload ${data.uploadId} completed!`);
  showThumbnail(data.thumbnailUrl);
});
```

**Option B: Polling (fallback)**
```javascript
// Poll every 5 seconds
const interval = setInterval(async () => {
  const status = await fetch(`/api/uploads/${uploadId}/status`);

  if (status.status === 'COMPLETED') {
    clearInterval(interval);
    showThumbnail(status.thumbnailUrl);
  }
}, 5000);
```

---

## Error Handling

### Upload Failures

**Scenario**: Network failure during upload to S3

**Handling**:
1. Client detects failed chunk (HTTP error or timeout)
2. Client retries failed chunk (up to 5 attempts with exponential backoff)
3. If all retries fail:
   - Update upload record: `status = 'FAILED', error = 'Network timeout'`
   - Client shows error message
   - User can retry upload from app

### Processing Failures

**Scenario**: Lambda function times out or crashes

**Handling**:
1. SQS message not deleted (Lambda didn't acknowledge)
2. SQS waits visibility timeout (5 minutes)
3. SQS re-delivers message (up to 3 retries)
4. If still failing: Move to Dead Letter Queue (DLQ)
5. CloudWatch alarm triggers → Ops team investigates
6. Upload record stays in `UPLOADED` status (not `COMPLETED`)
7. User sees "Processing..." in UI

**Manual recovery**:
- Ops team inspects DLQ message
- Fixes issue (e.g., Lambda memory, corrupt file)
- Reprocesses manually or re-queues message

### Idempotency

**Scenario**: Lambda processes same file twice (SQS retry)

**Protection**:
```javascript
async function processUpload(s3Key) {
  // Check if already processed
  const upload = await db.query(
    'SELECT status FROM uploads WHERE s3_key = $1',
    [s3Key]
  );

  if (upload.status === 'COMPLETED') {
    console.log('Already processed, skipping');
    return; // Idempotent
  }

  // Proceed with processing...
}
```

**Why important?**
- At-least-once delivery (SQS guarantee)
- Prevents duplicate thumbnails, duplicate notifications
- Safe to retry

---

## Performance Characteristics

| Metric | Target | Actual (expected) |
|--------|--------|-------------------|
| **Presigned URL generation** | <50ms | ~20ms (P95) |
| **Upload to S3 (5MB file)** | <30s | ~15s from Mexico (with Transfer Acceleration) |
| **Upload to S3 (50MB file)** | <3min | ~2min (multipart, parallel) |
| **Processing latency** | <30s | ~10s (image), ~20s (PDF) |
| **End-to-end (upload + process)** | <2min | ~1min for typical 5MB file |
| **Upload success rate** | >98% | 99% (with retries) |

---

## Why This Design Works for BuildPeer

1. **Handles spikes**: 2,000 uploads in 1 hour = 1,800 to S3 directly (infinite capacity), Lambda auto-scales
2. **Unreliable networks**: Multipart + retries = resumable uploads from construction sites
3. **Small team**: Managed services (ECS, Lambda, S3) = minimal ops burden
4. **Cost-effective**: Pay only for what you use (Lambda execution time, S3 storage)
5. **Scalable**: Architecture handles 10x growth without code changes

---

**Next section**: Architectural decisions and trade-offs
