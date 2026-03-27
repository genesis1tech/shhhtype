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
              +------+-------+  +-------+-------+
                     |                  |
                     |        (fallback) |
                     v                  v
              +------+-------+  +-------+--------+
              |Zoho Campaigns|  | LemonSqueezy   |
              |              |  | (alt payments) |
              | - Welcome    |  +----------------+
              | - Trial drip |
              | - Onboarding |
              | - Win-back   |
              | - Newsletter |
              +--------------+

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

| Day | Action | Email (Zoho Campaigns) |
|-----|--------|------------------------|
| 0 | Account created, trial license issued | Welcome + license key + download link |
| 1 | — | Onboarding: "Set up your first hotkey" |
| 3 | — | Tips: "Try AI rewrite & cloud vs local" |
| 5 | CRM status → "Trial Expiring" | Urgency: "Your trial expires in 2 days" |
| 6 | — | Feature highlight: "What you'll miss" |
| 7 | License expires, app shows upgrade prompt | Trial expired + subscribe CTA |
| 10 | — | Win-back: "We miss you — 20% off first month" |
| 14 | — | Last chance: "Final reminder before we close your trial" |
| 30+ | CRM status → "Churned" (if no purchase) | — |

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
- Action: Add contact to Zoho Campaigns "Trial Drip" mailing list
- Action: Trigger Campaigns welcome sequence (Day 0 email)

**Flow 2: Trial Expiring**
- Trigger: Scheduled (daily check)
- Condition: Trial end date = today + 2 days
- Action: Update CRM contact status = "Trial Expiring"
- (Campaigns handles Day 5/6 emails automatically via drip sequence)

**Flow 3: Trial Expired**
- Trigger: Keygen webhook `license.expired`
- Action: Update CRM contact status = "Trial Expired"
- Action: Move contact from "Trial Drip" to "Win-Back" list in Campaigns

**Flow 4: Subscription Created**
- Trigger: Zoho Billing webhook `subscription_created`
- Action: Call Keygen API to upgrade license
- Action: Update CRM contact status = "Customer"
- Action: Remove contact from trial/win-back lists in Campaigns
- Action: Add contact to "Customers" list in Campaigns

**Flow 5: Subscription Renewed**
- Trigger: Zoho Billing webhook `subscription_renewed`
- Action: Call Keygen API to extend license expiry

**Flow 6: Subscription Cancelled**
- Trigger: Zoho Billing webhook `subscription_cancelled`
- Action: Call Keygen API to suspend license at period end
- Action: Update CRM contact status = "Churned"
- Action: Move contact to "Win-Back" list in Campaigns

**Flow 7: Payment Failed**
- Trigger: Zoho Billing webhook `payment_failed`
- Action: Update CRM contact with payment failure flag
- Action: Trigger "Payment Failed" workflow in Campaigns

---

## 7. Zoho Campaigns: Email Sequences

### Mailing Lists

| List Name | Purpose | Entry Trigger |
|-----------|---------|---------------|
| `Trial Drip` | Onboarding + trial conversion | Account created (Zoho Flow 1) |
| `Win-Back` | Re-engage expired trials & churned | Trial expired or subscription cancelled |
| `Customers` | Active subscribers | Subscription created |
| `Newsletter` | Product updates, tips, releases | Opt-in (all users) |

### Sequence 1: Trial Drip (7 emails over 14 days)

Triggered when contact is added to "Trial Drip" list.

| # | Day | Subject Line | Content |
|---|-----|-------------|---------|
| 1 | 0 | Welcome to ShhhType! Here's your license key | License key, download link, quick-start guide, setup video link |
| 2 | 1 | Set up your first hotkey in 30 seconds | Step-by-step hotkey setup, screenshot walkthrough, link to docs |
| 3 | 3 | You're missing the best part: AI Rewrite | Demo of AI rewrite feature, cloud vs local mode comparison, tips |
| 4 | 5 | Your trial expires in 2 days | Feature recap, usage stats if available, subscribe CTA with pricing |
| 5 | 6 | Here's what you'll lose tomorrow | Side-by-side: with vs without ShhhType, urgency CTA |
| 6 | 7 | Your trial has ended | "We hope you loved it", subscribe CTA, pricing ($19/mo or $200/yr) |
| 7 | 10 | We miss you — here's 20% off your first month | Discount code `COMEBACK20`, limited time (72 hours), subscribe CTA |

**Exit Condition:** Contact subscribes (moved to Customers list) → immediately stop drip sequence.

### Sequence 2: Win-Back (3 emails over 21 days)

Triggered when contact is added to "Win-Back" list (trial expired without purchase, or subscription cancelled).

| # | Day | Subject Line | Content |
|---|-----|-------------|---------|
| 1 | 0 | We saved your spot | Remind what they're missing, 20% discount code, subscribe CTA |
| 2 | 7 | Still typing everything by hand? | Pain point reminder, testimonials/use cases, subscribe CTA |
| 3 | 14 | Last chance: your discount expires tomorrow | Final 20% off reminder, urgency, direct subscribe link |

