---
title: "Invoicing Module — Diagrams"
---

## 1. Invoice Lifecycle Flow

```
                    ┌──────────┐
                    │  START   │
                    └────┬─────┘
                         │
                         ▼
                  ┌──────────────┐
                  │ Create Draft │
                  │   Invoice    │
                  └──────┬───────┘
                         │
              ┌──────────┼──────────┐
              │          │          │
              ▼          ▼          ▼
        ┌──────────┐ ┌────────┐ ┌──────┐
        │  Edit    │ │ Delete │ │ Send │
        │  Draft   │ │ Draft  │ │      │
        └──────────┘ └────────┘ └──┬───┘
                                   │
                         ┌─────────┼─────────┐
                         │         │         │
                         ▼         ▼         ▼
                   ┌──────────┐ ┌─────┐ ┌────────────┐
                   │ Download │ │Email│ │ Transmit   │
                   │   PDF    │ │ PDF │ │  to ZRA    │
                   └──────────┘ └─────┘ └────────────┘
                                   │
                         ┌─────────┼──────────┐
                         │         │          │
                         ▼         ▼          ▼
                   ┌──────────┐ ┌──────┐ ┌───────────┐
                   │  Record  │ │Overdue│ │  Void     │
                   │ Payment  │ │(auto) │ │  Invoice  │
                   └────┬─────┘ └──────┘ └───────────┘
                        │
              ┌─────────┴─────────┐
              │                   │
              ▼                   ▼
       ┌──────────────┐    ┌──────────┐
       │  Partially   │    │   Paid   │
       │    Paid      │───▶│ (closed) │
       └──────────────┘    └──────────┘
```

## 2. Quote-to-Invoice Conversion Flow

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Create    │     │    Send     │     │  Customer   │
│   Quote     │────▶│   Quote     │────▶│  Accepts    │
│  (Draft)    │     │  (Sent)     │     │             │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                               │
                                               ▼
                    ┌─────────────────────────────────┐
                    │    Convert to Sales Invoice      │
                    │  - Copies customer, line items   │
                    │  - Sets quote status: converted  │
                    │  - Creates draft invoice         │
                    └────────────────┬────────────────┘
                                     │
                                     ▼
                              ┌──────────────┐
                              │   Invoice    │
                              │   (Draft)    │
                              └──────────────┘
```

## 3. Purchase Order Flow

```
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│  Create PO    │     │  Issue to     │     │   Goods       │
│  (Draft)      │────▶│   Vendor      │────▶│  Received     │
└───────────────┘     └───────────────┘     └───────┬───────┘
                                                    │
                                                    ▼
                              ┌─────────────────────────────────┐
                              │  Convert to Purchase Invoice     │
                              │  - Copies vendor, items, prices  │
                              │  - Sets PO status: invoiced      │
                              │  - Creates purchase invoice      │
                              └──────────────┬──────────────────┘
                                             │
                                             ▼
                                    ┌────────────────┐
                                    │   Purchase     │
                                    │   Invoice      │
                                    │   (Draft)      │
                                    └────────────────┘
```

## 4. Payment Allocation Flow

```
┌────────────────┐
│ Record Payment │
│ Amount: K5,000 │
└───────┬────────┘
        │
        ▼
