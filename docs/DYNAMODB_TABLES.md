# DynamoDB Table Definitions

## Table 1: TWC_MEDIA_CATEGORY

User-configurable categories per tenant.

### Table Configuration

```json
{
  "TableName": "TWC_MEDIA_CATEGORY",
  "BillingMode": "PAY_PER_REQUEST",
  "KeySchema": [
    { "AttributeName": "tenantId", "KeyType": "HASH" },
    { "AttributeName": "categoryId", "KeyType": "RANGE" }
  ],
  "AttributeDefinitions": [
    { "AttributeName": "tenantId", "AttributeType": "S" },
    { "AttributeName": "categoryId", "AttributeType": "S" }
  ]
}
```

### AWS CLI

```bash
aws dynamodb create-table \
  --table-name TWC_MEDIA_CATEGORY \
  --billing-mode PAY_PER_REQUEST \
  --key-schema \
    AttributeName=tenantId,KeyType=HASH \
    AttributeName=categoryId,KeyType=RANGE \
  --attribute-definitions \
    AttributeName=tenantId,AttributeType=S \
    AttributeName=categoryId,AttributeType=S
```

### Example Item

```json
{
  "tenantId": { "S": "camillaandmarc-au" },
  "categoryId": { "S": "vip-event" },
  "name": { "S": "VIP Event" },
  "description": { "S": "Assets for VIP customer events" },
  "sortOrder": { "N": "10" },
  "isDefault": { "BOOL": false },
  "createdAt": { "S": "2025-03-01T10:00:00Z" },
  "updatedAt": { "S": "2025-03-01T10:00:00Z" }
}
```

### Default Categories (seed on tenant creation)

```json
[
  { "categoryId": "collection", "name": "Collection", "sortOrder": 1, "isDefault": true },
  { "categoryId": "invitation", "name": "Invitation", "sortOrder": 2, "isDefault": true },
  { "categoryId": "promotion", "name": "Promotion", "sortOrder": 3, "isDefault": true },
  { "categoryId": "product", "name": "Product", "sortOrder": 4, "isDefault": true },
  { "categoryId": "brand", "name": "Brand", "sortOrder": 5, "isDefault": true },
  { "categoryId": "other", "name": "Other", "sortOrder": 99, "isDefault": true }
]
```

---

## Table 2: TWC_MEDIA_ASSET

Main table for media asset records.

### Table Configuration

```json
{
  "TableName": "TWC_MEDIA_ASSET",
  "BillingMode": "PAY_PER_REQUEST",
  "KeySchema": [
    { "AttributeName": "tenantId", "KeyType": "HASH" },
    { "AttributeName": "assetId", "KeyType": "RANGE" }
  ],
  "AttributeDefinitions": [
    { "AttributeName": "tenantId", "AttributeType": "S" },
    { "AttributeName": "assetId", "AttributeType": "S" },
    { "AttributeName": "categoryCreatedAt", "AttributeType": "S" },
    { "AttributeName": "statusCreatedAt", "AttributeType": "S" }
  ],
  "GlobalSecondaryIndexes": [
    {
      "IndexName": "tenantId-category-index",
      "KeySchema": [
        { "AttributeName": "tenantId", "KeyType": "HASH" },
        { "AttributeName": "categoryCreatedAt", "KeyType": "RANGE" }
      ],
      "Projection": { "ProjectionType": "ALL" }
    },
    {
      "IndexName": "tenantId-status-index",
      "KeySchema": [
        { "AttributeName": "tenantId", "KeyType": "HASH" },
        { "AttributeName": "statusCreatedAt", "KeyType": "RANGE" }
      ],
      "Projection": { "ProjectionType": "ALL" }
    }
  ]
}
```

### AWS CLI

```bash
aws dynamodb create-table \
  --table-name TWC_MEDIA_ASSET \
  --billing-mode PAY_PER_REQUEST \
  --key-schema \
    AttributeName=tenantId,KeyType=HASH \
    AttributeName=assetId,KeyType=RANGE \
  --attribute-definitions \
    AttributeName=tenantId,AttributeType=S \
    AttributeName=assetId,AttributeType=S \
    AttributeName=categoryCreatedAt,AttributeType=S \
    AttributeName=statusCreatedAt,AttributeType=S \
  --global-secondary-indexes \
    '[
      {
        "IndexName": "tenantId-category-index",
        "KeySchema": [
          {"AttributeName": "tenantId", "KeyType": "HASH"},
          {"AttributeName": "categoryCreatedAt", "KeyType": "RANGE"}
        ],
        "Projection": {"ProjectionType": "ALL"}
      },
      {
        "IndexName": "tenantId-status-index",
        "KeySchema": [
          {"AttributeName": "tenantId", "KeyType": "HASH"},
          {"AttributeName": "statusCreatedAt", "KeyType": "RANGE"}
        ],
        "Projection": {"ProjectionType": "ALL"}
      }
    ]'
```

