# ShhhType Account, Trial & Subscription Flow

## System Architecture

```
                         +-------------------+
                         |   Landing Page    |
                         |  (shhhtype.com)   |
                         +--------+----------+
                                  |
                          POST /api/signup
                                  |
                                  v
                    +----------------------------+
                    |     Keygen API Server       |
                    | (Coolify on Hostinger)      |
                    |                            |
                    |  - User accounts           |
                    |  - License keys            |
                    |  - Machine activations     |
                    |  - Validation endpoint     |
                    +-----+----------+-----------+
                          |          |
                 Webhooks |          | Webhooks
                          v          v
              +-----------+--+  +---+-----------+
              |  Zoho Flow   |  |  Zoho Flow    |
              |  (CRM sync)  |  | (Billing sync)|
              +------+-------+  +-------+-------+
                     |                  |
                     v                  v
              +------+-------+  +-------+-------+
              |  Zoho CRM    |  | Zoho Billing  |
              |              |  |               |
              | - Contacts   |  | - $19/mo plan |
              | - Pipeline   |  | - $200/yr plan|
              | - Status     |  | - Invoices    |
              +--------------+  +-------+-------+
                                        |
                              (fallback) |
                                        v
                                +-------+--------+
                                | LemonSqueezy   |
                                | (alt payments) |
                                +----------------+

                    +----------------------------+
                    |   ShhhType macOS App       |
                    |   (Tauri / Rust)           |
                    |                            |
                    |  Validates license on      |
                    |  startup via Keygen API    |
                    +----------------------------+
```

---

## 1. Account Creation Flow

### User Journey
1. User clicks **"Start Free Trial"** on landing page
2. Signup modal collects: **Full Name, Email, Password**
3. Form submits to `POST https://api.shhhtype.com/api/signup`
4. Backend creates user in Keygen + generates trial license
5. User receives email with **license key + download link**
6. User installs app, enters license key

### API: POST /api/signup

**Request:**
```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securepassword123"
}
```

**Backend Logic:**
```
1. Validate input (email format, password >= 8 chars)
2. Check if email already exists in Keygen → return error if duplicate
3. Create user in Keygen API
4. Generate license key under TRIAL policy (7-day expiry)
5. Keygen webhook fires → Zoho Flow → Create contact in Zoho CRM
6. Send welcome email via Zoho (or transactional email service)
7. Return license key to frontend
```

**Response (success):**
```json
{
  "success": true,
  "license_key": "SHHH-XXXX-XXXX-XXXX-XXXX",
  "trial_ends_at": "2026-04-03T00:00:00Z",
  "download_url": "https://shhhtype.com/download"
}
```

**Response (error):**
```json
{
  "success": false,
  "message": "An account with this email already exists."
}
```

### Keygen API Call: Create User
```bash
curl -X POST https://api.shhhtype.com/v1/accounts/{ACCOUNT_ID}/users \
  -H "Authorization: Bearer {ADMIN_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "data": {
      "type": "users",
      "attributes": {
        "firstName": "John",
        "lastName": "Doe",
        "email": "john@example.com",
        "password": "securepassword123"
      }
    }
  }'
```

### Keygen API Call: Create Trial License
```bash
curl -X POST https://api.shhhtype.com/v1/accounts/{ACCOUNT_ID}/licenses \
  -H "Authorization: Bearer {ADMIN_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "data": {
      "type": "licenses",
      "attributes": {
        "expiry": "2026-04-03T00:00:00Z"
      },
      "relationships": {
        "policy": {
          "data": { "type": "policies", "id": "{TRIAL_POLICY_ID}" }
        },
        "user": {
          "data": { "type": "users", "id": "{USER_ID}" }
        }
      }
    }
  }'
```

---

## 2. Trial Flow

### Trial Policy (Keygen Configuration)
| Setting | Value |
|---------|-------|
| Name | `ShhhType Trial` |
| Duration | 7 days |
| Max Machines | 1 |
| Require Fingerprint | Yes |
| Auth Strategy | LICENSE |
| Expiration Strategy | RESTRICT_ACCESS |

