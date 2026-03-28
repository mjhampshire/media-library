# TWC Media Library - Design Document

**Status:** Prototype
**Last Updated:** March 2025

---

## Overview

A media library for the TWC backoffice that allows retailers to upload and manage media assets (images, PDFs, videos) for use in outreach messages (SMS, email, etc.).

### Key Use Cases

1. **Collection PDFs** - Upload new collection lookbook, auto-generate SMS-optimized (compressed) and email-optimized versions
2. **Waiver Forms** - Upload appointment waivers that can be attached to confirmation emails
3. **Invitations** - Event invitations with multiple size variants
4. **Promotional Images** - Sale banners, new arrival highlights

---

## Data Model

### Media Asset (DynamoDB)

```
Table: TWC_MEDIA_ASSET

Partition Key: tenantId (String)
Sort Key: assetId (String)

GSI1: tenantId-category-index
  - PK: tenantId
  - SK: category#createdAt

GSI2: tenantId-status-index
  - PK: tenantId
  - SK: status#createdAt
```

#### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| **tenantId** | String | Yes | Retailer/tenant ID |
| **assetId** | String | Yes | Unique asset ID (ULID for sortability) |
| **title** | String | Yes | Display name (e.g., "Summer 2025 Collection") |
| **description** | String | No | Optional description |
| **category** | String | Yes | Asset category (see enum below) |
| **tags** | List<String> | No | Searchable tags |
| **originalFileName** | String | Yes | Original uploaded file name |
| **mimeType** | String | Yes | MIME type (image/jpeg, application/pdf, etc.) |
| **fileSize** | Number | Yes | File size in bytes |
| **status** | String | Yes | Asset status (see enum below) |
| **expiresAt** | String (ISO) | No | Optional expiry date |
| **currentVersion** | Number | Yes | Current version number |
| **variants** | Map | Yes | Channel-specific variants (see below) |
| **metadata** | Map | No | Custom metadata key-value pairs |
| **createdAt** | String (ISO) | Yes | Creation timestamp |
| **createdBy** | String | Yes | User ID who created |
| **updatedAt** | String (ISO) | Yes | Last update timestamp |
| **updatedBy** | String | Yes | User ID who last updated |

#### Categories (User-Configurable)

Categories are managed per-tenant. Backoffice users can add/edit/delete categories.

**Default categories** (seeded on tenant creation):

| Value | Description |
|-------|-------------|
| `collection` | Collection lookbooks, PDFs |
| `invitation` | Event invitations |
| `promotion` | Sale banners, promotional images |
| `product` | Product images (if not from catalog) |
| `brand` | Brand assets, logos |
| `other` | Uncategorized |

See **TWC_MEDIA_CATEGORY** table below for storage.

#### Status Enum

| Value | Description |
|-------|-------------|
| `processing` | Upload complete, variants being generated |
| `active` | Ready for use |
| `expired` | Past expiry date (auto-set by TTL job) |
| `archived` | Manually archived |

#### Variants Map Structure

In Phase 1, variants are **manually uploaded** by the user. Each variant is tagged with its intended channel.

```json
{
  "original": {
    "s3Key": "media/tenant123/abc123/v1/original.pdf",
    "fileName": "Summer_2025_Collection.pdf",
    "fileSize": 8500000,
    "mimeType": "application/pdf"
  },
  "sms": {
    "s3Key": "media/tenant123/abc123/v1/sms.jpg",
    "fileName": "Summer_2025_SMS.jpg",
    "fileSize": 850000,
    "mimeType": "image/jpeg"
  },
  "whatsapp": {
    "s3Key": "media/tenant123/abc123/v1/whatsapp.jpg",
    "fileName": "Summer_2025_WhatsApp.jpg",
    "fileSize": 2500000,
    "mimeType": "image/jpeg"
  },
  "email": {
    "s3Key": "media/tenant123/abc123/v1/email.pdf",
    "fileName": "Summer_2025_Email.pdf",
    "fileSize": 2500000,
    "mimeType": "application/pdf"
  }
}
```

#### Variant Types

| Variant | Description | Max Size | Formats | Channel Use |
|---------|-------------|----------|---------|-------------|
| `sms` | Optimized for MMS | 1.2 MB | JPG, PNG, PDF | MMS messages |
| `whatsapp` | WhatsApp Business API | 5 MB | JPG, PNG | WhatsApp messages |
| `email` | Email attachments | 10 MB | PDF, JPG, PNG | Email campaigns |
| `original` | Full quality archive | 10 MB | PDF, PNG, JPG | Backup/source |

