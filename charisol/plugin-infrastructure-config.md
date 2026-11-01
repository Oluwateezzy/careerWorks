# Infrastructure & S3 Configuration — Figma Plugin Migration

> Changes required to AWS S3, CDK, and IAM to support plugin-direct uploads.

---

## 1. S3 Bucket CORS Configuration

The plugin runs inside `https://www.figma.com` and needs to `PUT` objects directly to S3 using presigned URLs. S3 blocks cross-origin `PUT` requests by default.

### Required CORS Rule

Add this to the project data bucket (`charisol-design-system-data` or the value of `AWS_S3_PROJECT_DATA_BUCKET`):

```json
{
  "CORSRules": [
    {
      "ID": "FigmaPluginUpload",
      "AllowedHeaders": [
        "Content-Type",
        "Content-Length",
        "x-amz-content-sha256",
        "x-amz-date",
        "Authorization"
      ],
      "AllowedMethods": ["PUT"],
      "AllowedOrigins": [
        "https://www.figma.com",
        "https://figma.com"
      ],
      "ExposeHeaders": ["ETag"],
      "MaxAgeSeconds": 3600
    }
  ]
}
```

### CDK Implementation

In `cdk/lib/charisol-design-system-stack.ts`, add CORS to the existing bucket:

```typescript
const projectDataBucket = new s3.Bucket(this, 'ProjectDataBucket', {
  // ... existing properties ...
  cors: [
    // ... existing CORS rules ...
    {
      id: 'FigmaPluginUpload',
      allowedHeaders: ['Content-Type', 'Content-Length', 'x-amz-content-sha256', 'x-amz-date', 'Authorization'],
      allowedMethods: [s3.HttpMethods.PUT],
      allowedOrigins: ['https://www.figma.com', 'https://figma.com'],
      exposedHeaders: ['ETag'],
      maxAge: 3600,
    },
  ],
});
```

### Verification

```bash
# Check current CORS
aws s3api get-bucket-cors --bucket charisol-design-system-data

# Test with curl (simulate Figma origin)
curl -v -X OPTIONS \
  -H "Origin: https://www.figma.com" \
  -H "Access-Control-Request-Method: PUT" \
  -H "Access-Control-Request-Headers: Content-Type" \
  https://charisol-design-system-data.s3.amazonaws.com/test
```

Expected response headers:
```
Access-Control-Allow-Origin: https://www.figma.com
Access-Control-Allow-Methods: PUT
Access-Control-Allow-Headers: Content-Type
```

---

## 2. S3 Key Layout for Plugin Uploads

### Upload Staging Area

```
projects/{projectId}/plugin-uploads/{uploadId}/
  ├── manifest.json                    # Inventory manifest
  ├── assets/
  │   ├── {hash1}.png                  # Deduplicated image assets
  │   ├── {hash2}.png
  │   └── ...
  └── renders/
      ├── {nodeId1}@{scale}.png        # Frame screenshots
      └── {nodeId2}@{scale}.png
```

### Finalized Artifact Location (after `complete-upload`)

After validation, the backend copies or references the manifest into the existing artifact structure:

```
projects/{projectId}/prototype/figma/{fileKey}/{inventoryVersion}/
  ├── inventory.json                   # Validated manifest (may be a copy or reference)
  └── metadata.json                    # Artifact metadata

projects/{projectId}/prototype/renders/{inventoryVersion}/
  ├── {nodeId1}@{scale}.png
  └── {nodeId2}@{scale}.png
```

### Design Decision: Copy vs. Reference

**Option A — Copy**: `complete-upload` copies manifest from staging to artifact location.
- Pro: Clean separation; staging can be cleaned up on lifecycle
- Con: Doubles storage temporarily

**Option B — Reference**: `complete-upload` just creates a DynamoDB record pointing to the staging S3 key.
- Pro: No data duplication; faster
- Con: Must not clean up staging until artifact is no longer needed

**Recommendation**: **Option B (Reference)** — record the `s3Key` in the `FigmaInventory` and `PrototypeArtifact` DynamoDB records pointing to the staging location. Add a lifecycle rule that only deletes staging files older than 30 days that have no associated active job.

---

## 3. S3 Lifecycle Rules

### Plugin Upload Cleanup

Add a lifecycle rule to clean up incomplete or abandoned plugin uploads:

```json
{
  "Rules": [
    {
      "ID": "CleanupIncompletePluginUploads",
      "Filter": {
        "Prefix": "projects/*/plugin-uploads/"
      },
      "Status": "Enabled",
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 1
      }
    },
    {
      "ID": "CleanupAbandonedPluginUploads",
      "Filter": {
        "Prefix": "projects/*/plugin-uploads/",
        "Tags": [
          {
            "Key": "upload-status",
            "Value": "pending"
          }
        ]
      },
      "Status": "Enabled",
      "Expiration": {
        "Days": 7
      }
    }
  ]
}
```

### CDK Implementation

```typescript
projectDataBucket.addLifecycleRule({
  id: 'CleanupIncompletePluginUploads',
  prefix: 'projects/',
  abortIncompleteMultipartUploadAfter: Duration.days(1),
});
```

---

## 4. IAM Policy Changes

### New: API Lambda Presign Permission

