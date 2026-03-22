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

## Table 5: TWC_CUSTOMER_MEDIA

Media exchanged between staff and customers (inbound, outbound customised).

### Table Configuration

```json
{
  "TableName": "TWC_CUSTOMER_MEDIA",
  "BillingMode": "PAY_PER_REQUEST",
  "KeySchema": [
    { "AttributeName": "tenantId", "KeyType": "HASH" },
    { "AttributeName": "customerIdMediaId", "KeyType": "RANGE" }
  ],
  "AttributeDefinitions": [
    { "AttributeName": "tenantId", "AttributeType": "S" },
    { "AttributeName": "customerIdMediaId", "AttributeType": "S" },
    { "AttributeName": "customerIdCreatedAt", "AttributeType": "S" },
    { "AttributeName": "messageId", "AttributeType": "S" }
  ],
  "GlobalSecondaryIndexes": [
    {
      "IndexName": "tenantId-customer-index",
      "KeySchema": [
        { "AttributeName": "tenantId", "KeyType": "HASH" },
        { "AttributeName": "customerIdCreatedAt", "KeyType": "RANGE" }
      ],
      "Projection": { "ProjectionType": "ALL" }
    },
    {
      "IndexName": "tenantId-message-index",
      "KeySchema": [
        { "AttributeName": "tenantId", "KeyType": "HASH" },
        { "AttributeName": "messageId", "KeyType": "RANGE" }
      ],
      "Projection": { "ProjectionType": "KEYS_ONLY" }
    }
  ]
}
```

### AWS CLI

```bash
aws dynamodb create-table \
  --table-name TWC_CUSTOMER_MEDIA \
  --billing-mode PAY_PER_REQUEST \
  --key-schema \
    AttributeName=tenantId,KeyType=HASH \
    AttributeName=customerIdMediaId,KeyType=RANGE \
  --attribute-definitions \
    AttributeName=tenantId,AttributeType=S \
    AttributeName=customerIdMediaId,AttributeType=S \
    AttributeName=customerIdCreatedAt,AttributeType=S \
    AttributeName=messageId,AttributeType=S \
  --global-secondary-indexes \
    '[
      {
        "IndexName": "tenantId-customer-index",
        "KeySchema": [
          {"AttributeName": "tenantId", "KeyType": "HASH"},
          {"AttributeName": "customerIdCreatedAt", "KeyType": "RANGE"}
        ],
        "Projection": {"ProjectionType": "ALL"}
      },
      {
        "IndexName": "tenantId-message-index",
        "KeySchema": [
          {"AttributeName": "tenantId", "KeyType": "HASH"},
          {"AttributeName": "messageId", "KeyType": "RANGE"}
        ],
        "Projection": {"ProjectionType": "KEYS_ONLY"}
      }
    ]'
```

### Example Item (Inbound from Customer)

```json
{
  "tenantId": { "S": "camillaandmarc-au" },
  "customerIdMediaId": { "S": "cust_sarah_chen#01HQMEDIA123" },
  "customerId": { "S": "cust_sarah_chen" },
  "mediaId": { "S": "01HQMEDIA123" },
  "customerIdCreatedAt": { "S": "cust_sarah_chen#2025-03-15T10:30:00Z" },
  "direction": { "S": "inbound" },
  "source": { "S": "customer" },
  "s3Key": { "S": "customers/camillaandmarc-au/cust_sarah_chen/inbound/01HQMEDIA123.pdf" },
  "fileName": { "S": "signed_waiver.pdf" },
  "mimeType": { "S": "application/pdf" },
  "fileSize": { "N": "125000" },
  "channel": { "S": "whatsapp" },
  "messageId": { "S": "msg_456" },
  "conversationId": { "S": "conv_789" },
  "documentType": { "S": "waiver" },
  "tags": { "L": [
    { "S": "signed" },
    { "S": "treatment" }
  ]},
  "notes": { "S": "Customer signed in-store on Mar 15. Verified ID." },
  "receivedAt": { "S": "2025-03-15T10:30:00Z" },
  "createdAt": { "S": "2025-03-15T10:30:00Z" }
}
```