### Timeline

| Day | Action |
|-----|--------|
| 0 | Account created, trial license issued, welcome email sent |
| 1-4 | Full feature access, app validates license on startup |
| 5 | Reminder email: "Your trial expires in 2 days" |
| 7 | License expires, app shows upgrade prompt |
| 7+ | Zoho CRM contact updated to "Trial Expired" |
| 10 | Follow-up email: "We miss you — 20% off first month?" (optional) |

### App Behavior During Trial
- Full access to all features (cloud transcription, AI rewrite, all languages)
- Status bar shows "Trial: X days remaining"
- Day 6-7: Subtle banner "Trial ending soon — Subscribe to keep access"

### Zoho CRM Contact Lifecycle
```
Trial → Trial Expiring (Day 5) → Trial Expired → Customer (on purchase)
                                              → Churned (no purchase after 30 days)
```

---

## 3. License Validation (macOS App)

### On App Launch
```
1. Read stored license key from local keychain
2. POST to Keygen validation endpoint
3. Check response: status, type, expiry
4. Cache result locally (valid for 24 hours)
5. If offline and cache valid → allow access
6. If expired → show upgrade screen
```

### API: Validate License
```bash
curl -X POST https://api.shhhtype.com/v1/accounts/{ACCOUNT_ID}/licenses/actions/validate-key \
  -H "Content-Type: application/json" \
  -d '{
    "meta": {
      "key": "SHHH-XXXX-XXXX-XXXX-XXXX"
    }
  }'
```

**Response:**
```json
{
  "meta": {
    "valid": true,
    "detail": "is valid",
    "code": "VALID"
  },
  "data": {
    "type": "licenses",
    "attributes": {
      "key": "SHHH-XXXX-XXXX-XXXX-XXXX",
      "status": "ACTIVE",
      "expiry": "2026-04-03T00:00:00Z",
      "metadata": {
        "plan": "trial"
      }
    }
  }
}
```

### Validation Codes
| Code | Meaning | App Behavior |
|------|---------|-------------|
| `VALID` | License is active | Full access |
| `EXPIRED` | Trial/subscription ended | Show upgrade screen |
| `SUSPENDED` | Payment failed or admin action | Show "Contact support" |
| `NO_MACHINE` | Not activated on this device | Prompt machine activation |
| `TOO_MANY_MACHINES` | Exceeded device limit | Show "Deactivate another device" |

### Machine Activation
```bash
curl -X POST https://api.shhhtype.com/v1/accounts/{ACCOUNT_ID}/machines \
  -H "Authorization: License {LICENSE_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "data": {
      "type": "machines",
      "attributes": {
        "fingerprint": "{MACHINE_UUID}",
        "name": "Johns MacBook Pro"
      },
      "relationships": {
        "license": {
          "data": { "type": "licenses", "id": "{LICENSE_ID}" }
        }
      }
    }
  }'
```

### Offline / Grace Period
- Cache validation result in macOS Keychain for 24 hours
- If network unavailable and cache is valid → allow access
- 3-day grace period after expiry before hard lock
- Grace period allows users time to renew without losing workflow

---

## 4. Purchase / Subscription Flow

### Two Paths

#### Path A: Zoho Billing (Primary)
```
User clicks "Subscribe" (landing page or in-app)
  → Redirect to Zoho Billing hosted checkout page
  → User selects Monthly ($19) or Annual ($200)
  → Enters payment info (Zoho handles PCI compliance)
  → On success:
      Zoho Billing webhook → Zoho Flow
        → Call Keygen API: upgrade license from TRIAL to SUBSCRIPTION
        → Update Zoho CRM contact: status = "Customer", plan = "monthly/annual"
        → Send confirmation email with receipt
  → License key stays the same — just gets upgraded
```

#### Path B: LemonSqueezy (Fallback)
```
User clicks alternate payment link
  → LemonSqueezy checkout page
  → On success:
      LemonSqueezy webhook → Zoho Flow (or direct webhook handler)
        → Call Keygen API: upgrade license
        → Update Zoho CRM contact
        → Send confirmation email
```