**Exit Condition:** Contact subscribes → stop sequence, move to Customers list.

### Sequence 3: Customer Onboarding (3 emails over 7 days)

Triggered when contact is added to "Customers" list.

| # | Day | Subject Line | Content |
|---|-----|-------------|---------|
| 1 | 0 | You're all set! Your ShhhType subscription is active | Confirmation, receipt link, quick tips, support contact |
| 2 | 3 | 3 power-user tips you probably missed | Advanced features: custom hotkeys, language switching, AI rewrite prompts |
| 3 | 7 | How's it going? We'd love your feedback | NPS survey or short feedback form, link to leave a review |

### Sequence 4: Payment Failed (3 emails over 10 days)

Triggered by Zoho Flow 7 when payment fails.

| # | Day | Subject Line | Content |
|---|-----|-------------|---------|
| 1 | 0 | Heads up: your payment didn't go through | Update payment method link (Zoho Billing portal), support contact |
| 2 | 3 | Your ShhhType access is at risk | Reminder to update payment, what happens if not resolved |
| 3 | 7 | Last chance to keep your subscription | Final warning, access will be suspended in 3 days |

**Exit Condition:** Payment succeeds → stop sequence.

### Sequence 5: Cancellation / Churn Prevention (2 emails)

Triggered when subscription is marked as "cancelling" (before period end).

| # | Day | Subject Line | Content |
|---|-----|-------------|---------|
| 1 | 0 | We're sorry to see you go | Confirm cancellation, access until period end, feedback survey |
| 2 | 7 (after expiry) | Come back anytime — 25% off for 3 months | Win-back offer, resubscribe CTA, discount code `WELCOME25` |

### Campaigns Setup Checklist

1. **Create mailing lists:** Trial Drip, Win-Back, Customers, Newsletter
2. **Design email templates** using ShhhType branding (rose/orange gradient, Montserrat font)
3. **Create autoresponder workflows** for each sequence above
4. **Set exit conditions** so contacts don't receive irrelevant emails after converting
5. **Configure CRM sync** so Campaigns lists stay in sync with CRM contact status
6. **Set up merge tags** for personalization:
   - `${CONTACT.First Name}` — personalized greeting
   - `${CUSTOM.License Key}` — include in welcome email
   - `${CUSTOM.Trial End Date}` — show expiry in urgency emails
   - `${CUSTOM.Plan Type}` — tailor content to plan
7. **Enable double opt-in** for Newsletter list (CAN-SPAM/GDPR compliance)
8. **Add unsubscribe footer** to all sequences (auto-handled by Campaigns)

### Discount Codes (Referenced in Emails)

| Code | Discount | Valid For | Used In |
|------|----------|-----------|---------|
| `COMEBACK20` | 20% off first month | 72 hours | Trial Drip #7, Win-Back #1 & #3 |
| `WELCOME25` | 25% off for 3 months | 14 days | Churn Prevention #2 |

> **Note:** Discount codes are created in Zoho Billing as coupon codes and referenced in Campaigns emails. The subscribe CTA links should include `?coupon={CODE}` parameter.

---

## 8. Keygen Setup Guide

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
ZOHO_CAMPAIGNS_AUTH_TOKEN=your-campaigns-auth-token
ZOHO_CAMPAIGNS_TRIAL_LIST_KEY=your-trial-list-key
ZOHO_CAMPAIGNS_WINBACK_LIST_KEY=your-winback-list-key
ZOHO_CAMPAIGNS_CUSTOMERS_LIST_KEY=your-customers-list-key
LEMONSQUEEZY_WEBHOOK_SECRET=your-ls-webhook-secret
```

### Webhook Configuration (Keygen → Zoho Flow)
Set up webhooks in Keygen admin for these events:
- `license.created` → `https://flow.zoho.com/webhook/keygen-license-created`
- `license.expired` → `https://flow.zoho.com/webhook/keygen-license-expired`
- `license.renewed` → `https://flow.zoho.com/webhook/keygen-license-renewed`
- `license.suspended` → `https://flow.zoho.com/webhook/keygen-license-suspended`

---

## 9. API Endpoints Summary (Coolify Server)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/signup` | Create account + generate trial license |
| `POST` | `/api/validate` | Validate license key (proxy to Keygen) |
| `POST` | `/api/activate` | Activate license on a machine |
| `POST` | `/api/deactivate` | Deactivate a machine |
| `POST` | `/webhooks/zoho-billing` | Handle Zoho Billing subscription events |
| `POST` | `/webhooks/lemonsqueezy` | Handle LemonSqueezy payment events |

---

## 10. Landing Page Integration

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

## 11. Hybrid LemonSqueezy Approach

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
