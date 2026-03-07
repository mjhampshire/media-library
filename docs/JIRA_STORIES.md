# TWC Media Library - Proposed Jira Stories

**Project:** TWC Media Library
**Epic:** Media Library for Outreach Channels

---

## Epic Description

Build a media library for the TWC backoffice that allows retailers to upload and manage media assets for use in outreach messages (SMS, WhatsApp, Email). Users manually upload channel-specific variants (optimized files for each channel) and can select them when composing messages.

---

## Stories

### Phase 1: Core Infrastructure

#### ML-001: Set up DynamoDB tables for media library
**Type:** Task
**Priority:** High
**Story Points:** 3

**Description:**
Create DynamoDB tables for the media library:
- `TWC_MEDIA_ASSET` - Main asset storage with tenant partitioning
- `TWC_MEDIA_CATEGORY` - User-configurable categories per tenant
- `TWC_MEDIA_VERSION` - Version history for assets

**Acceptance Criteria:**
- [ ] Tables created with correct partition/sort keys
- [ ] GSIs configured for category and status queries
- [ ] Terraform/CloudFormation scripts committed
- [ ] Default categories seeded for each tenant (collection, invitation, promotion, product, brand, other)

---

#### ML-002: Set up S3 bucket for media storage
**Type:** Task
**Priority:** High
**Story Points:** 2

**Description:**
Create a private S3 bucket for media asset storage with proper security configuration.

**Acceptance Criteria:**
- [ ] Private bucket created (no public access)
- [ ] Versioning enabled
- [ ] Lifecycle rules configured (Glacier after 1 year for archived)
- [ ] IAM policies for presigned URL generation
- [ ] CORS configuration for direct uploads from browser

---

#### ML-003: Implement presigned URL generation API
**Type:** Story
**Priority:** High
**Story Points:** 3

**Description:**
As a backoffice user, I want to get a presigned URL for uploading files directly to S3, so that uploads are fast and don't go through the API server.

**Acceptance Criteria:**
- [ ] `POST /api/v1/media/upload-url` endpoint created
- [ ] Validates file size limits per variant type:
  - SMS: 1.2 MB max
  - WhatsApp: 5 MB max
  - Email: 10 MB max
  - Original: 10 MB max
- [ ] Validates MIME type per variant:
  - SMS: image/jpeg, image/png, application/pdf
  - WhatsApp: image/jpeg, image/png only (no PDF)
  - Email: image/jpeg, image/png, application/pdf
- [ ] Returns presigned URL valid for 1 hour
- [ ] Returns uploadId for tracking

---

### Phase 1: Media Management APIs

#### ML-004: Implement Create Asset API
**Type:** Story
**Priority:** High
**Story Points:** 5

**Description:**
As a backoffice user, I want to create a new media asset with uploaded variants, so that it's available for use in outreach messages.

**Acceptance Criteria:**
- [ ] `POST /api/v1/media` endpoint created
- [ ] Accepts title, description, category, tags, expiresAt
- [ ] Accepts variants map with uploaded file references
- [ ] Validates at least one channel variant (sms, whatsapp, or email) is provided
- [ ] Creates thumbnail from first image variant (or PDF first page)
- [ ] Stores asset in DynamoDB with status "active"
- [ ] Returns created asset with presigned URLs for variants

---

#### ML-005: Implement List Assets API
**Type:** Story
**Priority:** High
**Story Points:** 3

**Description:**
As a backoffice user, I want to browse all media assets with filtering and pagination, so that I can find assets quickly.

**Acceptance Criteria:**
- [ ] `GET /api/v1/media` endpoint created
- [ ] Supports filtering by category, status, search term
- [ ] Returns paginated results with cursor
- [ ] Returns thumbnail URLs for list display
- [ ] Orders by createdAt descending (newest first)

---

#### ML-006: Implement Get Asset Detail API
**Type:** Story
**Priority:** High
**Story Points:** 2

**Description:**
As a backoffice user, I want to view all details of a media asset including all variants, so that I can manage it effectively.

**Acceptance Criteria:**
- [ ] `GET /api/v1/media/{assetId}` endpoint created
- [ ] Returns full asset details with metadata
- [ ] Returns presigned URLs for all variants
- [ ] Returns version history summary

---

#### ML-007: Implement Update Asset API
**Type:** Story
**Priority:** Medium
**Story Points:** 2

**Description:**
As a backoffice user, I want to update asset metadata (title, description, tags, expiry), so that I can keep assets current.

**Acceptance Criteria:**
- [ ] `PUT /api/v1/media/{assetId}` endpoint created
- [ ] Can update title, description, tags, expiresAt, status
- [ ] Updates updatedAt and updatedBy fields
- [ ] Cannot change category (would require re-indexing)

---

#### ML-008: Implement Delete/Archive Asset API
**Type:** Story
**Priority:** Medium
**Story Points:** 2