### Keygen: Upgrade License (TRIAL → SUBSCRIPTION)
```bash
curl -X PATCH https://api.shhhtype.com/v1/accounts/{ACCOUNT_ID}/licenses/{LICENSE_ID} \
  -H "Authorization: Bearer {ADMIN_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "data": {
      "type": "licenses",
      "attributes": {
        "expiry": "2026-04-27T00:00:00Z",
        "metadata": {
          "plan": "monthly",
          "zoho_subscription_id": "SUB-XXXXX"
        }
      },
      "relationships": {
        "policy": {
          "data": { "type": "policies", "id": "{SUBSCRIPTION_POLICY_ID}" }
        }
      }
    }
  }'
```

### Subscription Policy (Keygen Configuration)
| Setting | Value |
|---------|-------|
| Name | `ShhhType Subscription` |
| Duration | null (managed by Zoho Billing) |
| Max Machines | 2 |
| Auth Strategy | LICENSE |
| Expiration Strategy | RESTRICT_ACCESS |
| Heartbeat | Optional (every 24h) |

---

## 5. Renewal & Cancellation

### Auto-Renewal (Zoho Billing)
```
Zoho Billing charges card on renewal date
  → Success: webhook → Zoho Flow → Keygen: extend license expiry
  → Failure: Zoho Billing dunning process starts
```

### Dunning (Failed Payment)
| Attempt | Timing | Action |
|---------|--------|--------|
| 1 | Day 0 | Retry charge, email "Payment failed" |
| 2 | Day 3 | Retry charge, email "Update payment method" |
| 3 | Day 7 | Final retry, email "Last chance" |
| — | Day 10 | Suspend subscription → Keygen suspends license |

### Cancellation
```
User cancels in Zoho Billing portal (or requests via email)
  → Zoho Billing marks subscription as "cancelling"
  → Access continues until current period ends
  → At period end: Zoho webhook → Keygen suspends license
  → Zoho CRM contact status → "Churned"
  → Win-back email sent 7 days later (optional)
```

### Reactivation
```
User re-subscribes via Zoho Billing
  → New subscription created
  → Webhook → Keygen: reactivate existing license (or issue new one)
  → Zoho CRM contact status → "Customer"
```

---

## 6. Zoho Setup Guide

### Zoho CRM: Custom Fields
Add these fields to the **Contacts** module:

| Field Name | Type | Values |
|-----------|------|--------|
| `License Key` | Single Line | — |
| `Plan Type` | Picklist | Trial, Monthly, Annual |
| `Account Status` | Picklist | Trial, Trial Expired, Customer, Churned, Suspended |
| `Trial Start Date` | Date | — |
| `Trial End Date` | Date | — |
| `Subscription ID` | Single Line | — |
| `Machine Count` | Number | — |

### Zoho Billing: Subscription Plans

**Plan 1: ShhhType Monthly**
| Setting | Value |
|---------|-------|
| Plan Code | `shhhtype-monthly` |
| Name | ShhhType Monthly |
| Price | $19.00 |
| Billing Cycle | Monthly |
| Trial Period | 0 days (trial handled by Keygen) |

**Plan 2: ShhhType Annual**
| Setting | Value |
|---------|-------|
| Plan Code | `shhhtype-annual` |
| Name | ShhhType Annual |
| Price | $200.00 |
| Billing Cycle | Yearly |
| Trial Period | 0 days |
| Description | Save $28/year |

### Zoho Flow: Automations to Configure

**Flow 1: New Trial Signup**
- Trigger: Keygen webhook `license.created` (policy = Trial)
- Action: Create Zoho CRM Contact with license key, status = "Trial"
- Action: Send welcome email via Zoho

**Flow 2: Trial Expiring Reminder**
- Trigger: Scheduled (daily check)
- Condition: Trial end date = today + 2 days
- Action: Send reminder email
- Action: Update CRM contact status = "Trial Expiring"

**Flow 3: Trial Expired**
- Trigger: Keygen webhook `license.expired`
- Action: Update CRM contact status = "Trial Expired"
- Action: Send trial expired email with subscribe CTA