┌──────────────────────────────────────┐
│         Allocate to Invoices         │
│                                      │
│  ┌──────────┐  ┌──────────┐         │
│  │ INV-001  │  │ INV-002  │         │
│  │ Due:3000 │  │ Due:4000 │         │
│  │ Pay:3000 │  │ Pay:2000 │         │
│  │ → PAID   │  │ → PARTIAL│         │
│  └──────────┘  └──────────┘         │
│                                      │
│  Total Allocated: K5,000             │
│  Unallocated: K0                     │
└──────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────┐
│         Ledger Updated               │
│  - Debtor balance decreased          │
│  - Invoice amountPaid updated        │
│  - Invoice amountDue recalculated    │
│  - Invoice status auto-updated       │
└──────────────────────────────────────┘
```

## 5. Credit Note / Debit Note Flow

```
┌─────────────────────────────────────────────────────────┐
│                  CREDIT NOTE (Customer)                  │
│                                                         │
│  Customer returns goods / pricing error found            │
│                                                         │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐       │
│  │  Create   │───▶│   Issue   │───▶│  Apply to │       │
│  │  (Draft)  │    │  to Cust  │    │  Invoice  │       │
│  └───────────┘    └───────────┘    └───────────┘       │
│                                          │              │
│         ┌────────────────────────────────┘              │
│         ▼                                               │
│  - Reduces customer balance (debtor ledger credit)      │
│  - Receives returned goods into inventory               │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                  DEBIT NOTE (Vendor)                     │
│                                                         │
│  Return goods to vendor / dispute vendor charge          │
│                                                         │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐       │
│  │  Create   │───▶│   Issue   │───▶│  Apply to │       │
│  │  (Draft)  │    │ to Vendor │    │  Purchase  │       │
│  └───────────┘    └───────────┘    └───────────┘       │
│                                          │              │
│         ┌────────────────────────────────┘              │
│         ▼                                               │
│  - Reduces vendor balance (creditor ledger credit)      │
│  - Deducts returned goods from inventory                │
└─────────────────────────────────────────────────────────┘
```

## 6. PDF Generation & Email Flow

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   User       │     │   Service    │     │   Template   │
│ clicks PDF   │────▶│ fetches data │────▶│  selection   │
│ or Email     │     │ + settings   │     │ (classic/    │
└──────────────┘     └──────────────┘     │ modern/min)  │
                                          └──────┬───────┘
                                                 │
                                                 ▼
                          ┌────────────────────────────────┐
                          │      HTML Generation           │
                          │                                │
                          │  Settings applied:             │
                          │  ✓ Logo (if showLogo=true)     │
                          │  ✓ Primary/accent colors       │
                          │  ✓ Company info from org       │
                          │  ✓ Tax columns (if shown)      │
                          │  ✓ Shipping address (if shown) │
                          │  ✓ Payment instructions        │
                          │  ✓ Footer text                 │
                          │  ✓ Terms & conditions          │
                          └────────────┬───────────────────┘
                                       │
                              ┌────────┴────────┐
                              │                 │
                              ▼                 ▼
                       ┌──────────┐      ┌──────────────┐
                       │  HTML →  │      │   sendEmail   │
                       │   PDF    │      │  via Resend   │
                       │ (buffer) │      │ + attachment  │
                       └────┬─────┘      └──────────────┘
                            │
                            ▼
                     ┌──────────────┐
                     │  Download    │
                     │  or Return   │
                     └──────────────┘
```

## 7. ZRA Smart Invoice Transmission Flow

```
┌──────────────────────────────────────────────────────────────┐
│                     DEVICE SETUP (one-time)                   │
│                                                              │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐         │
│  │ Register   │───▶│ Initialize │───▶│  Device    │         │
│  │ Device     │    │ with ZRA   │    │  Active    │         │
│  │ (serial#)  │    │ (TPIN+BHF) │    │  (apiKey)  │         │
│  └────────────┘    └────────────┘    └────────────┘         │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                  INVOICE TRANSMISSION                         │
│                                                              │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐         │
│  │ User clicks│    │ Build ZRA  │    │  Send to   │         │
│  │ "Transmit  │───▶│ payload    │───▶│  ZRA API   │         │
│  │  to ZRA"   │    │ (wire fmt) │    │            │         │
│  └────────────┘    └────────────┘    └─────┬──────┘         │
│                                            │                 │
│                                  ┌─────────┼─────────┐      │
│                                  │         │         │      │
│                                  ▼         ▼         ▼      │
│                           ┌──────────┐ ┌────────┐ ┌──────┐  │
│                           │Transmitted│ │ Failed │ │Retry │  │
│                           │(pending)  │ │        │ │(auto)│  │
│                           └────┬─────┘ └────┬───┘ └──┬───┘  │
│                                │            │        │      │
│                                ▼            ▼        │      │
│                          ┌──────────┐ ┌──────────┐   │      │
│                          │Validated │ │ Rejected │   │      │
│                          │(receipt#)│ │ (reason) │◀──┘      │
│                          └──────────┘ └──────────┘          │
└──────────────────────────────────────────────────────────────┘

ZRA Payload (wire format):
┌─────────────────────────────────────┐
│  tpin, bhfId (TPIN + Branch)        │
│  cisInvcNo (invoice number)         │
│  custTpin (customer TPIN)           │
│  rcptTyCd: "S" (Sales)             │
│  pmtTyCd: "01" (Cash)              │
│  salesSttsCd: "02" (Approved)      │
│  salesDt: "20250518" (YYYYMMDD)    │
│  totTaxblAmt, totTaxAmt, totAmt    │
│  taxblAmtA..E, taxAmtA..E          │
│  itemList: [{                       │
│    itemSeq, itemNm,                 │
│    qty, prc, splyAmt,              │
│    taxTyCd, taxblAmt, taxAmt       │
│  }]                                 │
└─────────────────────────────────────┘
```