### Example Item (Outbound from Library - Reference Only)

```json
{
  "tenantId": { "S": "camillaandmarc-au" },
  "customerIdMediaId": { "S": "cust_sarah_chen#01HQMEDIA456" },
  "customerId": { "S": "cust_sarah_chen" },
  "mediaId": { "S": "01HQMEDIA456" },
  "customerIdCreatedAt": { "S": "cust_sarah_chen#2025-03-10T09:00:00Z" },
  "direction": { "S": "outbound" },
  "source": { "S": "library" },
  "mediaLibraryAssetId": { "S": "01HQXYZ123ABC" },
  "fileName": { "S": "Waiver_Form.pdf" },
  "mimeType": { "S": "application/pdf" },
  "fileSize": { "N": "450000" },
  "channel": { "S": "whatsapp" },
  "messageId": { "S": "msg_123" },
  "conversationId": { "S": "conv_789" },
  "documentType": { "S": "waiver" },
  "sentBy": { "S": "user_jane_smith" },
  "createdAt": { "S": "2025-03-10T09:00:00Z" }
}
```

### Example Item (Outbound Customised - Stored)

```json
{
  "tenantId": { "S": "camillaandmarc-au" },
  "customerIdMediaId": { "S": "cust_sarah_chen#01HQMEDIA789" },
  "customerId": { "S": "cust_sarah_chen" },
  "mediaId": { "S": "01HQMEDIA789" },
  "customerIdCreatedAt": { "S": "cust_sarah_chen#2025-03-08T14:00:00Z" },
  "direction": { "S": "outbound" },
  "source": { "S": "staff_upload" },
  "s3Key": { "S": "customers/camillaandmarc-au/cust_sarah_chen/outbound/01HQMEDIA789.pdf" },
  "fileName": { "S": "Sarah_Quote_March.pdf" },
  "mimeType": { "S": "application/pdf" },
  "fileSize": { "N": "890000" },
  "channel": { "S": "email" },
  "messageId": { "S": "msg_111" },
  "documentType": { "S": "quote" },
  "tags": { "L": [
    { "S": "quote-v2" },
    { "S": "march-2025" }
  ]},
  "notes": { "S": "Updated quote with 10% loyalty discount applied" },
  "sentBy": { "S": "user_jane_smith" },
  "createdAt": { "S": "2025-03-08T14:00:00Z" }
}
```

### Document Types (Predefined Values)

| Value | Description |
|-------|-------------|
| `waiver` | Waiver forms (sent or signed) |
| `quote` | Price quotes, proposals |
| `lookbook` | Personalised lookbooks |
| `id` | ID documents, verification |
| `receipt` | Receipts, invoices |
| `other` | Uncategorised |

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
| Get customer media by ID | TWC_CUSTOMER_MEDIA | `tenantId = :t AND customerIdMediaId = :cm` |
| List customer media | tenantId-customer-index | `tenantId = :t AND begins_with(customerIdCreatedAt, :cust)` |
| Find media by message | tenantId-message-index | `tenantId = :t AND messageId = :m` |

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

resource "aws_dynamodb_table" "customer_media" {
  name         = "TWC_CUSTOMER_MEDIA"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "tenantId"
  range_key    = "customerIdMediaId"

  attribute {
    name = "tenantId"
    type = "S"
  }

  attribute {
    name = "customerIdMediaId"
    type = "S"
  }

  attribute {
    name = "customerIdCreatedAt"
    type = "S"
  }

  attribute {
    name = "messageId"
    type = "S"
  }

  global_secondary_index {
    name            = "tenantId-customer-index"
    hash_key        = "tenantId"
    range_key       = "customerIdCreatedAt"
    projection_type = "ALL"
  }

  global_secondary_index {
    name            = "tenantId-message-index"
    hash_key        = "tenantId"
    range_key       = "messageId"
    projection_type = "KEYS_ONLY"
  }

  tags = {
    Service = "twc-medialib"
  }
}
```