**Description:**
As a backoffice user, I want to archive or delete media assets, so that I can manage my media library effectively.

**Acceptance Criteria:**
- [ ] `DELETE /api/v1/media/{assetId}` endpoint created
- [ ] Soft delete - sets status to "archived"
- [ ] Archived assets not shown in default list/picker
- [ ] S3 files retained (not deleted) for audit trail

---

#### ML-009: Implement Upload New Version API
**Type:** Story
**Priority:** Medium
**Story Points:** 3

**Description:**
As a backoffice user, I want to upload a new version of an existing asset, so that I can update content while preserving history.

**Acceptance Criteria:**
- [ ] `POST /api/v1/media/{assetId}/versions` endpoint created
- [ ] Creates new version record in TWC_MEDIA_VERSION
- [ ] Increments currentVersion on main asset
- [ ] Accepts changelog description
- [ ] Previous versions remain accessible

---

#### ML-010: Implement Get Variant URL API (for outreach service)
**Type:** Story
**Priority:** High
**Story Points:** 2

**Description:**
As the outreach service, I need to get a presigned URL for a specific variant of a media asset, so that I can attach it to messages.

**Acceptance Criteria:**
- [ ] `GET /api/v1/media/{assetId}/url?variant={variant}` endpoint created
- [ ] Returns presigned URL for specified variant (sms, whatsapp, email)
- [ ] Returns 404 if variant not available
- [ ] Includes mimeType and fileSize in response
- [ ] URL expires in 1 hour (configurable)

---

### Phase 1: Category Management

#### ML-011: Implement Category CRUD APIs
**Type:** Story
**Priority:** Medium
**Story Points:** 3

**Description:**
As a backoffice user, I want to manage custom categories for organizing media assets, so that I can organize assets according to my needs.

**Acceptance Criteria:**
- [ ] `GET /api/v1/media/categories` - list all categories
- [ ] `POST /api/v1/media/categories` - create custom category
- [ ] `PUT /api/v1/media/categories/{id}` - update category name/description
- [ ] `DELETE /api/v1/media/categories/{id}` - delete (only if no assets)
- [ ] Default categories cannot be deleted
- [ ] Category IDs are URL-safe slugs

---

### Phase 1: Backoffice UI

#### ML-012: Build Media Library List page
**Type:** Story
**Priority:** High
**Story Points:** 5

**Description:**
As a backoffice user, I want to see a list of all media assets with filtering and search, so that I can browse and manage my media library.

**Acceptance Criteria:**
- [ ] Add "Media Library" to backoffice navigation
- [ ] Display assets in table with preview, title, category, variants, expiry, status
- [ ] Show variant badges (SMS, WA, Email) indicating which variants exist
- [ ] Filter by category and status dropdowns
- [ ] Search by title/description/tags
- [ ] Pagination with cursor-based navigation
- [ ] Show "Expiring Soon" badge for assets expiring within 7 days
- [ ] Show "Expired" badge for past-expiry assets

---

#### ML-013: Build Upload Media modal
**Type:** Story
**Priority:** High
**Story Points:** 8

**Description:**
As a backoffice user, I want to upload new media assets with channel-specific variants, so that I can add assets to my library.

**Acceptance Criteria:**
- [ ] Modal with form: title, description, category, expiry date, tags
- [ ] Four upload zones for variants:
  - SMS (max 1.2 MB, JPG/PNG/PDF)
  - WhatsApp (max 5 MB, JPG/PNG only)
  - Email (max 10 MB, JPG/PNG/PDF)
  - Original (max 10 MB, optional)
- [ ] Client-side validation of file size and type per variant
- [ ] Direct upload to S3 with progress indicator
- [ ] At least one channel variant required
- [ ] Success/error feedback
- [ ] Refresh list on successful upload

---

#### ML-014: Build Asset Detail panel/page
**Type:** Story
**Priority:** High
**Story Points:** 5

**Description:**
As a backoffice user, I want to view asset details including all variants and version history, so that I can manage individual assets.

**Acceptance Criteria:**
- [ ] Display asset metadata (title, category, dates, tags, status)
- [ ] Show expiry warning banner if applicable
- [ ] Display all variants with preview thumbnails
- [ ] "Copy URL" and "Download" actions for each variant
- [ ] Version history list with changelog
- [ ] "Restore" previous version action
- [ ] Edit, Delete, Archive buttons

---

#### ML-015: Build Category Management tab
**Type:** Story
**Priority:** Medium
**Story Points:** 3

**Description:**
As a backoffice user, I want to manage categories from within the Media Library, so that I can organize assets my way.

**Acceptance Criteria:**
- [ ] "Categories" tab in Media Library
- [ ] List all categories with asset count
- [ ] "Add Category" button with name/description form
- [ ] Edit/Delete actions for custom categories
- [ ] Show "Default" badge on system categories
- [ ] Prevent delete if category has assets

