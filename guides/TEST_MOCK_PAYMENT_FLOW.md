---
title: "Testing Mock Payment Flow - Step-by-Step Guide"
description: "A step-by-step test scenario for a PACRA annual return payment against the mock gateway, from filing creation through checkout."
---

## Prerequisites

Ensure your dev environment is ready:

```bash
# Navigate to project root
cd c:\Users\elija\OneDrive\Desktop\bumara

# Install dependencies (if not already done)
pnpm install

# Start all dev servers
pnpm dev
```

This will start:
- **App (Tenant)**: http://localhost:3000
- **Backoffice**: http://localhost:3002
- **Backend API**: http://localhost:3000/api/v1

---

## Test Scenario: PACRA Annual Return Payment

### Step 1: Create a Filing that Requires Payment

**Option A: Via Tenant App UI**

1. Navigate to http://localhost:3000
2. Login as a tenant organization
3. Go to **Filings** or **Compliance Dashboard**
4. Create a new **PACRA Annual Return** filing
   - This should auto-create a payment request with status `required_pending`

**Option B: Via Database (Quick Setup)**

```sql
-- Insert a test payment request
INSERT INTO payment_requests (
  id,
  organization_id,
  filing_id,
  regulator_id,
  status,
  regulator_fee,
  handling_fee,
  total_amount,
  currency,
  created_at,
  updated_at
) VALUES (
  gen_random_uuid(),
  'YOUR_ORG_ID_HERE', -- Replace with actual org ID
  'YOUR_FILING_ID_HERE', -- Replace with actual filing ID
  'reg_pacra',
  'required_pending',
  50000, -- K500 regulator fee
  0,     -- K0 handling fee (0% for filings)
  50000, -- K500 total
  'ZMW',
  NOW(),
  NOW()
);
```

### Step 2: Initiate Payment

**Via Tenant App:**

1. Navigate to the filing detail page
2. Click the **"Pay Now"** button
3. The backend will call:
   ```
   POST /api/v1/compliance/payments/{paymentRequestId}/initiate
   {
     "successUrl": "/filings/{id}/success",
     "cancelUrl": "/filings/{id}"
   }
   ```

**Expected Result:**
- Payment status changes to: `pending_gateway`
- You are redirected to: `http://localhost:3000/api/v1/payments/mock/checkout/{sessionId}`

### Step 3: Mock Checkout UI

You should see a **beautiful checkout page** with:
- 💳 Payment gateway header
- 💰 Amount display: **ZMW 500.00**
- 📋 Description: "Compliance payment for [Organization Name]"
- Two buttons:
  - ✅ **Simulate Success** (green)
  - ❌ **Simulate Failure** (red)

**Screenshot Check:**
- Background: Purple gradient
- Card: White with shadow
- Amount: Large, bold, centered
- Session ID in footer

### Step 4: Simulate Payment Success

1. Click **"Simulate Success"** button
2. JavaScript sends:
   ```json
   POST /api/v1/payments/mock/simulate
   {
     "sessionId": "ps_mock_...",
     "outcome": "success"
   }
   ```

**What Happens (Backend):**

1. **Gateway Simulation**:
   ```typescript
   gateway.simulatePaymentSuccess(sessionId)
   // Creates payment: { id: "pay_mock_...", status: "succeeded" }
   ```

2. **Webhook Event Trigger**:
   ```typescript
   processPaymentWebhookEvent({
     type: "payment.succeeded",
     data: { paymentId, paymentRequestId, metadata }
   })
   ```

3. **Auto-Verification**:
   ```typescript
   // Update payment: status = "paid_gateway_verified"
   handlePaymentVerification({
     paymentRequestId,
     verifiedByAgentId: null, // system
     verificationNotes: "Auto-verified via mock webhook"
   })
   // Update payment: status = "paid_platform_verified"
   ```

4. **Submission Job Creation** (if gates pass):
   ```typescript
   computeSubmissionReadiness()
   // If all gates green → createSubmissionJob()
   ```

5. **Redirect to Success**:
   ```
   window.location.href = successUrl + "?payment_status=success&payment_id=" + paymentId
   ```

### Step 5: Verify Results

**Check Payment Status:**

```sql
SELECT
  id,
  status,
  paid_at,
  external_payment_id,
  payment_provider,
  created_at,
  updated_at
FROM payment_requests
WHERE id = 'YOUR_PAYMENT_ID'
ORDER BY updated_at DESC;
```

**Expected:**
```
status: "paid_platform_verified"
paid_at: [timestamp]
external_payment_id: "pay_mock_1234567890"
payment_provider: "mock"
```

**Check Submission Job Created:**

```sql
SELECT
  id,
  source_type,
  source_id,
  regulator_id,
  status,
  created_at
FROM submission_jobs
WHERE source_id = 'YOUR_FILING_ID'
ORDER BY created_at DESC
LIMIT 1;
```

**Expected:**
```
source_type: "filing"
source_id: [your filing ID]
status: "pending"
regulator_id: "reg_pacra"
```

**Check Backend Logs:**

Look for these log entries:
```
✅ "Payment succeeded via webhook, marked as paid and gateway-verified"
✅ "Payment auto-verified and submission triggered (if gates passed)"
✅ "Submission job created"
```

---

## Alternative Test: Simulate Payment Failure

### Steps:

