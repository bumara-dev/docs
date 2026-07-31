---
title: "Payment Flow Testing Guide"
description: "Manual test walkthrough for the end-to-end payment flow, covering the mock gateway, the payment modal, and auto-verification."
---

## Overview
This guide walks through manual testing of the complete payment flow integration, including the mock payment gateway, payment modal UI, and auto-verification.

## Prerequisites

### Backend Configuration
Ensure `packages/backend/.env` is configured for mock mode:
```env
PAYMENT_PROVIDER=mock
BASE_URL=http://localhost:3000
# SKIP_PAYMENT_CHECKS=true  # This should be commented out or removed
```

### Dev Server Running
```bash
pnpm dev
```
Server should be running on `http://localhost:3000`

## Test Scenarios

### Scenario 1: Filing Payment Flow

#### Step 1: Navigate to a Filing
1. Go to: `http://localhost:3000/regulators/pacra/filings`
2. Select a filing that requires payment
3. Or create a new filing that will require payment

#### Step 2: Complete Required Tasks
1. Complete any required tasks (if applicable)
2. Upload any required documents (if applicable)
3. The filing should now show "Ready for Submission"

#### Step 3: Request Submission
1. Click **"Request Submission"** button
2. If payment is required, the system will:
   - Create a payment request record
   - Show the **Payment Required Modal**

#### Step 4: Review Payment Modal
The modal should display:
- **Header**: Credit card icon with "Payment Required" title
- **Amount Card**: Gradient card showing:
  - Regulator fee (e.g., ZMW 100.00)
  - Handling fee (e.g., ZMW 10.00)
  - Total amount (e.g., ZMW 110.00)
- **How It Works**: 3 numbered steps with colored badges
- **Pay Now Button**: Prominent button with three icons
- **X Close Button**: Only in header (no duplicate close buttons)

#### Step 5: Initiate Payment
1. Click **"Pay Now"** button
2. The button should show loading state: "Redirecting to checkout..."
3. Modal cannot be closed during redirect
4. Browser should redirect to mock checkout page:
   ```
   http://localhost:3000/api/v1/mock/checkout?session_id=...
   ```

#### Step 6: Complete Mock Payment
On the checkout page:
1. Review payment details (amount, reference)
2. Click **"Simulate Successful Payment"** button
3. The mock gateway will:
   - Process the payment
   - Update payment status to `paid_platform_verified`
   - Trigger webhook auto-verification
   - Create submission job
4. Redirect to success page with query param: `?payment_status=success`

#### Step 7: Verify Submission Created
1. After redirect, you should see a success toast
2. Navigate to submissions to verify:
   - Submission job was created
   - Job status is `queued` or `assigned`
   - Payment status is `paid_platform_verified`

### Scenario 2: Service Request Payment Flow

Follow the same steps as Scenario 1, but for service requests:
1. Navigate to: `http://localhost:3000/regulators/pacra/service-requests`
2. Select or create a service request that requires payment
3. Complete tasks and documents
4. Click "Request Bumara Submission"
5. Complete payment flow as described above

### Scenario 3: View Existing Payment

#### When Payment Already Exists
1. Navigate to a filing/service request with an existing payment
2. The blockers panel should show "Payment Required"
3. Click **"View Payment Details"** button
4. Modal opens with existing payment details
5. If payment is not yet completed, you can click "Pay Now"
6. If payment is completed, modal shows status accordingly

### Scenario 4: Cancel Payment

1. Follow steps to initiate payment
2. On the mock checkout page, click **"Cancel Payment"**
3. Should redirect back with `?payment_status=cancel`
4. Payment status remains `pending`
5. You can retry payment later

### Scenario 5: Payment Failure

1. Follow steps to initiate payment
2. On mock checkout page, click **"Simulate Failed Payment"**
3. Payment status should be updated to `failed`
4. Error message should be shown
5. You can retry by creating a new payment request

## Expected Behaviors

### Payment Status Flow
```
pending → processing → paid_gateway_confirmed → paid_platform_verified → submission_job_created
```

