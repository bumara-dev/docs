---
title: "Invoicing Module — User Guide"
description: "The Invoicing module lets you manage your entire billing cycle: create quotes, send invoices, track payments, and stay compliant with Zambian tax..."
---

## What is this module?

The Invoicing module lets you manage your entire billing cycle: create quotes, send invoices, track payments, and stay compliant with Zambian tax regulations through ZRA Smart Invoice integration.

## Feature Overview

### Sales (Accounts Receivable)
- **Sales Invoices** — Create, send, and track invoices to your customers
- **Estimates/Quotes** — Prepare quotations that can be converted into invoices with one click
- **Credit Notes** — Issue refunds or adjustments against existing invoices
- **Delivery Notes** — Document goods dispatched to customers
- **Customer Management** — Maintain a customer directory with contact and billing details
- **Customer Statements** — Generate account statements showing all transactions and aging

### Purchases (Accounts Payable)
- **Purchase Invoices** — Record invoices received from vendors
- **Purchase Orders** — Create POs that convert to purchase invoices when goods arrive
- **Debit Notes** — Issue deductions or return claims against vendor invoices
- **Vendor Management** — Maintain a vendor directory with bank and tax details
- **Vendor Statements** — Generate account statements for vendor reconciliation

### Payments
- **Record Payments** — Log payments by cash, bank transfer, mobile money, cheque, or card
- **Payment Allocation** — Link payments to specific invoices (partial or full)
- **Void Payments** — Reverse incorrect payments with full audit trail

### Products & Pricing
- **Product Catalog** — Manage products and services with SKU, pricing, and tax rates
- **Tax Rates** — Configure tax rates used across documents

### Reports
- **Dashboard** — KPIs including total revenue, outstanding, overdue, and paid amounts
- **Aging Report** — Accounts receivable aging by customer (current, 30, 60, 90+ days)
- **Revenue Report** — Revenue trends by day, week, or month
- **Tax Summary** — Tax collected by rate over a given period

### PDF Documents
- **Three Templates** — Choose from Classic, Modern, or Minimal PDF designs
- **Customizable** — Toggle logo, tax columns, shipping address, payment instructions
- **Branding** — Set primary and accent colors that flow into all PDF documents
- **Download** — Download any document as a PDF
- **Email** — Send any document as a PDF attachment to customers or vendors

### ZRA Smart Invoice (Tax Compliance)
- **Device Registration** — Register and initialize VSDC devices with ZRA
- **Invoice Transmission** — Submit sales invoices to ZRA with proper tax codes
- **Retry & Monitoring** — Track transmission status, retry failures automatically
- **Compliance Reports** — Generate X, Z, daily, and monthly reports

---

## How-To Guides

### Create and Send an Invoice

1. Navigate to **Sales Invoices** from the sidebar
2. Click **Create Invoice**
3. Select a customer (or create a new one)
4. Add line items by selecting products or typing descriptions
5. Set quantities, prices, and tax types for each item
6. Review totals — subtotal, discount, tax, and grand total are calculated automatically
7. Click **Create Invoice** (saves as draft)
8. From the invoice detail page:
   - Click **Send** to mark as sent
   - Click **Download PDF** for a local copy
   - Click **Email PDF** to send directly to the customer
   - Click **Transmit to ZRA** for tax compliance (if ZRA is configured)

### Create a Quote and Convert to Invoice

1. Navigate to **Quotes** → **New Quote**
2. Fill in customer, line items, and expiry date
3. Save the quote
4. Click **Send** to share with the customer
5. Once accepted, click **Convert to Invoice** — a new sales invoice is created with all the quote data pre-filled

### Record a Payment

Option A — From the invoice detail page:
1. Open the invoice
2. Click **Record Payment**
3. Enter amount, method, date, and reference number
4. Submit — the invoice status updates automatically

Option B — From the Payments section:
1. Navigate to **Payments** → **New Payment**
2. Select customer or vendor
3. Enter payment details
4. Allocate to one or more invoices
5. Save

### Set Up PDF Templates

1. Go to **Settings** → **Invoice Template**
2. Choose a template style: Classic, Modern, or Minimal
3. Toggle visible sections (logo, tax columns, shipping, payment info)
4. Set primary and accent colors
5. Add default footer text and terms & conditions
6. Click **Save** — all future PDFs will use these settings

### Set Up ZRA Smart Invoice

1. Go to **Settings** → **ZRA Smart Invoice**
2. Click **Register Device** and enter your VSDC serial number
3. Click **Initialize** on the new device
4. Enter your TPIN (10 characters), Branch ID (3 characters), and select environment
5. Click **Initialize with ZRA** — the device connects to ZRA and receives an API key
6. Once active, you can transmit invoices from any invoice detail page using the **Transmit to ZRA** button

---

## Navigation Map

```
Invoicing
├── Dashboard ─────────── Overview with KPIs and charts
├── Sales
│   ├── Invoices ──────── Create, view, manage sales invoices
│   ├── Quotes ────────── Estimates that convert to invoices
│   ├── Credit Notes ──── Customer returns and adjustments
│   ├── Delivery Notes ── Goods dispatch documentation
│   └── Customers ─────── Customer directory and statements
├── Purchases
│   ├── Purchase Invoices  Vendor bills
│   ├── Purchase Orders ── Requisitions that convert to invoices
│   ├── Debit Notes ────── Vendor deductions
│   └── Vendors ────────── Vendor directory and statements
├── Payments ──────────── Record and allocate payments
├── Products ──────────── Product and service catalog
├── Recurring ─────────── Automated invoice templates
├── Reports
│   ├── Aging Report ──── AR aging by customer
│   ├── Revenue Report ── Revenue trends
│   └── Tax Summary ──── Tax breakdown by period
└── Settings
    ├── General ────────── Company info, defaults, bank details
    ├── Invoice Template ─ PDF design, colors, toggles
    └── ZRA Smart Invoice  Device management, transmissions
```

## Document Statuses

| Status | Meaning |
|--------|---------|
| Draft | Not yet finalized — can be edited or deleted |
| Sent | Delivered to customer/vendor |
| Viewed | Customer has opened the invoice |
| Partially Paid | Some payment received |
| Paid | Fully settled |
| Overdue | Past due date, not fully paid |
| Void | Cancelled — no longer valid |

## Payment Methods Supported

- Cash
- Bank Transfer
- Mobile Money (MTN, Airtel, Zamtel)
- Cheque
- Credit/Debit Card