1. Follow Steps 1-3 above (create filing, initiate payment, see checkout)
2. Click **"Simulate Failure"** button instead
3. JavaScript sends:
   ```json
   POST /api/v1/payments/mock/simulate
   {
     "sessionId": "ps_mock_...",
     "outcome": "failure",
     "failureReason": "insufficient_funds"
   }
   ```

**Expected Results:**

**Payment Status:**
```sql
SELECT status FROM payment_requests WHERE id = 'YOUR_PAYMENT_ID';
-- Expected: "required_pending" (reverted back)
```

**No Submission Job:**
```sql
SELECT COUNT(*) FROM submission_jobs WHERE source_id = 'YOUR_FILING_ID';
-- Expected: 0 (no job created on failure)
```

**Redirect:**
```
User redirected back to cancelUrl (filing detail page)
Error message shown: "Payment failed"
```

---

## Test Checklist

Use this checklist to verify all functionality:

### ✅ Gateway Configuration
- [ ] `PAYMENT_PROVIDER=mock` in backend .env
- [ ] `BASE_URL=http://localhost:3000` in backend .env
- [ ] Dev server running (`pnpm dev`)

### ✅ Payment Initiation
- [ ] Filing/service request created
- [ ] "Pay Now" button visible
- [ ] Click "Pay Now" redirects to checkout
- [ ] Payment status changes to `pending_gateway`

### ✅ Mock Checkout UI
- [ ] Checkout page loads at `/api/v1/payments/mock/checkout/{sessionId}`
- [ ] Amount displays correctly (ZMW X.XX)
- [ ] Description shows organization name
- [ ] Two buttons visible: Success (green) + Failure (red)
- [ ] Session ID shown in footer

### ✅ Payment Success Flow
- [ ] Click "Simulate Success"
- [ ] Processing spinner appears
- [ ] Redirected to success URL after ~2 seconds
- [ ] Payment status = `paid_gateway_verified` → `paid_platform_verified`
- [ ] Submission job created (check DB)
- [ ] Backend logs show "auto-verified" message

### ✅ Payment Failure Flow
- [ ] Click "Simulate Failure" (on fresh checkout)
- [ ] Payment status reverts to `required_pending`
- [ ] No submission job created
- [ ] User sees error or returns to cancel URL

### ✅ Auto-Verification
- [ ] Payment verification happens automatically (no manual backoffice step)
- [ ] `verifiedByAgentId` is `null` (system verification)
- [ ] Verification notes mention "Auto-verified via mock webhook"

### ✅ Submission Job Creation
- [ ] Submission job created if all gates pass
- [ ] Job has correct `source_type` and `source_id`
- [ ] Job status is `pending`
- [ ] Job visible in backoffice submission queue

---

## Troubleshooting

### Issue: Checkout 404 Not Found

**Symptom**: Clicking "Pay Now" redirects to 404

**Solution**:
1. Check `PAYMENT_PROVIDER=mock` in backend .env
2. Restart dev server: `pnpm dev`
3. Check backend logs for route registration

### Issue: Payment Not Auto-Verified

**Symptom**: Payment status stuck at `paid_gateway_verified`

**Solution**:
1. Check backend logs for errors in `handlePaymentVerification`
2. Ensure `handlePaymentVerification` is exported from api-services
3. Check if payment request has valid `organizationId`

### Issue: Submission Job Not Created

**Symptom**: Payment verified but no submission job

**Possible Causes**:
1. **Gates not passing**: Required tasks incomplete
2. **Filing not ready**: Missing required documents
3. **Check gates**:
   ```sql
   -- Check what gates are blocking
   SELECT * FROM tasks
   WHERE filing_id = 'YOUR_FILING_ID'
   AND status != 'completed';
   ```

**Solution**:
- Complete all required tasks first
- Upload required documents
- Ensure filing is in correct status

### Issue: Wrong Amount Displayed

**Symptom**: Checkout shows wrong amount

**Solution**:
- Check fee calculator is returning correct amount
- Verify `regulator_fee + handling_fee = total_amount`
- Check currency conversion (ngwee → kwacha)

---

## Next Steps After Testing

Once testing is complete:

1. **Switch back to Stripe** (when ready):
   ```bash
   # In packages/backend/.env
   PAYMENT_PROVIDER=stripe
   # Uncomment Stripe keys
   ```

2. **Write automated tests**:
   - Location: `packages/backend/src/modules/compliance/payments/__tests__/`
   - Test cases: Mock flow, Stripe webhooks, auto-verification

3. **Deploy to staging**:
   - Keep `PAYMENT_PROVIDER=mock` on staging for testing
   - Switch to `stripe` only on production

---

## Quick Test Command (Optional)

If you want to quickly test without UI, you can use this curl command:

```bash
# 1. Get session ID from initiatePayment response
SESSION_ID="ps_mock_1234567890"

# 2. Open checkout in browser
open "http://localhost:3000/api/v1/payments/mock/checkout/${SESSION_ID}"

# 3. Or simulate directly via API
curl -X POST http://localhost:3000/api/v1/payments/mock/simulate \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "'${SESSION_ID}'",
    "outcome": "success"
  }'
```

---

**Happy Testing!** 🎉

If you encounter any issues, check:
- Backend logs (look for "Payment" or "webhook" entries)
- Database payment_requests table
- Database submission_jobs table