### Auto-Verification
When payment is successful, the webhook handler should:
1. Verify payment with gateway
2. Update status to `paid_platform_verified`
3. Check if source (filing/service request) is ready
4. Create submission job if ready
5. Update payment status to `submission_job_created`

### UI States

#### Payment Modal States
- **Initial**: Shows payment details and "Pay Now" button
- **Loading**: Shows "Redirecting to checkout..." with spinner
- **Closed**: Modal is dismissed, no payment initiated

#### Blockers Panel States
- **Payment Pending**: Shows "Payment Required" with "View Payment Details" button
- **Payment Complete**: Payment blocker is removed
- **Ready for Submission**: Shows green "Request Submission" button

## Common Issues & Troubleshooting

### Issue: "Payment Required" modal doesn't appear
**Solution**:
- Check that `SKIP_PAYMENT_CHECKS` is NOT set in `.env`
- Verify payment calculation returns an amount > 0
- Check browser console for errors

### Issue: Redirect to checkout fails
**Solution**:
- Verify `BASE_URL=http://localhost:3000` in `.env`
- Check that backend server is running
- Check network tab for API errors

### Issue: Webhook auto-verification doesn't trigger
**Solution**:
- Check backend logs for webhook processing errors
- Verify payment status in database
- Check that submission source is actually ready (all tasks done, docs uploaded)

### Issue: Modal shows wrong payment amount
**Solution**:
- Check regulator fee configuration
- Verify handling fee calculation (10% default)
- Check payment calculator in `packages/payments/calculator.ts`

## Database Verification

### Check Payment Status
```sql
SELECT
  id,
  reference,
  regulator_fee,
  handling_fee,
  total_amount,
  status,
  created_at
FROM payment_requests
WHERE source_type = 'filing' AND source_id = '<filing-id>'
ORDER BY created_at DESC
LIMIT 1;
```

### Check Submission Job
```sql
SELECT
  id,
  status,
  source_type,
  source_id,
  created_at
FROM submission_jobs
WHERE source_type = 'filing' AND source_id = '<filing-id>'
ORDER BY created_at DESC
LIMIT 1;
```

### Check Payment Gateway Records
```sql
SELECT
  id,
  session_id,
  status,
  gateway_payment_id,
  created_at
FROM payment_gateway_records
WHERE payment_request_id = '<payment-id>'
ORDER BY created_at DESC;
```

## Testing Checklist

- [ ] Filing payment flow works end-to-end
- [ ] Service request payment flow works end-to-end
- [ ] Payment modal displays correctly (no duplicate close buttons)
- [ ] Payment modal shows accurate amounts (regulator fee + handling fee)
- [ ] "Pay Now" button redirects to mock checkout
- [ ] Mock checkout displays payment details correctly
- [ ] "Simulate Successful Payment" completes payment
- [ ] Auto-verification triggers after payment success
- [ ] Submission job is created automatically
- [ ] "Cancel Payment" returns without creating submission
- [ ] "Simulate Failed Payment" updates status correctly
- [ ] "View Payment Details" opens modal with existing payment
- [ ] Payment status updates are reflected in UI
- [ ] Cannot close modal during redirect
- [ ] Browser navigation works correctly (back/forward)
- [ ] Loading states are shown appropriately
- [ ] Error states are handled gracefully
- [ ] Toast notifications appear at appropriate times
- [ ] Database records are created/updated correctly

## Next Steps After Testing

1. **Write Automated Tests**: See PAYMENT_FLOW_TESTS.md
2. **Switch to Stripe**: See STRIPE_INTEGRATION.md
3. **Add Payment Analytics**: Track conversion rates, failure reasons, etc.
4. **Implement Payment Reminders**: Notify users of unpaid payment requests
5. **Add Payment History**: Show payment timeline and audit trail

## Related Documentation

- [Payment Gateway Simulation](/ARCHITECTURE/payment-gateway-simulation)
- [Regulator Payment System](/ARCHITECTURE/regulator-payment-system)
- [Payment UI Integration Summary](/guides/PAYMENT_UI_INTEGRATION_SUMMARY)
