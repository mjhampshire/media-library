# TWC Media Library - Project Status

**Last Updated:** March 2025
**Status:** Design Complete - Ready for Development

---

## What It Does

A media library for the TWC backoffice that allows retailers to:
- Upload media assets (images, PDFs, videos)
- Auto-generate channel-optimized variants (SMS, email, thumbnail)
- Organize with categories and tags
- Set expiry dates
- Track versions
- Use in outreach messages (SMS, email campaigns)

---

## Use Cases

| Use Case | Example |
|----------|---------|
| Collection Lookbooks | Upload PDF (email) + compressed image (SMS) |
| Event Invitations | Upload invitation images for different channels |
| Promotional Assets | Sale banners, new arrival highlights |

---

## Documents

| Document | Description |
|----------|-------------|
| `docs/DESIGN.md` | Full design: data model, APIs, UI mockups, processing flow |
| `docs/DYNAMODB_TABLES.md` | DynamoDB table definitions with AWS CLI/Terraform |

---

## Data Model Summary

### TWC_MEDIA_CATEGORY (DynamoDB)

User-configurable categories per tenant.

| Field | Type | Description |
|-------|------|-------------|
| tenantId | String | Partition key |
| categoryId | String | Sort key - URL-safe slug |
| name | String | Display name |
| sortOrder | Number | For ordering in dropdowns |
| isDefault | Boolean | System default (can't delete) |

### TWC_MEDIA_ASSET (DynamoDB)

| Field | Type | Description |
|-------|------|-------------|
| tenantId | String | Partition key - retailer ID |
| assetId | String | Sort key - ULID |
| title | String | Display name |
| description | String | Optional description |
| category | String | Category ID (user-configurable) |
| tags | List | Searchable tags |
| status | String | active, expired, archived |
| expiresAt | String | Optional expiry (shows warning when passed) |
| currentVersion | Number | Current version number |
| variants | Map | Manually uploaded variants (original, sms, email) |
| createdAt | String | ISO timestamp |
| createdBy | String | User ID |
| updatedAt | String | ISO timestamp |
| updatedBy | String | User ID |

### TWC_MEDIA_VERSION (DynamoDB)

Stores version history for each asset.

---

## API Summary

### Media Assets

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/media` | List assets (with filters) |
| GET | `/api/v1/media/{assetId}` | Get asset details |
| POST | `/api/v1/media/upload-url` | Get presigned upload URL |
| POST | `/api/v1/media` | Create asset after upload |
| PUT | `/api/v1/media/{assetId}` | Update metadata |
| DELETE | `/api/v1/media/{assetId}` | Delete asset |
| POST | `/api/v1/media/{assetId}/versions` | Upload new version |
| GET | `/api/v1/media/{assetId}/versions` | Get version history |
| GET | `/api/v1/media/{assetId}/url?variant=sms` | Get variant URL |

### Categories

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/media/categories` | List categories |
| POST | `/api/v1/media/categories` | Create category |
| PUT | `/api/v1/media/categories/{id}` | Update category |
| DELETE | `/api/v1/media/categories/{id}` | Delete category |

---

## UI Screens

1. **Media Library List** - Grid/table of all assets with search, filter by category/status
2. **Upload Modal** - Drag-drop upload with metadata fields
3. **Asset Detail** - Preview, variants, version history
4. **Media Picker** - For outreach composition (auto-selects correct variant)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Frontend | React (existing framework) |
| API | Existing backend |
| Storage | S3 (private, presigned URLs) |
| Database | DynamoDB |

**Phase 2:** Lambda + Sharp for auto-generating variants from a single upload.

---

## Architecture (Phase 1)

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Backoffice    │────▶│   Your API      │────▶│   DynamoDB      │
│   (React)       │     │                 │     │                 │
└────────┬────────┘     └─────────────────┘     └─────────────────┘
         │
         │ (direct upload via presigned URL)
         ▼
┌─────────────────┐
│   S3 Bucket     │
│   (private)     │
└─────────────────┘
```

User manually uploads each variant (SMS, email, original) with file size guidelines.

---

## Next Steps (Phase 1)

1. [ ] Review design docs, confirm requirements
2. [ ] Create DynamoDB tables (Terraform)
3. [ ] Create S3 bucket with lifecycle policies
4. [ ] Implement API endpoints in your backend
5. [ ] Build React components (list, upload, detail, category management)
6. [ ] Integrate media picker into outreach composer

## Phase 2 (Future)

- [ ] Auto-generate variants via Lambda + Sharp
- [ ] Thumbnail generation for previews
- [ ] Video support
