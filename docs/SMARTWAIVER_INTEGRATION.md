# SmartWaiver Integration - Design Addendum

**Status:** Draft
**Last Updated:** March 2025
**Parent Document:** DESIGN.md

---

## Overview

Integration with [SmartWaiver](https://www.smartwaiver.com) to enable digital consent forms (e.g., ear piercing waivers, treatment consent) linked to customer profiles.

### Key Capabilities

| Feature | Description |
|---------|-------------|
| **Send waiver** | Generate prefilled waiver link, send via SMS/email/WhatsApp |
| **Track status** | Know if waiver is pending, sent, or signed |
| **Capture evidence** | Fetch signed PDF, store in customer media |
| **View history** | Staff sees waiver status and signed documents in customer profile |

---

## Integration Flow

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Staff Action   │     │   TWC Backend   │     │   SmartWaiver   │
│  "Send Waiver"  │────▶│                 │────▶│   API           │
└─────────────────┘     │  1. Prefill     │     │                 │
                        │     customer    │     │  Returns:       │
                        │     data        │     │  Waiver URL     │
                        └────────┬────────┘     └─────────────────┘
                                 │
                                 ▼
                        ┌─────────────────┐
                        │  Send to        │
                        │  Customer       │
                        │  (SMS/Email)    │
                        └────────┬────────┘
                                 │
                                 ▼
┌─────────────────┐     ┌─────────────────┐
│  Customer       │     │  SmartWaiver    │
│  Signs Waiver   │────▶│  Hosted Form    │
└─────────────────┘     └────────┬────────┘
                                 │
                                 ▼
                        ┌─────────────────┐     ┌─────────────────┐
                        │  SmartWaiver    │     │   TWC Backend   │
                        │  Webhook        │────▶│                 │
                        │  (waiver.sign)  │     │  2. Fetch PDF   │
                        └─────────────────┘     │  3. Store in    │
                                                │     customer    │
                                                │     media       │
                                                │  4. Update      │
                                                │     profile     │
                                                └─────────────────┘
```

---

## Data Model Changes

### Customer Profile Extension

Add waiver tracking fields to customer profile (DynamoDB or existing customer table):

| Field | Type | Description |
|-------|------|-------------|
| `waiverStatus` | String | `none`, `sent`, `signed`, `expired` |
| `waiverTemplateId` | String | SmartWaiver template ID used |
| `waiverSentAt` | String (ISO) | When waiver link was sent |
| `waiverSignedAt` | String (ISO) | When customer signed |
| `waiverExpiresAt` | String (ISO) | When waiver consent expires (optional) |
| `waiverMediaId` | String | Link to `TWC_CUSTOMER_MEDIA` record |
| `smartwaiverWaiverId` | String | SmartWaiver's waiver ID (for API lookups) |

**Note:** If a customer requires multiple waivers (e.g., different treatments), consider a `TWC_CUSTOMER_WAIVER` table instead of profile fields. See [Multiple Waivers](#multiple-waivers-per-customer) section.

### TWC_CUSTOMER_MEDIA Updates

Add new source type for SmartWaiver-originated documents:

**Source Enum Update:**

| Value | Description |
|-------|-------------|
| `customer` | Customer sent directly (existing) |
| `library` | From media library (existing) |
| `staff_upload` | Staff uploaded (existing) |
| `smartwaiver` | **NEW** - Fetched from SmartWaiver API |

**Additional Fields for SmartWaiver Media:**

| Field | Type | Description |
|-------|------|-------------|
| `smartwaiverWaiverId` | String | SmartWaiver waiver ID |
| `smartwaiverTemplateId` | String | Template used |
| `waiverSignedAt` | String (ISO) | Signature timestamp from SmartWaiver |
| `participantName` | String | Name as signed on waiver |

---

## SmartWaiver Configuration

### Tenant Settings

Store SmartWaiver credentials per tenant:

```
Table: TWC_TENANT_INTEGRATIONS (or existing settings table)

{
  "tenantId": "retailer-xyz",
  "integrations": {
    "smartwaiver": {
      "enabled": true,
      "apiKey": "sw_live_xxxx...",  // Encrypted
      "defaultTemplateId": "tpl_abc123",
      "webhookSecret": "whsec_xxx...",  // For validating webhooks
      "templates": [
        {
          "templateId": "tpl_abc123",
          "name": "Ear Piercing Consent",
          "useCase": "piercing"
        },
        {
          "templateId": "tpl_def456",
          "name": "Treatment Waiver",
          "useCase": "treatment"
        }
      ]
    }
  }
}
```

### SmartWaiver Account Setup

Required configuration in SmartWaiver dashboard:

1. **Create API Key:** Settings > API > Create Key
2. **Configure Webhook:** Settings > Webhooks > Add endpoint
   - URL: `https://api.twc.com/webhooks/smartwaiver`
   - Events: `waiver.signed`
3. **Template Setup:** Add custom field for TWC customer ID (auto-tag)

---

## API Design

### 1. Send Waiver to Customer

Generate a prefilled waiver link and optionally send via outreach channel.

```http
POST /api/v1/customers/{customerId}/waiver/send
```

**Request:**
```json
{
  "templateId": "tpl_abc123",
  "channel": "sms",
  "prefillData": {
    "firstName": "Sarah",
    "lastName": "Chen",
    "email": "sarah@email.com",
    "phone": "+61412345678"
  },
  "expiresInDays": 7,
  "messageTemplate": "Hi {{firstName}}, please complete your consent form: {{waiverUrl}}"
}
```

**Response:**
```json
{
  "success": true,
  "waiverUrl": "https://waiver.smartwaiver.com/w/abc123xyz/web/?auto_tag=cust_12345",
  "sentVia": "sms",
  "expiresAt": "2025-04-03T00:00:00Z"
}
```

**Flow:**
1. Call SmartWaiver API: `POST /v4/templates/{templateId}/prefill`
2. SmartWaiver returns unique waiver URL with customer data prefilled
3. URL includes `auto_tag` parameter with TWC customer ID
4. Send message via outreach service (SMS/email/WhatsApp)
5. Update customer profile: `waiverStatus: 'sent'`, `waiverSentAt: now()`

### 2. Get Waiver Status

```http
GET /api/v1/customers/{customerId}/waiver/status
```

**Response:**
```json
{
  "status": "signed",
  "templateName": "Ear Piercing Consent",
  "sentAt": "2025-03-20T10:00:00Z",
  "signedAt": "2025-03-20T14:30:00Z",
  "expiresAt": "2025-03-20T00:00:00Z",
  "mediaId": "01HQWAIVER123",
  "signedPdfUrl": "https://presigned-url..."
}
```

### 3. Resend Waiver

```http
POST /api/v1/customers/{customerId}/waiver/resend
```

**Request:**
```json
{
  "channel": "whatsapp"
}
```

Resends the existing waiver link (if not expired) via a different channel.

### 4. List Available Waiver Templates

```http
GET /api/v1/waiver/templates
```

**Response:**
```json
{
  "templates": [
    {
      "templateId": "tpl_abc123",
      "name": "Ear Piercing Consent",
      "useCase": "piercing"
    },
    {
      "templateId": "tpl_def456",
      "name": "Treatment Waiver",
      "useCase": "treatment"
    }
  ]
}
```

---

## Webhook Handler

### Endpoint

```http
POST /webhooks/smartwaiver
```

### SmartWaiver Webhook Payload

When a waiver is signed, SmartWaiver sends:

```json
{
  "event": "waiver.signed",
  "waiver_id": "5f8b3c2a1d4e5f6a7b8c9d0e",
  "template_id": "tpl_abc123",
  "timestamp": "2025-03-20T14:30:00Z",
  "auto_tag": "cust_12345"
}
```

### Webhook Processing Flow

```python
async def handle_smartwaiver_webhook(payload):
    # 1. Validate webhook signature
    validate_signature(payload, webhook_secret)

    # 2. Extract customer ID from auto_tag
    customer_id = payload["auto_tag"]
    waiver_id = payload["waiver_id"]

    # 3. Fetch signed waiver PDF from SmartWaiver
    pdf_data = await smartwaiver_client.get_waiver_pdf(waiver_id)

    # 4. Fetch waiver details (participant info, signed date)
    waiver_details = await smartwaiver_client.get_waiver(waiver_id)

    # 5. Store PDF in S3 via customer media
    media_record = await customer_media_service.capture({
        "customerId": customer_id,
        "direction": "inbound",
        "source": "smartwaiver",
        "documentType": "waiver",
        "fileName": f"waiver_{waiver_id}.pdf",
        "mimeType": "application/pdf",
        "content": pdf_data,
        "metadata": {
            "smartwaiverWaiverId": waiver_id,
            "smartwaiverTemplateId": payload["template_id"],
            "waiverSignedAt": payload["timestamp"],
            "participantName": waiver_details["participant_name"]
        }
    })

    # 6. Update customer profile
    await customer_service.update(customer_id, {
        "waiverStatus": "signed",
        "waiverSignedAt": payload["timestamp"],
        "waiverMediaId": media_record["mediaId"],
        "smartwaiverWaiverId": waiver_id
    })

    # 7. Optional: Notify staff
    await notify_staff(customer_id, "Waiver signed")
```

---

## SmartWaiver API Client

### Required API Calls

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `POST` | `/v4/templates/{id}/prefill` | Generate prefilled waiver URL |
| `GET` | `/v4/waivers/{id}` | Get waiver details |
| `GET` | `/v4/waivers/{id}?pdf=true` | Get signed PDF (Base64) |
| `GET` | `/v4/templates` | List available templates |

### Client Implementation

```python
class SmartWaiverClient:
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.smartwaiver.com/v4"

    async def create_prefilled_waiver(
        self,
        template_id: str,
        customer_id: str,
        participant: dict,
        expiration_days: int = 7
    ) -> str:
        """Generate a prefilled waiver URL."""
        response = await self._post(
            f"/templates/{template_id}/prefill",
            json={
                "adult": True,
                "participants": [{
                    "firstName": participant.get("firstName"),
                    "lastName": participant.get("lastName"),
                    "email": participant.get("email"),
                    "phone": participant.get("phone")
                }],
                "autoTag": customer_id,
                "expirationInSeconds": expiration_days * 86400
            }
        )
        return response["prefillUrl"]

    async def get_waiver(self, waiver_id: str) -> dict:
        """Get waiver details including participant info."""
        return await self._get(f"/waivers/{waiver_id}")

    async def get_waiver_pdf(self, waiver_id: str) -> bytes:
        """Get signed waiver as PDF bytes."""
        response = await self._get(f"/waivers/{waiver_id}?pdf=true")
        pdf_base64 = response["pdf"]
        return base64.b64decode(pdf_base64)

    async def list_templates(self) -> list:
        """List all waiver templates."""
        response = await self._get("/templates")
        return response["templates"]
```

---

## UI Design

### Customer Profile - Waiver Status

Add waiver status indicator to customer profile header or dedicated section:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Sarah Chen                                              VIP Customer   │
│  sarah.chen@email.com | +61 412 345 678                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Waiver Status                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  ✅ Signed                                                       │   │
│  │  Ear Piercing Consent • Signed Mar 20, 2025                     │   │
│  │  [View Document]  [Send New Waiver]                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  [Orders]  [Wishlist]  [Notes]  [Messages]  [Media]                    │
└─────────────────────────────────────────────────────────────────────────┘
```

**Status Variants:**

| Status | Display |
|--------|---------|
| `none` | "No waiver required" or "No waiver on file" with [Send Waiver] button |
| `sent` | "⏳ Waiver Pending - Sent Mar 20, 2025" with [Resend] [Cancel] |
| `signed` | "✅ Signed - Mar 20, 2025" with [View Document] [Send New Waiver] |
| `expired` | "⚠️ Expired - Was signed Mar 20, 2024" with [Send New Waiver] |

### Send Waiver Modal

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Send Waiver to Sarah Chen                                         [×] │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Waiver Type *                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Ear Piercing Consent                                         ▼  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Send Via *                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  (•) SMS to +61 412 345 678                                     │   │
│  │  ( ) WhatsApp to +61 412 345 678                                │   │
│  │  ( ) Email to sarah.chen@email.com                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Message Preview                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Hi Sarah, please complete your consent form before your         │   │
│  │ appointment: [waiver link]                                       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Link expires in: [7 days ▼]                                           │
│                                                                         │
│                                              [Cancel]  [Send Waiver]   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Customer Media Tab - Waiver Documents

Waiver documents appear in the existing Media tab with `documentType: waiver`:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  [Orders]  [Wishlist]  [Notes]  [Messages]  [Media]                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  [📥 Received]  [📤 Sent]  [All]              [+ Upload]  🔍 Search    │
│                                                                         │
│  Filter: [Waiver ▼]  [All Channels ▼]                                  │
│                                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                              │
│  │  📄      │  │  📄      │  │  📄      │                              │
│  │          │  │          │  │          │                              │
│  │ Ear      │  │ Treatment│  │ Ear      │                              │
│  │ Piercing │  │ Waiver   │  │ Piercing │                              │
│  │ Consent  │  │          │  │ Consent  │                              │
│  │          │  │          │  │          │                              │
│  │ Mar 20   │  │ Jan 15   │  │ Mar 20   │                              │
│  │ 2025     │  │ 2025     │  │ 2024     │                              │
│  │ ✅ Signed│  │ ✅ Signed│  │ ⚠️ Expired│                              │
│  └──────────┘  └──────────┘  └──────────┘                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Multiple Waivers Per Customer

If customers may need multiple active waivers (different treatments, annual renewals), use a dedicated table:

### TWC_CUSTOMER_WAIVER Table

```
Table: TWC_CUSTOMER_WAIVER

Partition Key: tenantId (String)
Sort Key: customerId#waiverId (String)

GSI1: tenantId-customerId-index
  - PK: tenantId
  - SK: customerId#status#createdAt
```

| Field | Type | Description |
|-------|------|-------------|
| `tenantId` | String | Retailer ID |
| `customerId` | String | Customer ID |
| `waiverId` | String | ULID |
| `templateId` | String | SmartWaiver template ID |
| `templateName` | String | Template name (denormalized) |
| `useCase` | String | `piercing`, `treatment`, `liability`, etc. |
| `status` | String | `sent`, `signed`, `expired`, `cancelled` |
| `sentAt` | String (ISO) | When link was sent |
| `sentVia` | String | `sms`, `email`, `whatsapp` |
| `signedAt` | String (ISO) | When signed |
| `expiresAt` | String (ISO) | Consent expiry |
| `mediaId` | String | Link to `TWC_CUSTOMER_MEDIA` |
| `smartwaiverWaiverId` | String | SmartWaiver ID |
| `smartwaiverUrl` | String | Original waiver URL |
| `sentBy` | String | Staff ID who sent |

This allows:
- Multiple waivers per customer
- Different waiver types
- Independent expiry tracking
- Full audit history

---

## Error Handling

| Scenario | Handling |
|----------|----------|
| SmartWaiver API down | Queue waiver request, retry with backoff |
| Webhook delivery failure | SmartWaiver retries; implement idempotency |
| PDF fetch fails | Retry 3x, then mark as "signed but PDF pending" |
| Customer ID not found in auto_tag | Log error, manual reconciliation queue |
| Duplicate webhook | Check if waiver already processed (idempotent) |

---

## Security Considerations

| Concern | Mitigation |
|---------|------------|
| API key storage | Encrypt at rest, use AWS Secrets Manager |
| Webhook validation | Verify SmartWaiver signature header |
| PDF access | Presigned URLs with short expiry (1 hour) |
| PII in transit | HTTPS only, no logging of customer data |
| Waiver tampering | Store SmartWaiver waiver ID for verification |

---

## Implementation Phases

### Phase 1: Core Integration
- [ ] SmartWaiver API client
- [ ] Send waiver endpoint
- [ ] Webhook handler
- [ ] PDF storage to customer media
- [ ] Basic UI (send waiver button, status display)

### Phase 2: Enhanced UX
- [ ] Customer profile waiver section
- [ ] Waiver history view
- [ ] Resend/cancel functionality
- [ ] Staff notifications on signature

### Phase 3: Advanced
- [ ] Multiple waivers per customer
- [ ] Waiver expiry reminders
- [ ] Appointment integration (require waiver before booking)
- [ ] Analytics (signature rate, time to sign)

---

## Testing

### Test Scenarios

1. **Send waiver** - Verify prefilled URL generated, message sent, status updated
2. **Webhook received** - Verify PDF fetched, stored, profile updated
3. **Duplicate webhook** - Verify idempotent (no duplicate records)
4. **Expired waiver** - Verify status shows expired, can send new
5. **View signed PDF** - Verify presigned URL works, document displays

### SmartWaiver Test Mode

Use SmartWaiver's test/sandbox environment for development:
- Test API key prefix: `sw_test_`
- Test waivers don't count against quota
- Webhooks can be manually triggered

---

## Appendix: SmartWaiver API Reference

**Base URL:** `https://api.smartwaiver.com/v4`

**Authentication:** Bearer token
```
Authorization: Bearer {api_key}
```

**Rate Limits:**
- Standard: 100 requests/minute
- Search: 5 requests/minute

**Key Endpoints:**

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/templates` | GET | List all templates |
| `/templates/{id}` | GET | Get template details |
| `/templates/{id}/prefill` | POST | Create prefilled waiver URL |
| `/waivers/{id}` | GET | Get waiver details |
| `/waivers/{id}?pdf=true` | GET | Get waiver with PDF (Base64) |
| `/search` | POST | Search signed waivers |
| `/webhooks/configure` | PUT | Configure webhook endpoint |

**Documentation:** https://api.smartwaiver.com/docs/v4/