**Flow 4: Subscription Created**
- Trigger: Zoho Billing webhook `subscription_created`
- Action: Call Keygen API to upgrade license
- Action: Update CRM contact status = "Customer"

**Flow 5: Subscription Renewed**
- Trigger: Zoho Billing webhook `subscription_renewed`
- Action: Call Keygen API to extend license expiry

**Flow 6: Subscription Cancelled**
- Trigger: Zoho Billing webhook `subscription_cancelled`
- Action: Call Keygen API to suspend license at period end
- Action: Update CRM contact status = "Churned"

**Flow 7: Payment Failed**
- Trigger: Zoho Billing webhook `payment_failed`
- Action: Update CRM contact with payment failure flag

---

## 7. Keygen Setup Guide

### Environment Variables (Coolify)
```env
KEYGEN_ACCOUNT_ID=your-account-id
KEYGEN_ADMIN_TOKEN=your-admin-bearer-token
KEYGEN_TRIAL_POLICY_ID=your-trial-policy-id
KEYGEN_SUBSCRIPTION_POLICY_ID=your-subscription-policy-id
ZOHO_CRM_CLIENT_ID=your-zoho-client-id
ZOHO_CRM_CLIENT_SECRET=your-zoho-client-secret
ZOHO_CRM_REFRESH_TOKEN=your-zoho-refresh-token
ZOHO_BILLING_WEBHOOK_SECRET=your-webhook-secret
LEMONSQUEEZY_WEBHOOK_SECRET=your-ls-webhook-secret
```

### Webhook Configuration (Keygen → Zoho Flow)
Set up webhooks in Keygen admin for these events:
- `license.created` → `https://flow.zoho.com/webhook/keygen-license-created`
- `license.expired` → `https://flow.zoho.com/webhook/keygen-license-expired`
- `license.renewed` → `https://flow.zoho.com/webhook/keygen-license-renewed`
- `license.suspended` → `https://flow.zoho.com/webhook/keygen-license-suspended`

---

## 8. API Endpoints Summary (Coolify Server)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/signup` | Create account + generate trial license |
| `POST` | `/api/validate` | Validate license key (proxy to Keygen) |
| `POST` | `/api/activate` | Activate license on a machine |
| `POST` | `/api/deactivate` | Deactivate a machine |
| `POST` | `/webhooks/zoho-billing` | Handle Zoho Billing subscription events |
| `POST` | `/webhooks/lemonsqueezy` | Handle LemonSqueezy payment events |

---

## 9. Landing Page Integration

### Signup Modal
- Collects: Full Name, Email, Password
- Submits to: `POST https://api.shhhtype.com/api/signup`
- On success: Shows license key + download link
- On error: Shows error message (duplicate email, validation, etc.)

### Pricing CTAs
- **"Start Free Trial"** → Opens signup modal
- After signup, user can subscribe via:
  - In-app upgrade prompt → Zoho Billing hosted checkout
  - Email links → Zoho Billing hosted checkout

### Subscribe Flow (Post-Trial)
- In-app: "Subscribe" button → opens browser to Zoho Billing checkout
- Checkout URL format: `https://billing.zoho.com/subscribe/{PLAN_CODE}?email={USER_EMAIL}`
- Pre-fill email from license/account

---

## 10. Hybrid LemonSqueezy Approach

Keep LemonSqueezy as a fallback payment processor for:
- Users in regions where Zoho Billing has limited payment method support
- Alternative if Zoho Billing experiences downtime
- A/B testing checkout conversion rates

### Implementation
- Primary checkout: Zoho Billing hosted page
- Secondary: LemonSqueezy checkout link (accessible via "Other payment methods" link)
- Both fire webhooks to the same Keygen upgrade flow
- Both update Zoho CRM contact status

### LemonSqueezy Webhook Handler
```
POST /webhooks/lemonsqueezy
  → Verify webhook signature
  → Extract customer email + order data
  → Find user in Keygen by email
  → Upgrade license from TRIAL to SUBSCRIPTION
  → Update Zoho CRM contact via API
```