### Example Item

```json
{
  "tenantId": { "S": "camillaandmarc-au" },
  "assetId": { "S": "01HQXYZ123ABC" },
  "title": { "S": "Summer 2025 Collection" },
  "description": { "S": "New arrivals lookbook for Summer 2025" },
  "category": { "S": "collection" },
  "categoryCreatedAt": { "S": "collection#2025-03-01T10:00:00Z" },
  "tags": { "L": [
    { "S": "summer" },
    { "S": "2025" },
    { "S": "lookbook" }
  ]},
  "originalFileName": { "S": "Summer_2025_Lookbook.pdf" },
  "mimeType": { "S": "application/pdf" },
  "fileSize": { "N": "2500000" },
  "status": { "S": "active" },
  "statusCreatedAt": { "S": "active#2025-03-01T10:00:00Z" },
  "expiresAt": { "S": "2025-06-01T00:00:00Z" },
  "currentVersion": { "N": "2" },
  "variants": { "M": {
    "original": { "M": {
      "s3Key": { "S": "media/camillaandmarc-au/01HQXYZ123ABC/v2/original.pdf" },
      "fileName": { "S": "Summer_2025_Collection.pdf" },
      "fileSize": { "N": "8500000" },
      "mimeType": { "S": "application/pdf" }
    }},
    "sms": { "M": {
      "s3Key": { "S": "media/camillaandmarc-au/01HQXYZ123ABC/v2/sms.jpg" },
      "fileName": { "S": "Summer_2025_SMS.jpg" },
      "fileSize": { "N": "850000" },
      "mimeType": { "S": "image/jpeg" }
    }},
    "email": { "M": {
      "s3Key": { "S": "media/camillaandmarc-au/01HQXYZ123ABC/v2/email.pdf" },
      "fileName": { "S": "Summer_2025_Email.pdf" },
      "fileSize": { "N": "2500000" },
      "mimeType": { "S": "application/pdf" }
    }}
  }},
  "createdAt": { "S": "2025-03-01T10:00:00Z" },
  "createdBy": { "S": "user_jane_smith" },
  "updatedAt": { "S": "2025-03-05T14:30:00Z" },
  "updatedBy": { "S": "user_john_doe" }
}
```

---

## Table 3: TWC_MEDIA_VERSION

Version history for assets.

### Table Configuration

```json
{
  "TableName": "TWC_MEDIA_VERSION",
  "BillingMode": "PAY_PER_REQUEST",
  "KeySchema": [
    { "AttributeName": "tenantIdAssetId", "KeyType": "HASH" },
    { "AttributeName": "version", "KeyType": "RANGE" }
  ],
  "AttributeDefinitions": [
    { "AttributeName": "tenantIdAssetId", "AttributeType": "S" },
    { "AttributeName": "version", "AttributeType": "N" }
  ]
}
```

### AWS CLI

```bash
aws dynamodb create-table \
  --table-name TWC_MEDIA_VERSION \
  --billing-mode PAY_PER_REQUEST \
  --key-schema \
    AttributeName=tenantIdAssetId,KeyType=HASH \
    AttributeName=version,KeyType=RANGE \
  --attribute-definitions \
    AttributeName=tenantIdAssetId,AttributeType=S \
    AttributeName=version,AttributeType=N
```

### Example Item