## 8. Inventory Integration Flow

```
┌─────────────────────────────────────────────────────┐
│          STOCK MOVEMENT TRIGGERS                     │
│                                                     │
│  Sales Invoice (sent/paid)                          │
│    → deductInventoryForInvoice()                    │
│    → Decreases stock at specified location          │
│                                                     │
│  Purchase Invoice (paid)                            │
│    → receiveInventoryForPurchaseInvoice()           │
│    → Increases stock at specified location          │
│                                                     │
│  Credit Note (issued)                               │
│    → receiveInventoryForCreditNote()                │
│    → Returns goods to stock (customer return)       │
│                                                     │
│  Debit Note (issued)                                │
│    → deductInventoryForDebitNote()                  │
│    → Returns goods to vendor (vendor return)        │
└─────────────────────────────────────────────────────┘
```

## 9. Notification Events

```
┌──────────────────────────────────────────────────────┐
│              INVOICING NOTIFICATION EVENTS            │
│                                                      │
│  Event Type              │ Category  │ Trigger       │
│  ────────────────────────┼───────────┼────────────── │
│  INVOICE_SENT            │ INVOICING │ Email sent    │
│  INVOICE_PAID            │ INVOICING │ Full payment  │
│  INVOICE_OVERDUE         │ INVOICING │ Past due      │
│  INVOICE_REMINDER        │ INVOICING │ Reminder sent │
│  CREDIT_NOTE_SENT        │ INVOICING │ CN emailed    │
│  DEBIT_NOTE_SENT         │ INVOICING │ DN emailed    │
│  DELIVERY_NOTE_SENT      │ INVOICING │ DN emailed    │
│  PURCHASE_INVOICE_SENT   │ INVOICING │ PI emailed    │
│  ESTIMATE_SENT           │ INVOICING │ Quote emailed │
│                                                      │
│  Email delivery via Resend adapter with:             │
│  - PDF attachment                                    │
│  - Custom HTML body with document details            │
│  - Tracking metadata (deliveryId, notificationId)    │
└──────────────────────────────────────────────────────┘
```

## 10. Frontend Component Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     PAGE LAYER                               │
│  app/(authenticated)/(modules)/invoicing/                    │
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐     │
│  │  List Page    │ │  Detail Page  │ │  Form Page    │     │
│  │  (table +     │ │  (info cards  │ │  (create/edit │     │
│  │   filters)    │ │  + actions)   │ │   with items) │     │
│  └───────┬───────┘ └───────┬───────┘ └───────┬───────┘     │
└──────────┼─────────────────┼─────────────────┼──────────────┘
           │                 │                 │
┌──────────┼─────────────────┼─────────────────┼──────────────┐
│          ▼                 ▼                 ▼  COMPONENT   │
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐     │
│  │ entity-list   │ │ entity-detail │ │ entity-form   │     │
│  └───────────────┘ └───────────────┘ └───────────────┘     │
│                                                             │
│  Shared Components:                                         │
│  ┌────────────────┐ ┌──────────────┐ ┌─────────────────┐   │
│  │ LineItemsEditor│ │CustomerSelect│ │RecordPaymentDlg │   │
│  │ (products,     │ │VendorSelect  │ │SendEmailDialog  │   │
│  │  qty, price,   │ │ProductSelect │ │StatusBadge      │   │
│  │  tax, totals)  │ │              │ │InvoicePreview   │   │
│  └────────────────┘ └──────────────┘ └─────────────────┘   │
└─────────────────────────────────────────────────────────────┘
           │                 │                 │
┌──────────┼─────────────────┼─────────────────┼──────────────┐
│          ▼                 ▼                 ▼  DATA LAYER  │
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐     │
│  │    Hooks      │ │   Fetchers    │ │    Types      │     │
│  │ (TanStack     │ │  (fetch +     │ │ (TypeScript   │     │
│  │  Query)       │ │  auth headers)│ │  interfaces)  │     │
│  └───────────────┘ └───────────────┘ └───────────────┘     │
└─────────────────────────────────────────────────────────────┘
```
