---
title: "Notifications Provider Setup"
description: "Configuration guides for Email (Resend) and WhatsApp (Meta) providers."
---

## Table of Contents

1. [Overview](#overview)
2. [Email Setup (Resend)](#email-setup-resend)
3. [WhatsApp Setup (Meta)](#whatsapp-setup-meta)
4. [Environment Variables](#environment-variables)
5. [Testing Providers](#testing-providers)

---

## Overview

The notification system uses two external providers:

| Provider | Channel | Purpose |
|----------|---------|---------|
| Resend | Email | Transactional emails |
| Meta WhatsApp Cloud API | WhatsApp | Template messages |

Both providers require account setup, domain/number verification, and webhook configuration.

---

## Email Setup (Resend)

### 1. Create Resend Account

1. Go to [resend.com](https://resend.com)
2. Sign up for an account
3. Verify your email address

### 2. Add Sending Domain

1. Go to **Domains** in the Resend dashboard
2. Click **Add Domain**
3. Enter your domain: `bumara.com` (or subdomain like `mail.bumara.com`)
4. Add the DNS records Resend provides:

| Type | Name | Value |
|------|------|-------|
| TXT | `_dmarc` | `v=DMARC1; p=none;` |
| TXT | `resend._domainkey` | (provided by Resend) |
| TXT | `@` | `v=spf1 include:resend.com ~all` |

5. Wait for verification (usually 5-10 minutes)

### 3. Create API Key

1. Go to **API Keys** in dashboard
2. Click **Create API Key**
3. Name: `bumara-notifications-production`
4. Permissions: `Full access` or `Sending access`
5. Copy the key (starts with `re_`)

### 4. Configure Webhooks

1. Go to **Webhooks** in dashboard
2. Click **Add Webhook**
3. Endpoint URL: `https://api.bumara.com/webhooks/email`
4. Select events:
   - `email.delivered`
   - `email.bounced`
   - `email.complained`
5. Copy the **Signing Secret**

### 5. Environment Variables

```bash
RESEND_TOKEN=re_xxxxx
RESEND_FROM=notifications@bumara.com
RESEND_WEBHOOK_SECRET=whsec_xxxxx
```

### 6. DNS Records Reference

```
; SPF record
@ IN TXT "v=spf1 include:resend.com ~all"

; DKIM record
resend._domainkey IN TXT "v=DKIM1; k=rsa; p=MIGfMA0GCSq..."

; DMARC record (start with none, upgrade to quarantine after monitoring)
_dmarc IN TXT "v=DMARC1; p=none; rua=mailto:dmarc@bumara.com"
```

---

## WhatsApp Setup (Meta)

### 1. Create Meta Business Account

1. Go to [business.facebook.com](https://business.facebook.com)
2. Create a Business Account if you don't have one
3. Complete business verification (may take days)

### 2. Set Up WhatsApp Business

1. Go to [developers.facebook.com](https://developers.facebook.com)
2. Create a new App → Select **Business** type
3. Add **WhatsApp** product to your app
4. Link to your Meta Business Account

### 3. Get Phone Number

For development:
- Use the test phone number provided by Meta (limited to 5 test recipients)

For production:
1. Go to **WhatsApp** → **Getting Started**
2. Click **Add Phone Number**
3. Register a real phone number (must be able to receive SMS/call for verification)
4. Complete phone number verification

### 4. Create Message Templates

Templates must be approved by Meta before use.

1. Go to **WhatsApp** → **Message Templates**
2. Click **Create Template**
3. Fill in details:

**Template: bumara_filing_reminder**
```
Category: UTILITY
Language: en

Header: None
Body: Reminder: Your {{1}} for {{2}} is due in {{3}} days. Submit before the deadline to avoid penalties.
Footer: Bumara - Business Compliance
Buttons: None
```

**Template: bumara_filing_overdue**
```
Category: UTILITY
Language: en

Header: None
Body: URGENT: Your {{1}} for {{2}} is {{3}} days overdue. Submit immediately to avoid penalties.
Footer: Bumara - Business Compliance
Buttons: None
```

**Template: bumara_task_assigned**
```
Category: UTILITY
Language: en

Header: None
Body: You have been assigned a new task: {{1}}. Log in to Bumara to view details.
Footer: Bumara - Business Compliance
Buttons: None
```

**Template: bumara_payment_required**
```
Category: UTILITY
Language: en

Header: None
Body: Payment of {{1}} is required to proceed with your submission. Please complete payment on Bumara.
Footer: Bumara - Business Compliance
Buttons: None
```

4. Submit each template for review (usually approved within 24 hours)

### 5. Configure Webhooks

1. Go to **WhatsApp** → **Configuration**
2. Under **Webhook**, click **Edit**
3. Callback URL: `https://api.bumara.com/webhooks/whatsapp`
4. Verify Token: Generate a random string (e.g., `bumara_whatsapp_verify_abc123`)
5. Click **Verify and Save**
6. Subscribe to **messages** field

### 6. Get Access Token

For development:
- Use temporary token from the Getting Started page (expires in 24 hours)

For production:
1. Go to **App Settings** → **Basic**
2. Note your **App ID** and **App Secret**
3. Generate a System User token with `whatsapp_business_messaging` permission
4. This token does not expire

### 7. Environment Variables

```bash
WHATSAPP_PHONE_NUMBER_ID=123456789012345
WHATSAPP_ACCESS_TOKEN=EAAxxxxx...
WHATSAPP_VERIFY_TOKEN=bumara_whatsapp_verify_abc123
WHATSAPP_WEBHOOK_SECRET=your_app_secret
WHATSAPP_API_VERSION=v18.0
```

### 8. Phone Number ID

To find your Phone Number ID:

1. Go to **WhatsApp** → **Getting Started**
2. The Phone Number ID is shown under your phone number
3. It's a 15-digit number (not the phone number itself)

---

## Environment Variables

### Complete Environment Configuration

```bash
# =========================================
# Email (Resend)
# =========================================
RESEND_TOKEN=re_xxxxx
RESEND_FROM=notifications@bumara.com
RESEND_WEBHOOK_SECRET=whsec_xxxxx

# =========================================
# WhatsApp (Meta)
# =========================================
# Phone Number ID (15-digit number from WhatsApp dashboard)
WHATSAPP_PHONE_NUMBER_ID=123456789012345

# System User Access Token (permanent token)
WHATSAPP_ACCESS_TOKEN=EAAxxxxx...

# Verify token for webhook setup
WHATSAPP_VERIFY_TOKEN=bumara_whatsapp_verify_abc123

# App Secret for webhook signature verification
WHATSAPP_WEBHOOK_SECRET=abcd1234...

# API version (use latest stable)
WHATSAPP_API_VERSION=v18.0
```

### Cloudflare Workers Configuration

Add secrets via Wrangler:

```bash
# Add secrets (don't commit to git)
wrangler secret put RESEND_TOKEN
wrangler secret put RESEND_WEBHOOK_SECRET
wrangler secret put WHATSAPP_ACCESS_TOKEN
wrangler secret put WHATSAPP_WEBHOOK_SECRET

# Add non-secret variables in wrangler.jsonc
{
  "vars": {
    "RESEND_FROM": "notifications@bumara.com",
    "WHATSAPP_PHONE_NUMBER_ID": "123456789012345",
    "WHATSAPP_VERIFY_TOKEN": "bumara_whatsapp_verify_abc123",
    "WHATSAPP_API_VERSION": "v18.0"
  }
}
```

---

## Testing Providers

### Email Testing

```typescript
// Test email sending
import { sendEmail } from '@repo/notifications/adapters/email';

async function testEmail() {
  const result = await sendEmail(env, {
    to: 'test@example.com',
    subject: 'Test Notification',
    html: '<h1>Test</h1><p>This is a test notification.</p>',
    text: 'Test - This is a test notification.',
    metadata: {
      deliveryId: 'test-123',
      notificationId: 'test-456',
      tenantId: 'test-org',
    },
  });
  
  console.log('Email sent:', result);
}
```

### WhatsApp Testing

With Meta test number, you can only send to numbers added as test recipients:

1. Go to **WhatsApp** → **Getting Started**
2. Add your phone number under **To** field
3. Send a test message

```typescript
// Test WhatsApp sending
import { sendWhatsAppTemplate } from '@repo/notifications/adapters/whatsapp';

async function testWhatsApp() {
  const result = await sendWhatsAppTemplate(env, {
    toE164: '+260971234567', // Must be a test recipient
    templateName: 'bumara_filing_reminder',
    language: 'en',
    variables: ['VAT Return', 'November 2025', '7'],
    metadata: {
      deliveryId: 'test-123',
      notificationId: 'test-456',
      tenantId: 'test-org',
    },
  });
  
  console.log('WhatsApp sent:', result);
}
```

### Webhook Testing

Use webhook testing tools:

1. **ngrok** - Tunnel local server to public URL
2. **Cloudflare Tunnel** - Alternative for Cloudflare users
3. **Meta Test Tool** - Built into WhatsApp dashboard

```bash
# Local development with ngrok
ngrok http 8787

# Update webhook URL in Meta dashboard to ngrok URL
# https://abc123.ngrok.io/webhooks/whatsapp
```

### Verify Webhook Signatures Locally

```typescript
// Test webhook verification
import { verifyWhatsAppSignature } from './whatsapp-webhook.handlers';

async function testWebhookVerification() {
  const body = '{"test": "payload"}';
  const secret = 'your_app_secret';
  
  // Generate signature
  const encoder = new TextEncoder();
  const key = await crypto.subtle.importKey(
    'raw',
    encoder.encode(secret),
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['sign']
  );
  const signature = await crypto.subtle.sign('HMAC', key, encoder.encode(body));
  const signatureHex = 'sha256=' + bufferToHex(signature);
  
  // Verify
  const isValid = await verifyWhatsAppSignature(body, signatureHex, secret);
  console.log('Signature valid:', isValid);
}
```

---

## Troubleshooting

### Email Issues

| Issue | Solution |
|-------|----------|
| Emails going to spam | Check SPF/DKIM/DMARC records, warm up domain |
| 401 Unauthorized | Check API key is correct and not expired |
| Rate limited | Upgrade Resend plan or implement backoff |
| Bounces not received | Verify webhook URL and signing secret |

### WhatsApp Issues

| Issue | Solution |
|-------|----------|
| Template rejected | Review Meta's template guidelines, avoid promotional language |
| Invalid recipient | Ensure phone is in E.164 format, user has WhatsApp |
| Rate limited | Check Meta's rate limits (varies by phone number quality) |
| Webhook not receiving | Verify callback URL, check app permissions |
| 190 error (token expired) | Generate new System User token |
| 131026 error | Recipient hasn't opted in or number invalid |

### Common Error Codes

**Meta WhatsApp:**
- `130429` - Rate limit exceeded
- `131026` - Invalid recipient
- `132000-132999` - Template errors
- `190` - Access token expired

**Resend:**
- `400` - Bad request (invalid email format)
- `401` - Invalid API key
- `429` - Rate limit exceeded