**WhatsApp Notes:**
- WhatsApp Business API requires images to be uploaded to Meta's servers
- Media IDs expire after ~24 hours, requiring re-upload for reuse
- Only JPEG and PNG supported (no PDF, GIF, or WebP for images)
- Max 5MB per image file
- See [WhatsApp Media API docs](https://developers.facebook.com/docs/whatsapp/cloud-api/reference/media)

**Phase 2:** Auto-generate variants from original upload using Lambda + Sharp/Ghostscript.

### Media Category (DynamoDB)

User-configurable categories per tenant.

```
Table: TWC_MEDIA_CATEGORY

Partition Key: tenantId (String)
Sort Key: categoryId (String)
```

#### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| **tenantId** | String | Yes | Retailer/tenant ID |
| **categoryId** | String | Yes | URL-safe slug (e.g., "collection", "vip-event") |
| **name** | String | Yes | Display name (e.g., "VIP Event") |
| **description** | String | No | Optional description |
| **sortOrder** | Number | No | For ordering in dropdowns |
| **isDefault** | Boolean | No | System default (can't be deleted) |
| **createdAt** | String | Yes | Creation timestamp |
| **updatedAt** | String | Yes | Last update timestamp |

---

### Media Version (DynamoDB)

For version history - separate table to avoid bloating the main record.

```
Table: TWC_MEDIA_VERSION

Partition Key: tenantId#assetId (String)
Sort Key: version (Number)
```

#### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| **tenantId** | String | Yes | Tenant ID |
| **assetId** | String | Yes | Parent asset ID |
| **version** | Number | Yes | Version number (1, 2, 3...) |
| **variants** | Map | Yes | Variants for this version |
| **changelog** | String | No | What changed in this version |
| **createdAt** | String (ISO) | Yes | Version creation time |
| **createdBy** | String | Yes | User who created this version |

---

## S3 Structure

```
twc-media-{env}/
├── {tenantId}/
│   ├── {assetId}/
│   │   ├── v1/
│   │   │   ├── original.pdf
│   │   │   ├── thumbnail.jpg
│   │   │   ├── sms.jpg
│   │   │   ├── whatsapp.jpg
│   │   │   └── email.jpg
│   │   └── v2/
│   │       ├── original.pdf
│   │       ├── thumbnail.jpg
│   │       ├── sms.jpg
│   │       ├── whatsapp.jpg
│   │       └── email.jpg
│   └── {assetId2}/
│       └── ...
```

### S3 Bucket Policy

- Private bucket (no public access)
- Access via presigned URLs only
- Lifecycle rule: Move to Glacier after 1 year for archived assets
- Versioning enabled at bucket level (backup, not app versioning)

---

## API Design

Base URL: `/api/v1/media`

### 1. List Assets

```http
GET /api/v1/media?category=collection&status=active&limit=20&cursor=xxx
```

**Query Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| category | String | Filter by category |
| status | String | Filter by status (default: active) |
| search | String | Search title, description, tags |
| limit | Number | Page size (default: 20, max: 100) |
| cursor | String | Pagination cursor |

**Response:**
```json
{
  "items": [
    {
      "assetId": "01HQXYZ...",
      "title": "Summer 2025 Collection",
      "category": "collection",
      "thumbnailUrl": "https://presigned-url...",
      "mimeType": "application/pdf",
      "status": "active",
      "expiresAt": "2025-06-01T00:00:00Z",
      "currentVersion": 2,
      "createdAt": "2025-03-01T10:00:00Z",
      "createdBy": "user_123"
    }
  ],
  "nextCursor": "eyJhc3NldElkIjoiMDFIUVhZWiJ9",
  "totalCount": 45
}
```

### 2. Get Asset Details

```http
GET /api/v1/media/{assetId}
```

**Response:**
```json
{
  "assetId": "01HQXYZ...",
  "title": "Summer 2025 Collection",
  "description": "New arrivals for Summer 2025",
  "category": "collection",
  "tags": ["summer", "2025", "lookbook"],
  "originalFileName": "Summer_2025_Lookbook.pdf",
  "mimeType": "application/pdf",
  "fileSize": 2500000,
  "status": "active",
  "expiresAt": "2025-06-01T00:00:00Z",
  "currentVersion": 2,
  "variants": {
    "original": {
      "url": "https://presigned-url...",
      "fileSize": 2500000,
      "mimeType": "application/pdf"
    },
    "thumbnail": {
      "url": "https://presigned-url...",
      "width": 200,
      "height": 200
    },
    "sms": {
      "url": "https://presigned-url...",
      "width": 600,
      "height": 600,
      "fileSize": 50000
    },
    "email": {
      "url": "https://presigned-url...",
      "width": 1200,
      "height": 1200,
      "fileSize": 200000
    }
  },
  "createdAt": "2025-03-01T10:00:00Z",
  "createdBy": "user_123",
  "updatedAt": "2025-03-05T14:30:00Z",
  "updatedBy": "user_456"
}
```

### 3. Get Upload URL

```http
POST /api/v1/media/upload-url
```

**Request:**
```json
{
  "fileName": "Summer_2025_Lookbook.pdf",
  "mimeType": "application/pdf",
  "fileSize": 2500000
}
```

**Response:**
```json
{
  "uploadId": "upload_abc123",
  "uploadUrl": "https://s3-presigned-url...",
  "expiresIn": 3600
}
```

### 4. Create Asset (after upload)

```http
POST /api/v1/media
```

**Request:**
```json
{
  "uploadId": "upload_abc123",
  "title": "Summer 2025 Collection",
  "description": "New arrivals for Summer 2025",
  "category": "collection",
  "tags": ["summer", "2025", "lookbook"],
  "expiresAt": "2025-06-01T00:00:00Z",
  "generateVariants": ["thumbnail", "sms", "email"]
}
```

**Response:**
```json
{
  "assetId": "01HQXYZ...",
  "status": "processing",
  "message": "Asset created. Variants are being generated."
}
```

### 5. Update Asset

```http
PUT /api/v1/media/{assetId}
```

**Request:**
```json
{
  "title": "Summer 2025 Collection - Updated",
  "description": "Updated description",
  "tags": ["summer", "2025", "lookbook", "sale"],
  "expiresAt": "2025-07-01T00:00:00Z",
  "status": "active"
}
```

### 6. Upload New Version

```http
POST /api/v1/media/{assetId}/versions
```

**Request:**
```json
{
  "uploadId": "upload_def456",
  "changelog": "Updated pricing page"
}
```

**Response:**
```json
{
  "assetId": "01HQXYZ...",
  "version": 3,
  "status": "processing"
}
```

### 7. Get Version History

```http
GET /api/v1/media/{assetId}/versions
```

**Response:**
```json
{
  "assetId": "01HQXYZ...",
  "currentVersion": 2,
  "versions": [
    {
      "version": 2,
      "changelog": "Updated cover image",
      "createdAt": "2025-03-05T14:30:00Z",
      "createdBy": "user_456",
      "isCurrent": true
    },
    {
      "version": 1,
      "changelog": null,
      "createdAt": "2025-03-01T10:00:00Z",
      "createdBy": "user_123",
      "isCurrent": false
    }
  ]
}
```

### 8. Delete Asset

```http
DELETE /api/v1/media/{assetId}
```

Soft delete - sets status to `archived`.

### 9. Get Variant URL (for outreach)

Used by the outreach service to get the appropriate variant for a channel.

```http
GET /api/v1/media/{assetId}/url?variant=sms
```

**Response:**
```json
{
  "url": "https://presigned-url...",
  "expiresIn": 3600,
  "mimeType": "image/jpeg",
  "fileSize": 50000
}
```

---

## Category Management APIs

### 10. List Categories

```http
GET /api/v1/media/categories
```

**Response:**
```json
{
  "categories": [
    { "categoryId": "collection", "name": "Collection", "isDefault": true, "sortOrder": 1 },
    { "categoryId": "invitation", "name": "Invitation", "isDefault": true, "sortOrder": 2 },
    { "categoryId": "vip-event", "name": "VIP Event", "isDefault": false, "sortOrder": 10 }
  ]
}
```

### 11. Create Category

```http
POST /api/v1/media/categories
```

**Request:**
```json
{
  "name": "VIP Event",
  "description": "Assets for VIP customer events"
}
```

**Response:**
```json
{
  "categoryId": "vip-event",
  "name": "VIP Event",
  "description": "Assets for VIP customer events"
}
```

### 12. Update Category

```http
PUT /api/v1/media/categories/{categoryId}
```

**Request:**
```json
{
  "name": "VIP Events",
  "description": "Updated description"
}
```

### 13. Delete Category

```http
DELETE /api/v1/media/categories/{categoryId}
```

Returns 400 if category is in use by assets or is a default category.

---

## Supported Media Types

### Images

| Format | MIME Type | Max Size | Notes |
|--------|-----------|----------|-------|
| JPEG | `image/jpeg` | 10 MB | Universal support |
| PNG | `image/png` | 10 MB | Supports transparency |
| GIF | `image/gif` | 5 MB | Animated supported; WhatsApp sends as video |
| WebP | `image/webp` | 10 MB | Convert to JPEG for email compatibility |
| HEIC | `image/heic` | 10 MB | iPhone format; auto-convert to JPEG on upload |

### Documents

| Format | MIME Type | Max Size | Notes |
|--------|-----------|----------|-------|
| PDF | `application/pdf` | 25 MB | Universal support |
| DOCX | `application/vnd.openxmlformats-officedocument.wordprocessingml.document` | 25 MB | Word documents |
| XLSX | `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` | 25 MB | Excel spreadsheets |
| CSV | `text/csv` | 10 MB | Data files |

### Video

| Format | MIME Type | Max Size | Notes |
|--------|-----------|----------|-------|
| MP4 | `video/mp4` | 100 MB | H.264 codec preferred |
| MOV | `video/quicktime` | 100 MB | iPhone format; convert to MP4 |
| WebM | `video/webm` | 100 MB | Web-optimized |

---

### Channel Compatibility Matrix

| Type | SMS/MMS | WhatsApp | Email | Notes |
|------|---------|----------|-------|-------|
| **JPEG** | ✅ ≤1.2MB | ✅ ≤5MB | ✅ ≤10MB | Universal |
| **PNG** | ✅ ≤1.2MB | ✅ ≤5MB | ✅ ≤10MB | Universal |
| **GIF** | ✅ ≤1.2MB | ⚠️ Converts to video | ✅ ≤5MB | WhatsApp sends as MP4 |
| **WebP** | ❌ | ✅ ≤5MB | ⚠️ Limited | Convert to JPEG for SMS/email |
| **HEIC** | ❌ | ❌ | ❌ | Always convert to JPEG |
| **PDF** | ✅ ≤1.2MB | ❌ | ✅ ≤25MB | WhatsApp doesn't support |
| **DOCX** | ❌ | ✅ ≤16MB | ✅ ≤25MB | No MMS support |
| **XLSX** | ❌ | ✅ ≤16MB | ✅ ≤25MB | No MMS support |
| **CSV** | ❌ | ✅ ≤16MB | ✅ ≤10MB | No MMS support |
| **MP4** | ⚠️ ≤600KB | ✅ ≤16MB | ✅ ≤25MB | MMS very limited |
| **MOV** | ❌ | ❌ | ⚠️ | Convert to MP4 |
| **WebM** | ❌ | ❌ | ⚠️ | Convert to MP4 |

**Legend:** ✅ Supported | ⚠️ Limited/Converted | ❌ Not supported

---

### Processing Pipeline

Some formats require server-side processing before storage:

| Input Format | Processing | Output |
|--------------|------------|--------|
| HEIC | Convert to JPEG | JPEG (original preserved) |
| MOV | Transcode to MP4 | MP4 (H.264) |
| WebM | Transcode to MP4 | MP4 (H.264) |
| GIF (for WhatsApp) | Convert to MP4 | MP4 loop |
| Large images | Generate thumbnail | 200x200 JPEG |
| Video | Generate thumbnail | First frame JPEG |
| PDF | Generate thumbnail | First page as JPEG |

**Implementation:** Lambda functions triggered on S3 upload:
- **Images:** Sharp (Node.js) or Pillow (Python)
- **Video:** FFmpeg via Lambda layer or ECS task
- **PDF thumbnails:** pdf-lib or Ghostscript

---

### Storage Considerations

| Type | Lifecycle | Rationale |
|------|-----------|-----------|
| Images | Standard | Frequently accessed |
| Documents | Standard | Frequently accessed |
| Video (original) | Intelligent-Tiering | Large files, variable access |
| Video (transcoded) | Standard | Delivery copies |
| Thumbnails | Standard | Always needed for UI |
| Archived assets | Glacier | Cost optimization |

---

## Media Processing Architecture

### Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              UPLOAD FLOW                                     │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌──────────┐      ┌──────────┐      ┌──────────────────┐
  │  Client  │─────▶│  API     │─────▶│  S3 (uploads/)   │
  │  Upload  │      │  Server  │      │  Presigned URL   │
  └──────────┘      └──────────┘      └────────┬─────────┘
                                               │
                                               │ S3 Event
                                               ▼
                    ┌──────────────────────────────────────────────────────┐
                    │                   EventBridge                         │
                    │              (routes by file type)                    │
                    └───────┬──────────────┬──────────────┬────────────────┘
                            │              │              │
              ┌─────────────▼──┐    ┌──────▼──────┐    ┌──▼───────────────┐
              │  Image Lambda  │    │  Doc Lambda │    │  Video Queue     │
              │  (Sharp)       │    │  (pdf-lib)  │    │  (SQS)           │
              └───────┬────────┘    └──────┬──────┘    └────────┬─────────┘
                      │                    │                    │
                      │                    │                    ▼
                      │                    │           ┌────────────────────┐
                      │                    │           │  ECS Fargate Task  │
                      │                    │           │  (FFmpeg)          │
                      │                    │           └─────────┬──────────┘
                      │                    │                     │
                      ▼                    ▼                     ▼
              ┌────────────────────────────────────────────────────────────┐
              │                    S3 (processed/)                          │
              │         Thumbnails, Converted Files, Variants              │
              └──────────────────────────┬─────────────────────────────────┘
                                         │
                                         ▼
                              ┌─────────────────────┐
                              │  DynamoDB Update    │
                              │  status: 'active'   │
                              └─────────────────────┘
```

---

### S3 Bucket Structure

```
twc-media-{env}/
├── uploads/                          # Raw uploads (temporary)
│   └── {uploadId}/
│       └── original.{ext}
│
├── {tenantId}/                       # Processed assets
│   └── assets/
│       └── {assetId}/
│           └── v{version}/
│               ├── original.{ext}    # Original (possibly converted)
│               ├── thumbnail.jpg     # 200x200 preview
│               ├── sms.jpg           # Channel variants
│               ├── whatsapp.jpg
│               ├── email.{ext}
│               └── converted.mp4     # If video was transcoded
│
└── customers/                        # Customer-specific media
    └── {customerId}/
        └── ...
```

---

### Lambda Functions

#### 1. Media Router (Entry Point)

Triggered by S3 `ObjectCreated` events on `uploads/` prefix.

```
Function: twc-media-router
Runtime: Node.js 20.x
Memory: 256 MB
Timeout: 30 seconds
Trigger: S3 ObjectCreated (uploads/*)
```

**Logic:**
```javascript
exports.handler = async (event) => {
  const { bucket, key } = parseS3Event(event);
  const mimeType = await detectMimeType(bucket, key);
  const uploadId = extractUploadId(key);

  // Route to appropriate processor
  if (isImage(mimeType)) {
    await invokeLambda('twc-media-image-processor', { bucket, key, uploadId });
  } else if (isDocument(mimeType)) {
    await invokeLambda('twc-media-doc-processor', { bucket, key, uploadId });
  } else if (isVideo(mimeType)) {
    await sendToSQS('twc-media-video-queue', { bucket, key, uploadId });
  } else {
    await handleUnsupportedType(uploadId, mimeType);
  }
};
```

---

#### 2. Image Processor

Handles JPEG, PNG, GIF, WebP, HEIC.

```
Function: twc-media-image-processor
Runtime: Node.js 20.x
Memory: 1024 MB (Sharp needs memory for large images)
Timeout: 60 seconds
Layers: Sharp layer
```

**Processing Steps:**

| Input | Output | Processing |
|-------|--------|------------|
| HEIC | JPEG | Convert using Sharp |
| Any image | thumbnail.jpg | Resize to 200x200, cover crop |
| Large image | sms.jpg | Resize to max 600px, compress to <1MB |
| Large image | whatsapp.jpg | Resize to max 1200px, compress to <5MB |
| Large image | email.jpg | Resize to max 1600px, quality 85% |
| Original | original.{ext} | Copy (or converted JPEG for HEIC) |

**Code:**
```javascript
const sharp = require('sharp');

async function processImage(bucket, key, uploadId, assetId, tenantId) {
  const input = await s3.getObject({ Bucket: bucket, Key: key });
  let image = sharp(input.Body);
  const metadata = await image.metadata();

  const outputs = [];

  // Convert HEIC to JPEG
  if (metadata.format === 'heif') {
    image = image.jpeg({ quality: 90 });
  }

  // Generate thumbnail
  outputs.push({
    key: `${tenantId}/assets/${assetId}/v1/thumbnail.jpg`,
    buffer: await image.clone()
      .resize(200, 200, { fit: 'cover' })
      .jpeg({ quality: 80 })
      .toBuffer()
  });

  // Generate SMS variant (max 1.2MB, max 600px)
  outputs.push({
    key: `${tenantId}/assets/${assetId}/v1/sms.jpg`,
    buffer: await image.clone()
      .resize(600, 600, { fit: 'inside', withoutEnlargement: true })
      .jpeg({ quality: 70 })
      .toBuffer()
  });

  // Generate WhatsApp variant (max 5MB, max 1200px)
  outputs.push({
    key: `${tenantId}/assets/${assetId}/v1/whatsapp.jpg`,
    buffer: await image.clone()
      .resize(1200, 1200, { fit: 'inside', withoutEnlargement: true })
      .jpeg({ quality: 80 })
      .toBuffer()
  });

  // Generate email variant (max 10MB, max 1600px)
  outputs.push({
    key: `${tenantId}/assets/${assetId}/v1/email.jpg`,
    buffer: await image.clone()
      .resize(1600, 1600, { fit: 'inside', withoutEnlargement: true })
      .jpeg({ quality: 85 })
      .toBuffer()
  });

  // Save original (converted if HEIC)
  outputs.push({
    key: `${tenantId}/assets/${assetId}/v1/original.${metadata.format === 'heif' ? 'jpg' : metadata.format}`,
    buffer: await image.toBuffer()
  });

  // Upload all outputs to S3
  await Promise.all(outputs.map(o =>
    s3.putObject({ Bucket: bucket, Key: o.key, Body: o.buffer })
  ));

  // Update DynamoDB
  await updateAssetStatus(assetId, 'active', outputs);

  // Cleanup upload
  await s3.deleteObject({ Bucket: bucket, Key: key });
}
```

---

#### 3. Document Processor

Handles PDF, DOCX, XLSX, CSV.

```
Function: twc-media-doc-processor
Runtime: Node.js 20.x
Memory: 512 MB
Timeout: 60 seconds
Layers: pdf-lib, LibreOffice (for DOCX/XLSX thumbnails)
```

**Processing Steps:**

| Input | Output | Processing |
|-------|--------|------------|
| PDF | thumbnail.jpg | Render first page using pdf-lib + canvas |
| DOCX/XLSX | thumbnail.jpg | Convert to PDF first, then render |
| Any doc | original.{ext} | Copy unchanged |

**Note:** DOCX/XLSX thumbnail generation is complex. Options:
1. **LibreOffice Lambda Layer** - Heavy (~300MB) but accurate
2. **External service** - CloudConvert, Zamzar API
3. **Generic icon** - Show file type icon instead of preview

**Recommended:** Use generic icons for Phase 1, add LibreOffice in Phase 2.

```javascript
async function processDocument(bucket, key, uploadId, assetId, tenantId) {
  const ext = path.extname(key).toLowerCase();
  const outputs = [];

  if (ext === '.pdf') {
    // Generate PDF thumbnail
    const pdfBuffer = await s3.getObject({ Bucket: bucket, Key: key });
    const thumbnail = await generatePdfThumbnail(pdfBuffer.Body);
    outputs.push({
      key: `${tenantId}/assets/${assetId}/v1/thumbnail.jpg`,
      buffer: thumbnail
    });
  } else {
    // DOCX, XLSX, CSV - use generic icon for now
    outputs.push({
      key: `${tenantId}/assets/${assetId}/v1/thumbnail.jpg`,
      buffer: await getGenericIcon(ext)
    });
  }

  // Copy original
  await s3.copyObject({
    CopySource: `${bucket}/${key}`,
    Bucket: bucket,
    Key: `${tenantId}/assets/${assetId}/v1/original${ext}`
  });

  await updateAssetStatus(assetId, 'active', outputs);
  await s3.deleteObject({ Bucket: bucket, Key: key });
}
```

---

#### 4. Video Processor (ECS Fargate)

Videos are too large and slow for Lambda. Use ECS Fargate with FFmpeg.

```
Queue: twc-media-video-queue (SQS)
Task: twc-media-video-processor
CPU: 1 vCPU
Memory: 2 GB
Container: Custom with FFmpeg
Timeout: 30 minutes
```

**Processing Steps:**

| Input | Output | Processing |
|-------|--------|------------|
| MOV | MP4 | Transcode H.264, AAC audio |
| WebM | MP4 | Transcode H.264, AAC audio |
| MP4 | MP4 | Copy or re-encode if needed |
| Any video | thumbnail.jpg | Extract frame at 1 second |
| Any video | whatsapp.mp4 | Compress to <16MB |
| Any video | email.mp4 | Compress to <25MB |

**ECS Task Definition:**
```json
{
  "family": "twc-media-video-processor",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "containerDefinitions": [{
    "name": "ffmpeg",
    "image": "your-ecr/twc-media-ffmpeg:latest",
    "environment": [
      { "name": "S3_BUCKET", "value": "twc-media-prod" }
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/twc-media-video",
        "awslogs-region": "ap-southeast-2",
        "awslogs-stream-prefix": "video"
      }
    }
  }]
}
```

**FFmpeg Commands:**

```bash
# Generate thumbnail (first frame at 1 second)
ffmpeg -i input.mov -ss 00:00:01 -vframes 1 -vf "scale=200:200:force_original_aspect_ratio=decrease,pad=200:200:(ow-iw)/2:(oh-ih)/2" thumbnail.jpg

# Convert MOV/WebM to MP4
ffmpeg -i input.mov -c:v libx264 -preset medium -crf 23 -c:a aac -b:a 128k -movflags +faststart output.mp4

# Compress for WhatsApp (<16MB)
ffmpeg -i input.mp4 -c:v libx264 -preset slow -crf 28 -maxrate 1M -bufsize 2M -c:a aac -b:a 96k -fs 15M whatsapp.mp4

# Compress for SMS (<600KB) - very aggressive
ffmpeg -i input.mp4 -c:v libx264 -preset slow -crf 35 -maxrate 200k -bufsize 400k -vf "scale=480:-2" -c:a aac -b:a 48k -fs 550K sms.mp4
```

---

### Processing Status Flow

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  uploading  │────▶│  processing │────▶│   active    │     │   failed    │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                           │                                       ▲
                           └───────────────────────────────────────┘
                                      (on error)
```

**DynamoDB Status Updates:**

| Status | Set By | Description |
|--------|--------|-------------|
| `uploading` | API | Presigned URL generated, waiting for upload |
| `processing` | Router Lambda | File received, processing started |
| `active` | Processor | All variants generated, ready for use |
| `failed` | Processor | Processing failed (with error message) |

---

### Error Handling

#### Retry Strategy

| Component | Retries | Backoff |
|-----------|---------|---------|
| Image Lambda | 2 | Exponential |
| Doc Lambda | 2 | Exponential |
| Video SQS | 3 | 30s, 60s, 120s |
| ECS Task | 2 | - |

#### Dead Letter Queue

Failed processing jobs go to `twc-media-dlq` for manual review.

```json
{
  "uploadId": "upload_abc123",
  "assetId": "01HQXYZ...",
  "tenantId": "retailer-xyz",
  "error": "FFmpeg exit code 1: Invalid data found when processing input",
  "attempts": 3,
  "lastAttempt": "2025-03-20T10:30:00Z"
}
```

#### Cleanup

- Failed uploads: Delete from `uploads/` after 24 hours (S3 lifecycle)
- Orphaned processing: Lambda checks for stale `processing` status hourly

---

### Infrastructure (Terraform)

```hcl
# S3 Bucket
resource "aws_s3_bucket" "media" {
  bucket = "twc-media-${var.environment}"
}

resource "aws_s3_bucket_notification" "media_upload" {
  bucket = aws_s3_bucket.media.id

  lambda_function {
    lambda_function_arn = aws_lambda_function.media_router.arn
    events              = ["s3:ObjectCreated:*"]
    filter_prefix       = "uploads/"
  }
}

# Image Processor Lambda
resource "aws_lambda_function" "image_processor" {
  function_name = "twc-media-image-processor"
  runtime       = "nodejs20.x"
  handler       = "index.handler"
  memory_size   = 1024
  timeout       = 60

  layers = [
    "arn:aws:lambda:ap-southeast-2:xxx:layer:sharp:1"
  ]

  environment {
    variables = {
      S3_BUCKET     = aws_s3_bucket.media.id
      DYNAMODB_TABLE = aws_dynamodb_table.media_asset.name
    }
  }
}

# Video Processing Queue
resource "aws_sqs_queue" "video_queue" {
  name                       = "twc-media-video-queue"
  visibility_timeout_seconds = 1800  # 30 minutes
  message_retention_seconds  = 86400 # 24 hours

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.dlq.arn
    maxReceiveCount     = 3
  })
}

# ECS Cluster for Video Processing
resource "aws_ecs_cluster" "media" {
  name = "twc-media-processing"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}
```

---

### Cost Estimates

| Component | Usage | Monthly Cost (est.) |
|-----------|-------|---------------------|
| Lambda (Image) | 10,000 invocations × 5s × 1GB | ~$1 |
| Lambda (Doc) | 5,000 invocations × 3s × 512MB | ~$0.50 |
| ECS Fargate (Video) | 500 tasks × 5min × 1vCPU | ~$5 |
| S3 Storage | 100 GB | ~$2.50 |
| S3 Requests | 100,000 | ~$0.50 |
| SQS | 10,000 messages | ~$0.01 |
| **Total** | | **~$10/month** |

*Scales linearly with usage. Video processing is the largest cost driver.*

---

### Monitoring & Alerting

#### CloudWatch Dashboards

**Dashboard: TWC Media Processing**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  TWC Media Processing Dashboard                           Last 24 hours ▼  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────┐  ┌─────────────────────────┐                  │
│  │  Uploads (24h)          │  │  Processing Status      │                  │
│  │  ████████████ 1,247     │  │  ✅ Success: 1,189 (95%)│                  │
│  │                         │  │  ⏳ Processing: 23      │                  │
│  │  Images: 892            │  │  ❌ Failed: 35 (3%)     │                  │
│  │  Docs: 298              │  │                         │                  │
│  │  Video: 57              │  │                         │                  │
│  └─────────────────────────┘  └─────────────────────────┘                  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Processing Time (p95)                                               │   │
│  │   8s ┤                                                               │   │
│  │   6s ┤     ╭───╮                                                     │   │
│  │   4s ┤ ╭───╯   ╰───╮     ╭───╮                                       │   │
│  │   2s ┤─╯           ╰─────╯   ╰───────────────────────────            │   │
│  │   0s ┼─────────────────────────────────────────────────────────      │   │
│  │      00:00    06:00    12:00    18:00    00:00                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌──────────────────────────────┐  ┌──────────────────────────────┐        │
│  │  Lambda Errors (24h)         │  │  SQS Queue Depth             │        │
│  │  Image Processor: 2          │  │  Video Queue: 3 messages     │        │
│  │  Doc Processor: 0            │  │  DLQ: 5 messages ⚠️           │        │
│  │  Router: 1                   │  │                              │        │
│  └──────────────────────────────┘  └──────────────────────────────┘        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Metrics

| Metric | Namespace | Dimensions | Description |
|--------|-----------|------------|-------------|
| `UploadsTotal` | TWC/Media | TenantId, MediaType | Total uploads |
| `ProcessingDuration` | TWC/Media | Processor, MediaType | Processing time (ms) |
| `ProcessingSuccess` | TWC/Media | Processor | Successful processing |
| `ProcessingFailed` | TWC/Media | Processor, ErrorType | Failed processing |
| `QueueDepth` | TWC/Media | QueueName | Messages waiting |
| `StorageUsed` | TWC/Media | TenantId | S3 storage bytes |

**Custom Metrics (Lambda):**

```javascript
const { MetricUnits, Metrics } = require('@aws-lambda-powertools/metrics');

const metrics = new Metrics({ namespace: 'TWC/Media', serviceName: 'image-processor' });

async function processImage(event) {
  const startTime = Date.now();

  try {
    // ... processing logic ...

    metrics.addMetric('ProcessingSuccess', MetricUnits.Count, 1);
    metrics.addMetric('ProcessingDuration', MetricUnits.Milliseconds, Date.now() - startTime);
    metrics.addDimension('MediaType', 'image');
    metrics.addDimension('TenantId', tenantId);

  } catch (error) {
    metrics.addMetric('ProcessingFailed', MetricUnits.Count, 1);
    metrics.addDimension('ErrorType', error.name);
    throw error;
  } finally {
    metrics.publishStoredMetrics();
  }
}
```

#### Alarms

| Alarm | Condition | Action |
|-------|-----------|--------|
| **HighErrorRate** | ProcessingFailed / Total > 5% over 15min | PagerDuty + Slack |
| **ProcessingBacklog** | QueueDepth > 100 for 10min | Slack |
| **DLQNotEmpty** | DLQ messages > 0 for 5min | Slack |
| **LambdaThrottled** | Throttles > 0 | Slack |
| **HighProcessingTime** | p95 duration > 30s for 15min | Slack |
| **ECSTaskFailed** | Task stopped with error | PagerDuty |
| **StorageQuotaWarning** | TenantStorage > 80% of quota | Email tenant |

**Terraform for Alarms:**

```hcl
resource "aws_cloudwatch_metric_alarm" "high_error_rate" {
  alarm_name          = "twc-media-high-error-rate"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  threshold           = 5
  alarm_description   = "Media processing error rate > 5%"

  metric_query {
    id          = "error_rate"
    expression  = "(failed / total) * 100"
    label       = "Error Rate"
    return_data = true
  }

  metric_query {
    id = "failed"
    metric {
      metric_name = "ProcessingFailed"
      namespace   = "TWC/Media"
      period      = 300
      stat        = "Sum"
    }
  }

  metric_query {
    id = "total"
    metric {
      metric_name = "UploadsTotal"
      namespace   = "TWC/Media"
      period      = 300
      stat        = "Sum"
    }
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
  ok_actions    = [aws_sns_topic.alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "dlq_not_empty" {
  alarm_name          = "twc-media-dlq-not-empty"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "ApproximateNumberOfMessagesVisible"
  namespace           = "AWS/SQS"
  period              = 300
  statistic           = "Average"
  threshold           = 0
  alarm_description   = "Dead letter queue has messages"

  dimensions = {
    QueueName = aws_sqs_queue.dlq.name
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

#### Logging

**Structured Logging (Lambda Powertools):**

```javascript
const { Logger } = require('@aws-lambda-powertools/logger');

const logger = new Logger({
  serviceName: 'image-processor',
  logLevel: 'INFO'
});

async function processImage(event) {
  const { uploadId, assetId, tenantId } = event;

  logger.appendKeys({ uploadId, assetId, tenantId });

  logger.info('Processing started', {
    mimeType: event.mimeType,
    fileSize: event.fileSize
  });

  try {
    // ... processing ...
    logger.info('Processing completed', {
      variantsGenerated: ['thumbnail', 'sms', 'email'],
      durationMs: Date.now() - startTime
    });
  } catch (error) {
    logger.error('Processing failed', {
      error: error.message,
      stack: error.stack
    });
    throw error;
  }
}
```

**Log Insights Queries:**

```sql
-- Failed uploads by tenant (last 24h)
fields @timestamp, tenantId, assetId, @message
| filter @message like /Processing failed/
| stats count() as failures by tenantId
| sort failures desc
| limit 20

-- Slow processing (>10s)
fields @timestamp, tenantId, assetId, durationMs
| filter durationMs > 10000
| sort durationMs desc
| limit 50

-- Processing volume by type
fields @timestamp, mimeType
| filter @message like /Processing started/
| stats count() as uploads by mimeType
| sort uploads desc
```

#### Alerting Channels

| Severity | Channel | Response Time |
|----------|---------|---------------|
| P1 - Critical | PagerDuty | 15 min |
| P2 - High | Slack #alerts + PagerDuty | 1 hour |
| P3 - Medium | Slack #alerts | 4 hours |
| P4 - Low | Email | Next business day |

**SNS Topic Subscriptions:**

```hcl
resource "aws_sns_topic" "alerts" {
  name = "twc-media-alerts"
}

resource "aws_sns_topic_subscription" "pagerduty" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "https"
  endpoint  = "https://events.pagerduty.com/integration/xxx/enqueue"
}

resource "aws_sns_topic_subscription" "slack" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.slack_notifier.arn
}
```

---

### Terraform Modules

Complete Terraform configuration for the media processing infrastructure.

#### Directory Structure

```
terraform/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── prod/
├── modules/
│   ├── media-storage/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── media-processing/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── iam.tf
│   ├── media-database/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── media-monitoring/
│       ├── main.tf
│       ├── variables.tf
│       ├── dashboards.tf
│       └── alarms.tf
└── shared/
    └── ecr/
        └── main.tf
```

---

#### Module: media-storage

**modules/media-storage/main.tf**

```hcl
# S3 Bucket for Media Assets
resource "aws_s3_bucket" "media" {
  bucket = "twc-media-${var.environment}"

  tags = {
    Name        = "TWC Media Storage"
    Environment = var.environment
    Project     = "twc-media"
  }
}

resource "aws_s3_bucket_versioning" "media" {
  bucket = aws_s3_bucket.media.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "media" {
  bucket = aws_s3_bucket.media.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = var.kms_key_arn
    }
  }
}

resource "aws_s3_bucket_public_access_block" "media" {
  bucket = aws_s3_bucket.media.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_lifecycle_configuration" "media" {
  bucket = aws_s3_bucket.media.id

  # Clean up failed uploads after 24 hours
  rule {
    id     = "cleanup-uploads"
    status = "Enabled"

    filter {
      prefix = "uploads/"
    }

    expiration {
      days = 1
    }
  }

  # Move archived assets to Glacier after 90 days
  rule {
    id     = "archive-old-assets"
    status = "Enabled"

    filter {
      and {
        prefix = ""
        tags = {
          status = "archived"
        }
      }
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }
  }

  # Move video originals to Intelligent-Tiering
  rule {
    id     = "video-tiering"
    status = "Enabled"

    filter {
      prefix = ""
    }

    transition {
      days          = 30
      storage_class = "INTELLIGENT_TIERING"
    }

    # Only apply to large files (videos)
    filter {
      object_size_greater_than = 10485760  # 10MB
    }
  }
}

resource "aws_s3_bucket_cors_configuration" "media" {
  bucket = aws_s3_bucket.media.id

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET", "PUT", "POST"]
    allowed_origins = var.cors_origins
    expose_headers  = ["ETag"]
    max_age_seconds = 3600
  }
}

# S3 Event Notification to EventBridge
resource "aws_s3_bucket_notification" "media" {
  bucket      = aws_s3_bucket.media.id
  eventbridge = true
}
```

**modules/media-storage/variables.tf**

```hcl
variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
}

variable "kms_key_arn" {
  description = "KMS key ARN for S3 encryption"
  type        = string
}

variable "cors_origins" {
  description = "Allowed CORS origins"
  type        = list(string)
  default     = ["https://*.twc.com"]
}
```

**modules/media-storage/outputs.tf**

```hcl
output "bucket_name" {
  value = aws_s3_bucket.media.id
}

output "bucket_arn" {
  value = aws_s3_bucket.media.arn
}

output "bucket_regional_domain" {
  value = aws_s3_bucket.media.bucket_regional_domain_name
}
```

---

#### Module: media-processing

**modules/media-processing/main.tf**

```hcl
# Lambda: Media Router
resource "aws_lambda_function" "router" {
  function_name = "twc-media-router-${var.environment}"
  role          = aws_iam_role.lambda_role.arn
  handler       = "index.handler"
  runtime       = "nodejs20.x"
  timeout       = 30
  memory_size   = 256

  filename         = var.router_zip_path
  source_code_hash = filebase64sha256(var.router_zip_path)

  environment {
    variables = {
      ENVIRONMENT           = var.environment
      IMAGE_PROCESSOR_ARN   = aws_lambda_function.image_processor.arn
      DOC_PROCESSOR_ARN     = aws_lambda_function.doc_processor.arn
      VIDEO_QUEUE_URL       = aws_sqs_queue.video_queue.url
      DYNAMODB_TABLE        = var.dynamodb_table_name
    }
  }

  tracing_config {
    mode = "Active"
  }

  tags = var.tags
}

# Lambda: Image Processor
resource "aws_lambda_function" "image_processor" {
  function_name = "twc-media-image-processor-${var.environment}"
  role          = aws_iam_role.lambda_role.arn
  handler       = "index.handler"
  runtime       = "nodejs20.x"
  timeout       = 60
  memory_size   = 1024

  filename         = var.image_processor_zip_path
  source_code_hash = filebase64sha256(var.image_processor_zip_path)

  layers = [
    var.sharp_layer_arn
  ]

  environment {
    variables = {
      ENVIRONMENT    = var.environment
      S3_BUCKET      = var.s3_bucket_name
      DYNAMODB_TABLE = var.dynamodb_table_name
    }
  }

  tracing_config {
    mode = "Active"
  }

  tags = var.tags
}

# Lambda: Document Processor
resource "aws_lambda_function" "doc_processor" {
  function_name = "twc-media-doc-processor-${var.environment}"
  role          = aws_iam_role.lambda_role.arn
  handler       = "index.handler"
  runtime       = "nodejs20.x"
  timeout       = 60
  memory_size   = 512

  filename         = var.doc_processor_zip_path
  source_code_hash = filebase64sha256(var.doc_processor_zip_path)

  environment {
    variables = {
      ENVIRONMENT    = var.environment
      S3_BUCKET      = var.s3_bucket_name
      DYNAMODB_TABLE = var.dynamodb_table_name
    }
  }

  tracing_config {
    mode = "Active"
  }

  tags = var.tags
}

# EventBridge Rule: Route S3 uploads to Lambda
resource "aws_cloudwatch_event_rule" "s3_upload" {
  name        = "twc-media-upload-${var.environment}"
  description = "Route S3 media uploads to processing"

  event_pattern = jsonencode({
    source      = ["aws.s3"]
    detail-type = ["Object Created"]
    detail = {
      bucket = {
        name = [var.s3_bucket_name]
      }
      object = {
        key = [{
          prefix = "uploads/"
        }]
      }
    }
  })

  tags = var.tags
}

resource "aws_cloudwatch_event_target" "router" {
  rule      = aws_cloudwatch_event_rule.s3_upload.name
  target_id = "media-router"
  arn       = aws_lambda_function.router.arn
}

resource "aws_lambda_permission" "eventbridge" {
  statement_id  = "AllowEventBridge"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.router.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.s3_upload.arn
}

# SQS: Video Processing Queue
resource "aws_sqs_queue" "video_queue" {
  name                       = "twc-media-video-${var.environment}"
  visibility_timeout_seconds = 1800  # 30 minutes
  message_retention_seconds  = 86400 # 24 hours
  receive_wait_time_seconds  = 20    # Long polling

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.dlq.arn
    maxReceiveCount     = 3
  })

  tags = var.tags
}

resource "aws_sqs_queue" "dlq" {
  name                      = "twc-media-dlq-${var.environment}"
  message_retention_seconds = 1209600  # 14 days

  tags = var.tags
}

# ECS Cluster for Video Processing
resource "aws_ecs_cluster" "media" {
  name = "twc-media-${var.environment}"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = var.tags
}

resource "aws_ecs_cluster_capacity_providers" "media" {
  cluster_name = aws_ecs_cluster.media.name

  capacity_providers = ["FARGATE", "FARGATE_SPOT"]

  default_capacity_provider_strategy {
    base              = 1
    weight            = 100
    capacity_provider = "FARGATE_SPOT"
  }
}

# ECS Task Definition: Video Processor
resource "aws_ecs_task_definition" "video_processor" {
  family                   = "twc-media-video-${var.environment}"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 1024
  memory                   = 2048
  execution_role_arn       = aws_iam_role.ecs_execution_role.arn
  task_role_arn            = aws_iam_role.ecs_task_role.arn

  container_definitions = jsonencode([
    {
      name      = "ffmpeg"
      image     = "${var.ecr_repository_url}:${var.video_processor_image_tag}"
      essential = true

      environment = [
        { name = "S3_BUCKET", value = var.s3_bucket_name },
        { name = "DYNAMODB_TABLE", value = var.dynamodb_table_name },
        { name = "ENVIRONMENT", value = var.environment }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.video.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "video"
        }
      }
    }
  ])

  tags = var.tags
}

# Lambda: SQS to ECS (triggers Fargate tasks)
resource "aws_lambda_function" "video_trigger" {
  function_name = "twc-media-video-trigger-${var.environment}"
  role          = aws_iam_role.lambda_role.arn
  handler       = "index.handler"
  runtime       = "nodejs20.x"
  timeout       = 30
  memory_size   = 256

  filename         = var.video_trigger_zip_path
  source_code_hash = filebase64sha256(var.video_trigger_zip_path)

  environment {
    variables = {
      ECS_CLUSTER         = aws_ecs_cluster.media.arn
      ECS_TASK_DEFINITION = aws_ecs_task_definition.video_processor.arn
      ECS_SUBNETS         = join(",", var.private_subnet_ids)
      ECS_SECURITY_GROUP  = var.ecs_security_group_id
    }
  }

  tags = var.tags
}

resource "aws_lambda_event_source_mapping" "video_queue" {
  event_source_arn = aws_sqs_queue.video_queue.arn
  function_name    = aws_lambda_function.video_trigger.arn
  batch_size       = 1
}

# CloudWatch Log Groups
resource "aws_cloudwatch_log_group" "router" {
  name              = "/aws/lambda/twc-media-router-${var.environment}"
  retention_in_days = 30
  tags              = var.tags
}

resource "aws_cloudwatch_log_group" "image" {
  name              = "/aws/lambda/twc-media-image-processor-${var.environment}"
  retention_in_days = 30
  tags              = var.tags
}

resource "aws_cloudwatch_log_group" "doc" {
  name              = "/aws/lambda/twc-media-doc-processor-${var.environment}"
  retention_in_days = 30
  tags              = var.tags
}

resource "aws_cloudwatch_log_group" "video" {
  name              = "/ecs/twc-media-video-${var.environment}"
  retention_in_days = 30
  tags              = var.tags
}
```

**modules/media-processing/iam.tf**

```hcl
# Lambda Execution Role
resource "aws_iam_role" "lambda_role" {
  name = "twc-media-lambda-${var.environment}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })

  tags = var.tags
}

resource "aws_iam_role_policy" "lambda_policy" {
  name = "twc-media-lambda-policy"
  role = aws_iam_role.lambda_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "arn:aws:logs:*:*:*"
      },
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:CopyObject"
        ]
        Resource = "${var.s3_bucket_arn}/*"
      },
      {
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:UpdateItem",
          "dynamodb:Query"
        ]
        Resource = var.dynamodb_table_arn
      },
      {
        Effect = "Allow"
        Action = [
          "sqs:SendMessage",
          "sqs:ReceiveMessage",
          "sqs:DeleteMessage",
          "sqs:GetQueueAttributes"
        ]
        Resource = aws_sqs_queue.video_queue.arn
      },
      {
        Effect = "Allow"
        Action = [
          "lambda:InvokeFunction"
        ]
        Resource = [
          aws_lambda_function.image_processor.arn,
          aws_lambda_function.doc_processor.arn
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "ecs:RunTask"
        ]
        Resource = aws_ecs_task_definition.video_processor.arn
      },
      {
        Effect = "Allow"
        Action = [
          "iam:PassRole"
        ]
        Resource = [
          aws_iam_role.ecs_execution_role.arn,
          aws_iam_role.ecs_task_role.arn
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "xray:PutTraceSegments",
          "xray:PutTelemetryRecords"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "cloudwatch:PutMetricData"
        ]
        Resource = "*"
      }
    ]
  })
}

# ECS Execution Role
resource "aws_iam_role" "ecs_execution_role" {
  name = "twc-media-ecs-execution-${var.environment}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })

  tags = var.tags
}

resource "aws_iam_role_policy_attachment" "ecs_execution" {
  role       = aws_iam_role.ecs_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# ECS Task Role
resource "aws_iam_role" "ecs_task_role" {
  name = "twc-media-ecs-task-${var.environment}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })

  tags = var.tags
}

resource "aws_iam_role_policy" "ecs_task_policy" {
  name = "twc-media-ecs-task-policy"
  role = aws_iam_role.ecs_task_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject"
        ]
        Resource = "${var.s3_bucket_arn}/*"
      },
      {
        Effect = "Allow"
        Action = [
          "dynamodb:UpdateItem"
        ]
        Resource = var.dynamodb_table_arn
      },
      {
        Effect = "Allow"
        Action = [
          "cloudwatch:PutMetricData"
        ]
        Resource = "*"
      }
    ]
  })
}
```

**modules/media-processing/variables.tf**

```hcl
variable "environment" {
  type = string
}

variable "aws_region" {
  type = string
}

variable "s3_bucket_name" {
  type = string
}

variable "s3_bucket_arn" {
  type = string
}

variable "dynamodb_table_name" {
  type = string
}

variable "dynamodb_table_arn" {
  type = string
}

variable "router_zip_path" {
  type = string
}

variable "image_processor_zip_path" {
  type = string
}

variable "doc_processor_zip_path" {
  type = string
}

variable "video_trigger_zip_path" {
  type = string
}

variable "sharp_layer_arn" {
  type = string
}

variable "ecr_repository_url" {
  type = string
}

variable "video_processor_image_tag" {
  type    = string
  default = "latest"
}

variable "private_subnet_ids" {
  type = list(string)
}

variable "ecs_security_group_id" {
  type = string
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

---

#### Module: media-database

**modules/media-database/main.tf**

```hcl
# DynamoDB: Media Assets
resource "aws_dynamodb_table" "media_asset" {
  name         = "TWC_MEDIA_ASSET_${upper(var.environment)}"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "tenantId"
  range_key    = "assetId"

  attribute {
    name = "tenantId"
    type = "S"
  }

  attribute {
    name = "assetId"
    type = "S"
  }

  attribute {
    name = "category"
    type = "S"
  }

  attribute {
    name = "status"
    type = "S"
  }

  attribute {
    name = "createdAt"
    type = "S"
  }

  # GSI: Query by category
  global_secondary_index {
    name            = "tenantId-category-index"
    hash_key        = "tenantId"
    range_key       = "category"
    projection_type = "ALL"
  }

  # GSI: Query by status
  global_secondary_index {
    name            = "tenantId-status-index"
    hash_key        = "tenantId"
    range_key       = "status"
    projection_type = "ALL"
  }

  point_in_time_recovery {
    enabled = true
  }

  server_side_encryption {
    enabled     = true
    kms_key_arn = var.kms_key_arn
  }

  tags = var.tags
}

# DynamoDB: Media Categories
resource "aws_dynamodb_table" "media_category" {
  name         = "TWC_MEDIA_CATEGORY_${upper(var.environment)}"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "tenantId"
  range_key    = "categoryId"

  attribute {
    name = "tenantId"
    type = "S"
  }

  attribute {
    name = "categoryId"
    type = "S"
  }

  server_side_encryption {
    enabled     = true
    kms_key_arn = var.kms_key_arn
  }

  tags = var.tags
}

# DynamoDB: Media Versions
resource "aws_dynamodb_table" "media_version" {
  name         = "TWC_MEDIA_VERSION_${upper(var.environment)}"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "tenantAssetId"
  range_key    = "version"

  attribute {
    name = "tenantAssetId"
    type = "S"
  }

  attribute {
    name = "version"
    type = "N"
  }

  server_side_encryption {
    enabled     = true
    kms_key_arn = var.kms_key_arn
  }

  tags = var.tags
}

# DynamoDB: Customer Media
resource "aws_dynamodb_table" "customer_media" {
  name         = "TWC_CUSTOMER_MEDIA_${upper(var.environment)}"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "tenantId"
  range_key    = "customerMediaId"

  attribute {
    name = "tenantId"
    type = "S"
  }

  attribute {
    name = "customerMediaId"
    type = "S"
  }

  attribute {
    name = "customerId"
    type = "S"
  }

  attribute {
    name = "createdAt"
    type = "S"
  }

  # GSI: Query by customer
  global_secondary_index {
    name            = "tenantId-customerId-index"
    hash_key        = "tenantId"
    range_key       = "customerId"
    projection_type = "ALL"
  }

  point_in_time_recovery {
    enabled = true
  }

  server_side_encryption {
    enabled     = true
    kms_key_arn = var.kms_key_arn
  }

  tags = var.tags
}
```

---

#### Environment Configuration

**environments/prod/main.tf**

```hcl
terraform {
  required_version = ">= 1.5.0"

  backend "s3" {
    bucket         = "twc-terraform-state"
    key            = "media/prod/terraform.tfstate"
    region         = "ap-southeast-2"
    encrypt        = true
    dynamodb_table = "twc-terraform-locks"
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = "twc-media"
      Environment = "prod"
      ManagedBy   = "terraform"
    }
  }
}

module "storage" {
  source = "../../modules/media-storage"

  environment  = "prod"
  kms_key_arn  = aws_kms_key.media.arn
  cors_origins = ["https://app.twc.com", "https://admin.twc.com"]
}

module "database" {
  source = "../../modules/media-database"

  environment = "prod"
  kms_key_arn = aws_kms_key.media.arn
  tags        = local.tags
}

module "processing" {
  source = "../../modules/media-processing"

  environment         = "prod"
  aws_region          = var.aws_region
  s3_bucket_name      = module.storage.bucket_name
  s3_bucket_arn       = module.storage.bucket_arn
  dynamodb_table_name = module.database.asset_table_name
  dynamodb_table_arn  = module.database.asset_table_arn

  router_zip_path          = "${path.module}/../../dist/router.zip"
  image_processor_zip_path = "${path.module}/../../dist/image-processor.zip"
  doc_processor_zip_path   = "${path.module}/../../dist/doc-processor.zip"
  video_trigger_zip_path   = "${path.module}/../../dist/video-trigger.zip"

  sharp_layer_arn           = "arn:aws:lambda:ap-southeast-2:xxx:layer:sharp:1"
  ecr_repository_url        = aws_ecr_repository.video.repository_url
  video_processor_image_tag = var.video_processor_version

  private_subnet_ids    = data.aws_subnets.private.ids
  ecs_security_group_id = aws_security_group.ecs.id

  tags = local.tags
}

module "monitoring" {
  source = "../../modules/media-monitoring"

  environment        = "prod"
  lambda_router_name = module.processing.router_function_name
  lambda_image_name  = module.processing.image_processor_function_name
  lambda_doc_name    = module.processing.doc_processor_function_name
  sqs_queue_name     = module.processing.video_queue_name
  dlq_name           = module.processing.dlq_name
  ecs_cluster_name   = module.processing.ecs_cluster_name
  alert_email        = var.alert_email
  slack_webhook_url  = var.slack_webhook_url
  pagerduty_endpoint = var.pagerduty_endpoint

  tags = local.tags
}

# KMS Key for encryption
resource "aws_kms_key" "media" {
  description             = "TWC Media encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  tags = local.tags
}

resource "aws_kms_alias" "media" {
  name          = "alias/twc-media-prod"
  target_key_id = aws_kms_key.media.key_id
}

locals {
  tags = {
    Project     = "twc-media"
    Environment = "prod"
  }
}
```

---

## Upload Flow (Phase 1)

In Phase 1, users manually upload each variant. No auto-generation.

### Upload Guidelines (shown in UI)

| Variant | Max Size | Allowed Formats | Notes |
|---------|----------|-----------------|-------|
| SMS | 1.2 MB | JPG, PNG, GIF, PDF | Compressed for MMS |
| WhatsApp | 5 MB (images), 16 MB (docs/video) | JPG, PNG, GIF, DOCX, XLSX, CSV, MP4 | No PDF for images |
| Email | 25 MB | All supported formats | Most flexible |
| Original | 100 MB | All supported formats | Full quality archive |

**Automatic conversions on upload:**
- HEIC → JPEG
- MOV/WebM → MP4
- Generate thumbnails for all types

### Upload Flow

```
User selects files in UI
         │
         ▼
┌─────────────────────────────┐
│  For each file:             │
│  1. Get presigned URL       │
│  2. Upload direct to S3     │
│  3. Tag with variant type   │
└─────────────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│  Create/Update Asset        │
│  - Add variant to map       │
│  - Set status = active      │
└─────────────────────────────┘
```

### Phase 2 Enhancement

Auto-generate SMS/email variants from a single original upload using Lambda + Sharp/Ghostscript.

---

## Backoffice UI

### Navigation

```
Backoffice Menu
├── Dashboard
├── Customers
├── Orders
├── ...
└── Media Library  ← NEW
```

### Screen: Media Library List

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Media Library                                            [+ Upload]   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 🔍 Search media...                    Category: [All ▼]         │   │
│  │                                       Status:   [Active ▼]      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌───────────┬────────────────────────┬──────────┬─────────┬────────┐  │
│  │ Preview   │ Title                  │ Category │ Expires │ Actions│  │
│  ├───────────┼────────────────────────┼──────────┼─────────┼────────┤  │
│  │ [thumb]   │ Summer 2025 Collection │ Collection│ Jun 1   │ ⋮     │  │
│  │           │ PDF • 2.5 MB • v2      │          │         │        │  │
│  ├───────────┼────────────────────────┼──────────┼─────────┼────────┤  │
│  │ [thumb]   │ VIP Event Invitation   │ Invitation│ Mar 15  │ ⋮     │  │
│  │           │ PNG • 1.2 MB • v1      │          │         │        │  │
│  ├───────────┼────────────────────────┼──────────┼─────────┼────────┤  │
│  │ [thumb]   │ Treatment Waiver Form  │ Waiver   │ -       │ ⋮     │  │
│  │           │ PDF • 450 KB • v3      │          │         │        │  │
│  └───────────┴────────────────────────┴──────────┴─────────┴────────┘  │
│                                                                         │
│  Showing 1-20 of 45                              [< Prev] [Next >]     │
└─────────────────────────────────────────────────────────────────────────┘
```

### Screen: Upload Media (Modal)

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Upload Media                                                     [×]  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Title *                                                                │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Summer 2025 Collection                                          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Description                                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ New arrivals lookbook for Summer 2025 season                    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Category *                              Expiry Date                    │
│  ┌──────────────────────┐               ┌──────────────────────┐       │
│  │ Collection        ▼  │               │ 01/06/2025       📅  │       │
│  └──────────────────────┘               └──────────────────────┘       │
│                                                                         │
│  Tags                                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ [summer] [2025] [lookbook] [+ Add tag]                          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│  Upload Variants                                                        │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ SMS Version (max 1.2 MB)                                          │ │
│  │ ┌─────────────────────────────────────────────────────────────┐   │ │
│  │ │  📁 Drag & drop or click to browse                          │   │ │
│  │ │     JPG, PNG, PDF                                            │   │ │
│  │ └─────────────────────────────────────────────────────────────┘   │ │
│  │ ✓ Summer_2025_SMS.jpg (850 KB)                           [×]     │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ WhatsApp Version (max 5 MB)                                       │ │
│  │ ┌─────────────────────────────────────────────────────────────┐   │ │
│  │ │  📁 Drag & drop or click to browse                          │   │ │
│  │ │     JPG, PNG only (no PDF)                                   │   │ │
│  │ └─────────────────────────────────────────────────────────────┘   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ Email Version (max 10 MB)                                         │ │
│  │ ┌─────────────────────────────────────────────────────────────┐   │ │
│  │ │  📁 Drag & drop or click to browse                          │   │ │
│  │ │     PDF, JPG, PNG supported                                  │   │ │
│  │ └─────────────────────────────────────────────────────────────┘   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ Original (max 10 MB) - Optional                                   │ │
│  │ ┌─────────────────────────────────────────────────────────────┐   │ │
│  │ │  📁 Drag & drop or click to browse                          │   │ │
│  │ │     Full quality archive copy                                │   │ │
│  │ └─────────────────────────────────────────────────────────────┘   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ⓘ At least one variant (SMS, WhatsApp, or Email) is required         │
│                                                                         │
│                                          [Cancel]  [Upload & Save]     │
└─────────────────────────────────────────────────────────────────────────┘
```

### Screen: Asset Detail (Side Panel or Full Page)

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Summer 2025 Collection                       [Edit] [Delete] [Archive]│
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ⚠️ This asset expired on Jun 1, 2025                    [Dismiss]     │
│                                                                         │
│  ┌──────────────────────────────┐  Status: ● Active (or ⚠ Expired)     │
│  │                              │  Category: Collection                 │
│  │                              │  Created: Mar 1, 2025 by Jane Smith   │
│  │      [Preview Image]         │  Updated: Mar 5, 2025 by John Doe     │
│  │                              │  Expires: Jun 1, 2025                 │
│  │                              │  Version: 2                           │
│  │                              │                                       │
│  └──────────────────────────────┘  Tags: summer, 2025, lookbook         │
│                                                                         │
│  Description                                                            │
│  New arrivals lookbook for Summer 2025 season                          │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│  Variants                                                               │
│                                                                         │
│  ┌─────────────┬─────────────┬─────────────┬─────────────┐             │
│  │  SMS        │  WhatsApp   │  Email      │  Original   │             │
│  │  [thumb]    │  [thumb]    │  [thumb]    │  [icon]     │             │
│  │  JPG 850KB  │  JPG 2.5MB  │  PDF 2.5MB  │  PDF 8.5MB  │             │
│  │  [Copy URL] │  [Copy URL] │  [Copy URL] │  [Download] │             │
│  │  [Download] │  [Download] │  [Download] │             │             │
│  └─────────────┴─────────────┴─────────────┴─────────────┘             │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│  Version History                                    [Upload New Version]│
│                                                                         │
│  v2 (current) • Mar 5, 2025 • John Doe                                 │
│     "Updated cover image"                                               │
│                                                                         │
│  v1 • Mar 1, 2025 • Jane Smith                                         │
│     Initial upload                                           [Restore] │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Expired assets:**
- Show warning banner at top of detail view
- Show ⚠ icon in list view
- Still accessible (not auto-deleted)
- User can manually delete or extend expiry date

---

## Integration with Outreach

When composing an SMS or email in the clienteling app:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Compose SMS                                                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  To: Sarah Johnson (+61 412 345 678)                                   │
│                                                                         │
│  Message:                                                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Hi Sarah! Our new Summer collection is here. Check out the      │   │
│  │ lookbook:                                                        │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Attachments:                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ [+ Add from Media Library]                                       │   │
│  │                                                                  │   │
│  │ ┌─────────┐                                                     │   │
│  │ │ [thumb] │ Summer 2025 Collection (SMS version)          [×]  │   │
│  │ │         │ 50 KB                                               │   │
│  │ └─────────┘                                                     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│                                                      [Send]            │
└─────────────────────────────────────────────────────────────────────────┘
```

The "Add from Media Library" button opens a picker that:
1. Shows active media assets
2. Auto-selects the appropriate variant based on channel:
   - SMS → uses `sms` variant
   - WhatsApp → uses `whatsapp` variant
   - Email → uses `email` variant
3. Returns the presigned URL to embed

**WhatsApp Integration Note:**
When sending via WhatsApp, the outreach service must:
1. Fetch the `whatsapp` variant URL
2. Upload to Meta's servers via WhatsApp Cloud API
3. Use the returned `media_id` in the message
4. Handle media_id expiry (~24h) for scheduled messages

---

## Technical Stack

| Component | Technology |
|-----------|------------|
| Frontend | React (your existing framework) |
| API | Your existing API framework |
| Storage | S3 (private, presigned URLs) |
| Database | DynamoDB |
| Image Processing | Lambda + Sharp |
| PDF Processing | Lambda + pdf-lib or Ghostscript |

---

## Customer Media

In addition to the shared media library, the system captures media exchanged between staff and customers. This enables:

- **Inbound documents** - Signed waivers, ID photos, customer-provided images
- **Outbound documents** - Personalised lookbooks, custom quotes, legal forms sent
- **Full history** - View all media associated with a customer in their profile

### Storage Strategy

To avoid storage explosion from campaigns (sending same image to 10,000 customers), we use a reference-based approach:

| Media Type | Storage | Rationale |
|------------|---------|-----------|
| **Inbound from customer** | Full copy (S3) | Unique content, legal/compliance value |
| **Outbound - from library** | Reference only | Link to TWC_MEDIA_ASSET, no duplication |
| **Outbound - customised** | Full copy (S3) | Personalised lookbooks, custom quotes |

### Retention Policy

Customer media is retained **indefinitely** for:
- Legal documents (waivers, contracts)
- Inbound customer content
- Customised outbound (quotes, personalised lookbooks)

Library-referenced outbound follows the library asset's lifecycle.

---

### Customer Media (DynamoDB)

```
Table: TWC_CUSTOMER_MEDIA

Partition Key: tenantId (String)
Sort Key: customerId#mediaId (String)

GSI1: tenantId-customerId-index
  - PK: tenantId
  - SK: customerId#createdAt

GSI2: tenantId-messageId-index (for linking to conversations)
  - PK: tenantId
  - SK: messageId
```

#### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| **tenantId** | String | Yes | Retailer/tenant ID |
| **customerId** | String | Yes | Customer ID |
| **mediaId** | String | Yes | Unique media ID (ULID) |
| **direction** | String | Yes | `inbound` or `outbound` |
| **source** | String | Yes | `customer`, `library`, `staff_upload` |
| **mediaLibraryAssetId** | String | Conditional | If source = `library`, reference to TWC_MEDIA_ASSET |
| **s3Key** | String | Conditional | If source != `library`, S3 storage key |
| **fileName** | String | Yes | Original file name |
| **mimeType** | String | Yes | MIME type |
| **fileSize** | Number | Yes | File size in bytes |
| **channel** | String | Yes | `whatsapp`, `sms`, `email` |
| **messageId** | String | No | Link to conversation/message |
| **conversationId** | String | No | Link to conversation thread |
| **tags** | List<String> | No | User-defined tags (e.g., "waiver", "quote", "id-document") |
| **documentType** | String | No | Predefined type: `waiver`, `quote`, `lookbook`, `id`, `other` |
| **notes** | String | No | Staff notes about this media |
| **sentBy** | String | Conditional | Staff ID (for outbound) |
| **receivedAt** | String | Conditional | Timestamp (for inbound) |
| **createdAt** | String | Yes | Record creation timestamp |

#### Document Types (Predefined)

| Value | Description |
|-------|-------------|
| `waiver` | Waiver forms (sent or signed) |
| `quote` | Price quotes, proposals |
| `lookbook` | Personalised lookbooks |
| `id` | ID documents, verification |
| `receipt` | Receipts, invoices |
| `other` | Uncategorised |

---

### S3 Structure (Customer Media)

```
twc-media-{env}/
├── {tenantId}/
│   ├── assets/                    # Shared library (existing)
│   │   └── {assetId}/
│   │       └── ...
│   └── customers/                 # Customer-specific media (NEW)
│       └── {customerId}/
│           ├── inbound/
│           │   ├── {mediaId}.jpg
│           │   └── {mediaId}.pdf
│           └── outbound/
│               ├── {mediaId}.pdf   # Customised only
│               └── ...
```

**Note:** Outbound from library assets are NOT stored here - only a reference in DynamoDB.

---

### Customer Media API

Base URL: `/api/v1/customers/{customerId}/media`

#### 1. List Customer Media

```http
GET /api/v1/customers/{customerId}/media?direction=inbound&documentType=waiver&limit=20
```

**Query Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| direction | String | `inbound`, `outbound`, or omit for all |
| documentType | String | Filter by document type |
| channel | String | Filter by channel |
| tags | String | Comma-separated tags |
| limit | Number | Page size (default: 20) |
| cursor | String | Pagination cursor |

**Response:**
```json
{
  "items": [
    {
      "mediaId": "01HQABC...",
      "direction": "inbound",
      "source": "customer",
      "fileName": "signed_waiver.pdf",
      "mimeType": "application/pdf",
      "fileSize": 125000,
      "documentType": "waiver",
      "tags": ["signed", "treatment"],
      "channel": "whatsapp",
      "receivedAt": "2025-03-15T10:30:00Z",
      "thumbnailUrl": "https://presigned..."
    },
    {
      "mediaId": "01HQDEF...",
      "direction": "outbound",
      "source": "library",
      "mediaLibraryAssetId": "01HQXYZ...",
      "fileName": "Waiver_Form.pdf",
      "documentType": "waiver",
      "channel": "whatsapp",
      "sentBy": "user_123",
      "createdAt": "2025-03-10T09:00:00Z",
      "thumbnailUrl": "https://presigned..."
    }
  ],
  "nextCursor": "..."
}
```

#### 2. Get Customer Media Detail

```http
GET /api/v1/customers/{customerId}/media/{mediaId}
```

**Response:**
```json
{
  "mediaId": "01HQABC...",
  "direction": "inbound",
  "source": "customer",
  "fileName": "signed_waiver.pdf",
  "mimeType": "application/pdf",
  "fileSize": 125000,
  "documentType": "waiver",
  "tags": ["signed", "treatment"],
  "notes": "Customer signed in-store on Mar 15",
  "channel": "whatsapp",
  "messageId": "msg_456",
  "conversationId": "conv_789",
  "receivedAt": "2025-03-15T10:30:00Z",
  "url": "https://presigned-download-url...",
  "expiresIn": 3600
}
```

#### 3. Upload Customer Media (Staff Upload)

For staff uploading customised content (lookbooks, quotes) directly to a customer.

```http
POST /api/v1/customers/{customerId}/media/upload-url
```

**Request:**
```json
{
  "fileName": "Sarah_Lookbook_March.pdf",
  "mimeType": "application/pdf",
  "fileSize": 2500000,
  "documentType": "lookbook"
}
```

**Response:**
```json
{
  "uploadId": "upload_xyz",
  "uploadUrl": "https://s3-presigned...",
  "expiresIn": 3600
}
```

#### 4. Create Customer Media Record

After upload completes:

```http
POST /api/v1/customers/{customerId}/media
```

**Request:**
```json
{
  "uploadId": "upload_xyz",
  "direction": "outbound",
  "documentType": "lookbook",
  "tags": ["personalised", "march-2025"],
  "notes": "Created for Sarah's spring wardrobe refresh"
}
```

#### 5. Auto-Capture from Messages (Internal)

When media is sent/received in conversations, the outreach service calls:

```http
POST /api/v1/customers/{customerId}/media/capture
```

**Request (Inbound):**
```json
{
  "direction": "inbound",
  "source": "customer",
  "channel": "whatsapp",
  "messageId": "msg_456",
  "conversationId": "conv_789",
  "s3Key": "tenants/tenant123/customers/cust456/inbound/abc123.jpg",
  "fileName": "photo.jpg",
  "mimeType": "image/jpeg",
  "fileSize": 150000
}
```

**Request (Outbound from Library):**
```json
{
  "direction": "outbound",
  "source": "library",
  "mediaLibraryAssetId": "01HQXYZ...",
  "channel": "whatsapp",
  "messageId": "msg_457",
  "sentBy": "user_123"
}
```

#### 6. Update Customer Media

```http
PUT /api/v1/customers/{customerId}/media/{mediaId}
```

**Request:**
```json
{
  "tags": ["signed", "verified"],
  "documentType": "waiver",
  "notes": "Verified by Jane on Mar 16"
}
```

#### 7. Delete Customer Media

```http
DELETE /api/v1/customers/{customerId}/media/{mediaId}
```

Soft delete - removes from UI but retains for compliance.

---

### Customer Profile UI - Media Tab

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Sarah Chen                                              ⭐ VIP Customer │
│  sarah.chen@email.com | +61 412 345 678                                │
├─────────────────────────────────────────────────────────────────────────┤
│  [Orders]  [Wishlist]  [Notes]  [Messages]  [Media]                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  [📥 Received]  [📤 Sent]  [All]              [+ Upload]  🔍 Search    │
│                                                                         │
│  Filter: [All Types ▼]  [All Channels ▼]  Tags: [waiver ×] [+ Add]    │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │   │
│  │  │  📄      │  │  🖼️      │  │  📄      │  │  📄      │        │   │
│  │  │          │  │          │  │          │  │          │        │   │
│  │  │ Signed   │  │ Product  │  │ Waiver   │  │ Quote    │        │   │
│  │  │ Waiver   │  │ Photo    │  │ Form     │  │ v2       │        │   │
│  │  │          │  │          │  │          │  │          │        │   │
│  │  │ Mar 15   │  │ Mar 12   │  │ Mar 10   │  │ Mar 8    │        │   │
│  │  │ 📥 In    │  │ 📥 In    │  │ 📤 Out   │  │ 📤 Out   │        │   │
│  │  │ WhatsApp │  │ WhatsApp │  │ WhatsApp │  │ Email    │        │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Showing 1-12 of 24                              [< Prev] [Next >]     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Media Detail Modal (Customer Media)

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Signed Waiver                                                    [×]   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌────────────────────────────────┐  Direction: 📥 Received            │
│  │                                │  Channel: WhatsApp                  │
│  │                                │  Received: Mar 15, 2025 10:30 AM    │
│  │      [PDF Preview]             │  Type: Waiver                       │
│  │                                │  File: signed_waiver.pdf (125 KB)   │
│  │                                │                                     │
│  │                                │  Tags: [signed] [treatment] [+]     │
│  └────────────────────────────────┘                                     │
│                                                                         │
│  Notes                                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Customer signed in-store on Mar 15. Verified ID.                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Linked Conversation                                                    │
│  └─▶ View in Messages                                                  │
│                                                                         │
│                                      [Download]  [Delete]  [Save]      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Next Steps

1. **Confirm data model** - Review fields, add/remove as needed
2. **Create DynamoDB tables** - Terraform/CloudFormation
3. **Implement APIs** - In your existing backend
4. **Build React components** - Upload, List, Detail views
5. **Set up Lambda** - Variant generation
6. **Integrate with outreach** - Media picker component
7. **Build customer media capture** - Auto-save from message flow
8. **Build customer profile media tab** - View/manage customer media