```json
{
  "tenantIdAssetId": { "S": "camillaandmarc-au#01HQXYZ123ABC" },
  "version": { "N": "2" },
  "variants": { "M": {
    "original": { "M": {
      "s3Key": { "S": "media/camillaandmarc-au/01HQXYZ123ABC/v2/original.pdf" },
      "fileName": { "S": "Summer_2025_Collection_v2.pdf" },
      "fileSize": { "N": "8500000" },
      "mimeType": { "S": "application/pdf" }
    }},
    "sms": { "M": {
      "s3Key": { "S": "media/camillaandmarc-au/01HQXYZ123ABC/v2/sms.jpg" },
      "fileName": { "S": "Summer_2025_SMS_v2.jpg" },
      "fileSize": { "N": "850000" },
      "mimeType": { "S": "image/jpeg" }
    }},
    "email": { "M": {
      "s3Key": { "S": "media/camillaandmarc-au/01HQXYZ123ABC/v2/email.pdf" },
      "fileName": { "S": "Summer_2025_Email_v2.pdf" },
      "fileSize": { "N": "2500000" },
      "mimeType": { "S": "application/pdf" }
    }}
  }},
  "changelog": { "S": "Updated cover image" },
  "createdAt": { "S": "2025-03-05T14:30:00Z" },
  "createdBy": { "S": "user_john_doe" }
}
```

---

## Table 4: TWC_MEDIA_UPLOAD (Optional)

Tracks pending uploads before they become assets. TTL cleans up abandoned uploads.

### Table Configuration

```json
{
  "TableName": "TWC_MEDIA_UPLOAD",
  "BillingMode": "PAY_PER_REQUEST",
  "KeySchema": [
    { "AttributeName": "uploadId", "KeyType": "HASH" }
  ],
  "AttributeDefinitions": [
    { "AttributeName": "uploadId", "AttributeType": "S" }
  ],
  "TimeToLiveSpecification": {
    "AttributeName": "ttl",
    "Enabled": true
  }
}
```

### Example Item

```json
{
  "uploadId": { "S": "upload_abc123" },
  "tenantId": { "S": "camillaandmarc-au" },
  "userId": { "S": "user_jane_smith" },
  "fileName": { "S": "Summer_2025_Lookbook.pdf" },
  "mimeType": { "S": "application/pdf" },
  "fileSize": { "N": "2500000" },
  "s3Key": { "S": "uploads/camillaandmarc-au/upload_abc123/Summer_2025_Lookbook.pdf" },
  "status": { "S": "pending" },
  "createdAt": { "S": "2025-03-01T10:00:00Z" },
  "ttl": { "N": "1709380800" }
}
```

---

## Access Patterns

| Pattern | Table | Key Condition |
|---------|-------|---------------|
| Get asset by ID | TWC_MEDIA_ASSET | `tenantId = :t AND assetId = :a` |
| List all assets | TWC_MEDIA_ASSET | `tenantId = :t` |
| List by category | tenantId-category-index | `tenantId = :t AND begins_with(categoryCreatedAt, :cat)` |
| List by status | tenantId-status-index | `tenantId = :t AND begins_with(statusCreatedAt, :status)` |
| Get version history | TWC_MEDIA_VERSION | `tenantIdAssetId = :ta` (scan reverse for newest first) |
| Get specific version | TWC_MEDIA_VERSION | `tenantIdAssetId = :ta AND version = :v` |
| Get pending upload | TWC_MEDIA_UPLOAD | `uploadId = :u` |

---

## Terraform

```hcl
resource "aws_dynamodb_table" "media_category" {
  name         = "TWC_MEDIA_CATEGORY"
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

  tags = {
    Service = "twc-medialib"
  }
}

resource "aws_dynamodb_table" "media_asset" {
  name         = "TWC_MEDIA_ASSET"
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
    name = "categoryCreatedAt"
    type = "S"
  }

  attribute {
    name = "statusCreatedAt"
    type = "S"
  }

  global_secondary_index {
    name            = "tenantId-category-index"
    hash_key        = "tenantId"
    range_key       = "categoryCreatedAt"
    projection_type = "ALL"
  }

  global_secondary_index {
    name            = "tenantId-status-index"
    hash_key        = "tenantId"
    range_key       = "statusCreatedAt"
    projection_type = "ALL"
  }

  tags = {
    Service = "twc-medialib"
  }
}

resource "aws_dynamodb_table" "media_version" {
  name         = "TWC_MEDIA_VERSION"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "tenantIdAssetId"
  range_key    = "version"

  attribute {
    name = "tenantIdAssetId"
    type = "S"
  }

  attribute {
    name = "version"
    type = "N"
  }

  tags = {
    Service = "twc-medialib"
  }
}

resource "aws_dynamodb_table" "media_upload" {
  name         = "TWC_MEDIA_UPLOAD"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "uploadId"

  attribute {
    name = "uploadId"
    type = "S"
  }

  ttl {
    attribute_name = "ttl"
    enabled        = true
  }

  tags = {
    Service = "twc-medialib"
  }
}
```
