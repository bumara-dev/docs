---
title: "Finance Module"
description: "Payments & Payouts documentation."
---

## Table of Contents

1. [Overview](#1-overview)
2. [Tenant Payments](#2-tenant-payments)
3. [Regulator Payouts](#3-regulator-payouts)
4. [Reconciliation](#4-reconciliation)

---

## 1. Overview

**Route:** `/payments`  
**Purpose:** Manage money flow between Tenant → Bumara → Regulator.

### 1.1 Money Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                         MONEY FLOW                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   TENANT                    BUMARA                    REGULATOR     │
│     │                         │                           │         │
│     │ ──── Payment ──────────►│                           │         │
│     │   (regulator fee +      │                           │         │
│     │    service fee)         │                           │         │
│     │                         │                           │         │
│     │                         │ ──── Payout ─────────────►│         │
│     │                         │   (regulator fee only)    │         │
│     │                         │                           │         │
│     │                   [Service fee retained]            │         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 Page Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│ Payments & Payouts                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐         │
│ │ Pending         │ │ Verified Today  │ │ Pending Payouts │         │
│ │ Verification    │ │                 │ │                 │         │
│ │     12          │ │     8           │ │     5           │         │
│ └─────────────────┘ └─────────────────┘ └─────────────────┘         │
│                                                                      │
│ [Tenant Payments] [Regulator Payouts] [Reconciliation]              │
│                                                                      │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ Table content based on selected tab                             │ │
│ │                                                                 │ │
│ └─────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Tenant Payments

### 2.1 Payment Request Status

```
NOT_REQUIRED (no payment needed)
       │
       ▼ (payment determined necessary)
REQUIRED_PENDING ◄─────────────────────────────┐
       │                                        │
       ▼ (tenant pays via gateway)              │
PAID_UNVERIFIED                                 │
       │                                        │
       ▼ (staff verifies with evidence)         │
PAID_VERIFIED ──────────────────────────────────┤ (refund requested)
       │                                        │
       └────────────────────────────────► REFUNDED
```

### 2.2 Payment Components

| Field | Description |
|-------|-------------|
| Regulator Fee | Amount payable to regulator |
| Service Fee | Bumara service charge |
| Total Amount | Regulator Fee + Service Fee |
| Currency | ZMW (Zambian Kwacha) |

### 2.3 Tenant Payments Table

| Column | Description |
|--------|-------------|
| Reference | Payment ID |
| Organization | Tenant name |
| Case | Linked filing/request |
| Regulator | ZRA, PACRA, etc. |
| Amount | Total in ZMW |
| Status | Current status badge |
| Paid At | When payment received |
| Actions | Verify button |

### 2.4 Payment Filters

| Filter | Options |
|--------|---------|
| Status | Pending / Verified / All |
| Organization | Search |
| Regulator | Multi-select |
| Date Range | Paid date |
| Amount Range | Min/Max |

### 2.5 Verification Workflow

**Pre-requisites:**
- Payment status is `PAID_UNVERIFIED`
- Gateway webhook has confirmed payment

**Verification Steps:**

1. Staff clicks "Verify" on payment row
2. Modal opens requiring:
   - Evidence document (required)
   - Verification note (optional)
3. Staff uploads/selects evidence
4. Staff clicks "Verify Payment"
5. System:
   - Updates status to `PAID_VERIFIED`
   - Links evidence document
   - Creates audit log
   - Updates case readiness gate

**Verification Modal:**

```
┌─────────────────────────────────────────────────────────────────────┐
│ Verify Payment                                                   ✕  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│ Organization: ABC Company Ltd                                       │
│ Amount: ZMW 5,000.00                                                │
│ Paid via: Mobile Money (Airtel)                                     │
│ Transaction ID: TXN-12345-ABCDE                                     │
│                                                                      │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │                                                                 │ │
│ │  📎 Drop evidence file here or click to upload                 │ │
│ │                                                                 │ │
│ │  Accepted: PDF, PNG, JPG (max 10MB)                            │ │
│ │                                                                 │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│ Note (optional):                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │                                                                 │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│                                         [Cancel] [Verify Payment]   │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.6 Evidence Requirements

Payment verification requires evidence document with tag:
- `PAYMENT_PROOF`

Acceptable evidence:
- Bank statement screenshot
- Mobile money confirmation
- Payment gateway confirmation
- Transaction receipt

### 2.7 API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `GET /payments/tenant` | GET | List tenant payments |
| `GET /payments/tenant/:id` | GET | Get payment detail |
| `POST /payments/tenant/:id/verify` | POST | Verify with evidence |

**Verify Request:**

```typescript
interface VerifyPaymentRequest {
  evidenceDocId: string;  // Required
  note?: string;
}
```

---

## 3. Regulator Payouts

### 3.1 Payout Status

```
NOT_REQUIRED (no payout needed)
       │
       ▼ (payout determined necessary)
QUEUED ◄──────────────────────────────────────┐
       │                                       │
       ▼ (staff initiates payment)             │
SENT_UNVERIFIED                                │
       │                                       │
       ▼ (evidence uploaded + approved)        │
PAID_VERIFIED ─────────────────────────► FAILED (if failed)
```

### 3.2 Regulator Payouts Table

| Column | Description |
|--------|-------------|
| Reference | Payout ID |
| Organization | Tenant name |
| Case | Linked filing/request |
| Regulator | ZRA, PACRA, etc. |
| Amount | Regulator fee amount |
| Status | Current status badge |
| Created | When payout created |
| Paid By | Staff who paid |
| Verified By | Manager who approved |
| Actions | Pay / Approve buttons |

### 3.3 Payout Filters

| Filter | Options |
|--------|---------|
| Status | Queued / Sent / Verified / All |
| Organization | Search |
| Regulator | Multi-select |
| Date Range | Created/Paid date |
| Needs Approval | Yes / No |

### 3.4 Create Payout Flow

**Pre-requisites:**
- Case requires regulator payment
- Tenant payment is `PAID_VERIFIED` (if required)

**Creation Steps:**

1. From case detail, click "Initiate Payout"
2. Modal shows:
   - Regulator name
   - Amount to pay
   - Payment method selection
3. Staff confirms details
4. System creates payout with status `QUEUED`

### 3.5 Pay Payout Flow

**Steps:**

1. Staff manually pays regulator (bank transfer, portal, office)
2. Staff clicks "Mark as Paid" on payout
3. Modal requires:
   - Payment reference (optional)
   - Evidence document (required)
4. Staff uploads evidence
5. System:
   - Updates status to `SENT_UNVERIFIED`
   - Links evidence document
   - Creates audit log

### 3.6 Approval Workflow

**Threshold-based approval:**

| Amount (ZMW) | Approval Required |
|--------------|-------------------|
| &lt; 5,000 | None |
| 5,000 - 50,000 | Manager |
| > 50,000 | Admin |

**Approval Steps:**

1. Payout marked as `SENT_UNVERIFIED`
2. If above threshold:
   - Manager sees in "Pending Approvals" queue
   - Manager reviews evidence
   - Manager clicks "Approve" or "Reject"
3. If approved:
   - Status → `PAID_VERIFIED`
   - Case gate updated
4. If rejected:
   - Status → `FAILED`
   - Reason logged
   - Staff notified

**Approval Modal:**

```
┌─────────────────────────────────────────────────────────────────────┐
│ Approve Payout                                                   ✕  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│ Organization: ABC Company Ltd                                       │
│ Regulator: ZRA                                                      │
│ Amount: ZMW 25,000.00                                               │
│ Paid By: John Smith on Jan 3, 2026                                  │
│                                                                      │
│ Evidence:                                                            │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ 📄 bank_transfer_proof.pdf                          [View]     │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│ Decision:                                                            │
│ ○ Approve - Confirm payment was made correctly                      │
│ ○ Reject - Payment has issues                                       │
│                                                                      │
│ Reason (required for rejection):                                     │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │                                                                 │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│                                              [Cancel] [Submit]      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.7 Evidence Requirements

Payout requires evidence document with tag:
- `PAYOUT_PROOF`

Acceptable evidence:
- Bank transfer confirmation
- Regulator payment receipt
- Reference number screenshot

### 3.8 Dual Control

For high-value payouts:

| Role | Responsibility |
|------|----------------|
| Analyst | Initiates and pays |
| Manager | Reviews and approves |

The same person cannot both pay and approve.

### 3.9 API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `GET /payments/payouts` | GET | List payouts |
| `POST /payments/payouts` | POST | Create payout |
| `GET /payments/payouts/:id` | GET | Get payout detail |
| `POST /payments/payouts/:id/pay` | POST | Mark as paid with evidence |
| `POST /payments/payouts/:id/approve` | POST | Approve/reject (manager+) |

**Create Payout Request:**

```typescript
interface CreatePayoutRequest {
  caseId: string;
  caseType: 'filing' | 'service_request';
  amount: number;  // In cents
  currency: 'ZMW';
}
```

**Pay Payout Request:**

```typescript
interface PayPayoutRequest {
  evidenceDocId: string;  // Required
  paymentReference?: string;
  note?: string;
}
```

**Approve Request:**

```typescript
interface ApprovePayoutRequest {
  decision: 'approve' | 'reject';
  reason?: string;  // Required if rejecting
}
```

---

## 4. Reconciliation

**Purpose:** Track and match money in vs money out.

### 4.1 Reconciliation View

| Section | Description |
|---------|-------------|
| Inflows | Tenant payments received |
| Outflows | Regulator payouts made |
| Net Position | Balance calculation |

### 4.2 Reconciliation Table

| Column | Description |
|--------|-------------|
| Date | Transaction date |
| Type | Inflow / Outflow |
| Reference | Payment/Payout ID |
| Organization | Tenant |
| Regulator | For outflows |
| Amount | Transaction amount |
| Balance | Running balance |

### 4.3 Export

- CSV export for accounting
- Date range selection
- Filter by organization/regulator

**MVP Status:** Basic view only. Advanced reconciliation features in phase 2.

---

## Audit Trail

All payment operations create audit events:

| Event | Description |
|-------|-------------|
| `payment_request_created` | Payment request generated |
| `payment_received` | Gateway webhook received |
| `payment_verified` | Staff verified payment |
| `payout_created` | Payout initiated |
| `payout_paid` | Staff marked as paid |
| `payout_approved` | Manager approved |
| `payout_rejected` | Manager rejected |

---

## Security Considerations

| Requirement | Implementation |
|-------------|----------------|
| Evidence required | Cannot verify/approve without document |
| Dual control | Payer ≠ Approver for high value |
| Audit logging | All actions logged |
| Role restrictions | Approval requires manager+ |
| Amount validation | Minor units stored, no floating point |

---

## File Locations

**Current Implementation:**

| Component | Location |
|-----------|----------|
| Payments page | `apps/backoffice/app/(authenticated)/(home)/(general)/payments/page.tsx` |
| Payment schema | `packages/database/src/schema/compliance/payment-requests.ts` |
| Payout schema | `packages/database/src/schema/compliance/regulator-payouts.ts` |

**To Create:**

| Component | Location |
|-----------|----------|
| API routes | `packages/backend/src/modules/compliance/payments.routes.ts` |
| Service | `packages/api-services/src/domains/compliance/payments.service.ts` |
| Verify modal | `apps/backoffice/components/payments/payment-verify-modal.tsx` |
| Approve modal | `apps/backoffice/components/payments/payout-approve-modal.tsx` |

---

## Related Documentation

- [Operations Module](/backoffice/modules/operations) - Cases & Tickets
- [Documents Module](/backoffice/modules/documents) - Evidence upload
- [Implementation Plan](/backoffice/implementation-plan) - Build steps

