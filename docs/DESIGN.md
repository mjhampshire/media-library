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

## Upload Flow (Phase 1)

In Phase 1, users manually upload each variant. No auto-generation.

### Upload Guidelines (shown in UI)

| Variant | Max Size | Allowed Formats | Notes |
|---------|----------|-----------------|-------|
| SMS | 1.2 MB | JPG, PNG, PDF | Compressed for MMS |
| WhatsApp | 5 MB | JPG, PNG | WhatsApp Business API (no PDF) |
| Email | 10 MB | PDF, JPG, PNG | Higher quality for email |
| Original | 10 MB | PDF, JPG, PNG | Full quality archive |

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

## Next Steps

1. **Confirm data model** - Review fields, add/remove as needed
2. **Create DynamoDB tables** - Terraform/CloudFormation
3. **Implement APIs** - In your existing backend
4. **Build React components** - Upload, List, Detail views
5. **Set up Lambda** - Variant generation
6. **Integrate with outreach** - Media picker component