---

### Phase 1: Front-Office Integration

#### ML-016: Build Media Picker component for outreach composer
**Type:** Story
**Priority:** High
**Story Points:** 8

**Description:**
As a front-office user composing a message, I want to select media from the library to attach, so that I can include images/PDFs in my outreach.

**Acceptance Criteria:**
- [ ] Modal picker triggered by "Add Media" button in composer
- [ ] Shows channel context (SMS, WhatsApp, or Email) in header
- [ ] Grid of available assets with thumbnails
- [ ] Search and category filter
- [ ] Shows "Ready" badge if asset has the required variant
- [ ] Shows "No [channel]" badge and disables if variant missing
- [ ] Single-select with visual checkmark
- [ ] Shows selected asset details in footer
- [ ] "Add to Message" button returns asset reference to composer
- [ ] Composer displays attached media with remove option

---

#### ML-017: Integrate WhatsApp media upload in outreach service
**Type:** Story
**Priority:** High
**Story Points:** 5

**Description:**
As the outreach service, when sending a WhatsApp message with media, I need to upload the media to Meta's servers first and use the returned media_id.

**Acceptance Criteria:**
- [ ] Fetch WhatsApp variant URL from media library API
- [ ] Download file from S3 presigned URL
- [ ] Upload to WhatsApp Cloud API (`POST /media`)
- [ ] Store returned media_id temporarily (expires in ~24h)
- [ ] Use media_id when sending image message
- [ ] Handle re-upload for scheduled messages if media_id expired
- [ ] Error handling for upload failures

---

### Phase 1: Quality & Polish

#### ML-018: Add expired asset handling
**Type:** Story
**Priority:** Medium
**Story Points:** 3

**Description:**
As a backoffice user, I want expired assets to be clearly marked and optionally hidden, so that I don't accidentally use outdated content.

**Acceptance Criteria:**
- [ ] Background job checks expiresAt daily
- [ ] Sets status to "expired" for past-expiry assets
- [ ] Expired assets hidden from media picker by default
- [ ] Expired assets show warning banner in detail view
- [ ] User can extend expiry date or archive expired assets

---

#### ML-019: Add asset usage tracking
**Type:** Story
**Priority:** Low
**Story Points:** 3

**Description:**
As a backoffice user, I want to see how often media assets are used in messages, so that I can understand what content is effective.

**Acceptance Criteria:**
- [ ] Track asset usage when attached to message
- [ ] Store usage count on asset record
- [ ] Display "Used X times" in asset detail
- [ ] Optional: show last used date

---

## Phase 2 (Future)

### ML-020: Auto-generate variants from original
**Type:** Story
**Priority:** Low
**Story Points:** 13

**Description:**
As a backoffice user, I want to upload a single high-quality original and have SMS/WhatsApp/Email variants generated automatically, so that I don't have to create them manually.

**Acceptance Criteria:**
- [ ] Lambda function triggered on original upload
- [ ] Use Sharp for image resizing/compression
- [ ] Use pdf-lib/Ghostscript for PDF thumbnail generation
- [ ] Generate variants meeting size/format requirements
- [ ] Update asset with generated variants
- [ ] Show processing status in UI

---

## Story Summary

| Phase | Stories | Total Points |
|-------|---------|--------------|
| Infrastructure | ML-001, ML-002, ML-003 | 8 |
| Media APIs | ML-004 to ML-010 | 19 |
| Category APIs | ML-011 | 3 |
| Backoffice UI | ML-012 to ML-015 | 21 |
| Front-Office | ML-016, ML-017 | 13 |
| Quality | ML-018, ML-019 | 6 |
| **Phase 1 Total** | **19 stories** | **~70 points** |
| Phase 2 | ML-020 | 13 |

---

## Suggested Sprint Breakdown

### Sprint 1: Foundation (Infrastructure + Core APIs)
- ML-001: DynamoDB tables
- ML-002: S3 bucket
- ML-003: Presigned URL API
- ML-004: Create Asset API
- ML-005: List Assets API
- ML-006: Get Asset Detail API

### Sprint 2: Complete APIs + Start UI
- ML-007: Update Asset API
- ML-008: Delete/Archive API
- ML-009: Version Upload API
- ML-010: Get Variant URL API
- ML-011: Category CRUD APIs
- ML-012: Media Library List page (start)

### Sprint 3: Backoffice UI
- ML-012: Media Library List page (complete)
- ML-013: Upload Media modal
- ML-014: Asset Detail panel
- ML-015: Category Management tab

### Sprint 4: Front-Office Integration + Polish
- ML-016: Media Picker component
- ML-017: WhatsApp media upload integration
- ML-018: Expired asset handling
- ML-019: Usage tracking