The API Lambda needs `s3:PutObject` permission on the plugin upload prefix to generate presigned PUT URLs:

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:PutObject",
    "s3:GetObject"
  ],
  "Resource": [
    "arn:aws:s3:::charisol-design-system-data/projects/*/plugin-uploads/*"
  ]
}
```

**Note**: The API Lambda does NOT upload files itself — it only generates presigned URLs. But S3 requires the signer to have `s3:PutObject` permission on the target key.

### Removed: Figma API Network Access (if applicable)

If the worker tasks (Fargate/ECS) had security groups or NAT configurations specifically for calling `api.figma.com`, those can be removed for the prototype workflow. However, keep them if the design-system token sync still uses the Figma REST API.

### CDK Implementation

```typescript
// Add presign permission to API Lambda role
apiLambdaRole.addToPolicy(new iam.PolicyStatement({
  actions: ['s3:PutObject', 's3:GetObject'],
  resources: [
    `${projectDataBucket.bucketArn}/projects/*/plugin-uploads/*`,
  ],
}));
```

---

## 5. Presigned URL Configuration

### Parameters

| Parameter | Value | Rationale |
|:----------|:------|:----------|
| `expiresIn` | 3600 seconds (1 hour) | Generous to allow slow connections; plugin retries with fresh URL on 403 |
| `ContentType` | From request body | Enforce correct content type on upload |
| `x-amz-content-sha256` | `UNSIGNED-PAYLOAD` | Plugin cannot sign the payload in advance |

### Size Limits

| Limit | Value | Rationale |
|:------|:------|:----------|
| Max files per presign request | 500 | Safety cap; typical export is 50–200 files |
| Max total upload size | 100 MB | Covers large files with many images |
| Max single file size | 20 MB | Single images rarely exceed 5 MB at 2x |
| Max manifest.json size | 10 MB | Large inventories (10k nodes) can be 5–8 MB |

### Backend Validation (in `presignPluginUpload`)

```javascript
const MAX_FILES = 500;
const MAX_TOTAL_BYTES = 100 * 1024 * 1024; // 100 MB
const MAX_SINGLE_FILE_BYTES = 20 * 1024 * 1024; // 20 MB
const ALLOWED_CONTENT_TYPES = new Set([
  'application/json',
  'image/png',
  'image/jpeg',
  'image/svg+xml',
]);

function validateUploadRequest(files) {
  if (!Array.isArray(files) || files.length === 0) {
    throw { code: 'INVALID_REQUEST', message: 'files array is required and must not be empty' };
  }
  if (files.length > MAX_FILES) {
    throw { code: 'TOO_MANY_FILES', message: `Maximum ${MAX_FILES} files per upload` };
  }
  
  let totalBytes = 0;
  const hasManifest = files.some(f => f.name === 'manifest.json');
  if (!hasManifest) {
    throw { code: 'MISSING_MANIFEST', message: 'manifest.json must be included in the upload' };
  }
  
  for (const file of files) {
    if (!ALLOWED_CONTENT_TYPES.has(file.contentType)) {
      throw { code: 'INVALID_CONTENT_TYPE', message: `Content type ${file.contentType} not allowed` };
    }
    if (file.sizeBytes > MAX_SINGLE_FILE_BYTES) {
      throw { code: 'FILE_TOO_LARGE', message: `File ${file.name} exceeds ${MAX_SINGLE_FILE_BYTES} bytes` };
    }
    totalBytes += file.sizeBytes;
  }
  
  if (totalBytes > MAX_TOTAL_BYTES) {
    throw { code: 'UPLOAD_TOO_LARGE', message: `Total upload size exceeds ${MAX_TOTAL_BYTES} bytes` };
  }
}
```

---

## 6. CloudWatch Monitoring

### Alarms to Add

| Alarm | Metric | Threshold | Rationale |
|:------|:-------|:----------|:----------|
| PluginUploadErrors | `presign-upload` 5xx count | > 5 per 5 min | Backend failure in presigning |
| PluginValidationErrors | `complete-upload` 422 count | > 10 per 5 min | Schema mismatch (schema drift risk) |
| S3UploadLatency | Custom metric from plugin (optional) | > 30s p99 | Slow uploads may cause presigned URL expiry |
| PluginJobCreationErrors | `complete-upload` 5xx count | > 3 per 5 min | Job creation failure |

### Custom Metrics (Optional — added in backend)

```javascript
// In pluginController.js
const cloudwatch = new CloudWatchClient({});

async function recordPluginMetric(metricName, value, unit = 'Count') {
  await cloudwatch.send(new PutMetricDataCommand({
    Namespace: 'Strata/PluginUploads',
    MetricData: [{
      MetricName: metricName,
      Value: value,
      Unit: unit,
      Timestamp: new Date(),
    }],
  }));
}
```

---

## 7. Rollout Checklist

### Pre-Deploy

- [ ] S3 CORS rule deployed to staging bucket
- [ ] S3 CORS verified with curl OPTIONS test
- [ ] IAM policy for presigned URL generation added
- [ ] S3 lifecycle rules configured
- [ ] `@aws-sdk/s3-request-presigner` added to backend `package.json`
- [ ] CDK stack synthesized and reviewed (`cdk synth`)
- [ ] CloudWatch alarms created in staging

### Post-Deploy Verification

- [ ] Plugin can request presigned URLs from staging backend
- [ ] Plugin can PUT a file to S3 via presigned URL
- [ ] CORS headers are present in S3 PUT response
- [ ] `complete-upload` successfully reads and validates the manifest from S3
- [ ] Job is created with correct state and phase
- [ ] Lifecycle rules are active (check with `aws s3api get-bucket-lifecycle-configuration`)
- [ ] CloudWatch alarms are in OK state
