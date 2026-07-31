---
title: "Invoicing Module — Data Model"
description: "Entity relationships and core table definitions behind invoices, customers, vendors, payments, and credit notes."
---

## Entity Relationship Diagram

```
┌─────────────────┐       ┌──────────────────┐       ┌─────────────────┐
│  organizations  │───┐   │    customers      │       │    vendors      │
│  (tenant root)  │   │   │  customerNumber   │       │  vendorNumber   │
└─────────────────┘   │   │  name, tpin       │       │  name, tpin     │
                      │   │  email, phone     │       │  email, phone   │
                      │   │  billing/shipping │       │  address        │
                      │   │  creditLimit      │       │  bankDetails    │
                      │   │  totalOwed/Paid   │       │  totalOwed/Paid │
                      │   └────────┬──────────┘       └────────┬────────┘
                      │            │                           │
                      │   ┌────────┴───────────────────────────┴────────┐
                      │   │                                             │
                      │   ▼                                             ▼
┌─────────────────┐   │  ┌──────────────────┐       ┌──────────────────┐
│    products     │   │  │  sales_invoices   │       │purchase_invoices │
│  name, sku      │   │  │  invoiceNumber    │       │  referenceNumber │
│  unitPrice      │   │  │  invoiceType      │       │  vendorInvNo     │
│  taxRateId ─────┼───┤  │  status           │       │  status          │
└─────────────────┘   │  │  currency         │       │  whtAmount       │
                      │  │  subtotal/tax/    │       │  subtotal/tax/   │
┌─────────────────┐   │  │  total/paid/due   │       │  total/paid/due  │
│   tax_rates     │   │  │  zraInvoiceId ────┼──┐    └────────┬─────────┘
│  name, rate     │   │  └────────┬──────────┘  │             │
│  isDefault      │   │           │             │             │
└─────────────────┘   │           ▼             │             ▼
                      │  ┌──────────────────┐   │    ┌──────────────────┐
                      │  │ invoice_line_items│   │    │(same line_items) │
                      │  │  description     │   │    │  lineItemType=   │
                      │  │  qty, unitPrice  │   │    │  "purchase"      │
                      │  │  taxType (A-D)   │   │    └──────────────────┘
                      │  │  taxAmount       │   │
                      │  │  totalAmount     │   │
                      │  └──────────────────┘   │
                      │                         │
                      │  ┌──────────────────┐   │    ┌──────────────────┐
                      │  │    estimates      │   │    │  delivery_notes  │
                      │  │  estimateNumber   │   │    │  deliveryNoteNo  │
                      │  │  status           │   │    │  deliveryDate    │
                      │  │  expiryDate       │   │    │  status          │
                      │  │  convertedInvId ──┼───┤    │  deliveredBy     │
                      │  └──────────────────┘   │    │  receivedBy      │
                      │                         │    └──────────────────┘
                      │  ┌──────────────────┐   │
                      │  │   credit_notes   │   │    ┌──────────────────┐
                      │  │  creditNoteNo    │   │    │   debit_notes    │
                      │  │  reason          │   │    │  debitNoteNo     │
                      │  │  invoiceId (FK)  │   │    │  reason          │
                      │  └──────────────────┘   │    │  invoiceId (FK)  │
                      │                         │    └──────────────────┘
                      │  ┌──────────────────┐   │
                      │  │    payments       │   │    ┌──────────────────┐
                      │  │  paymentNumber    │   │    │payment_allocations│
                      │  │  paymentMethod    │   │    │  paymentId (FK)  │
                      │  │  amount, currency │   │    │  invoiceId (FK)  │
                      │  │  allocatedAmount  │   │    │  allocatedAmount │
                      │  └────────┬──────────┘   │    └──────────────────┘
                      │           │              │
                      │           ▼              │
                      │  ┌──────────────────┐    │
                      │  │account_transactions│   │
                      │  │  accountType      │   │
                      │  │  debit/credit     │   │
                      │  │  runningBalance   │   │
                      │  └──────────────────┘   │
                      │                         │
                      │  ┌──────────────────┐   │
                      ├──│ invoice_settings  │   │
                      │  │  pdfTemplateName  │   │
                      │  │  showLogo         │   │
                      │  │  primaryColor     │   │
                      │  │  bankDetails      │   │
                      │  └──────────────────┘   │
                      │                         │
                      │  ┌──────────────────┐   │    ┌──────────────────┐
                      └──│ zra_vsdc_devices  │   └───▶│zra_transmissions │
                         │  deviceSerial     │        │  transmitStatus  │
                         │  tpin, branchId   │        │  qrCode          │
                         │  apiKey           │        │  receiptNumber   │
                         │  isInitialized    │        │  retryCount      │
                         └──────────────────┘        └──────────────────┘
```

## Core Tables

### Transaction Documents
| Table | Purpose | Key Fields |
|-------|---------|------------|
| `sales_invoices` | Invoices to customers | invoiceNumber, invoiceType (standard/export/lpo), status, amounts |
| `purchase_invoices` | Invoices from vendors | referenceNumber, vendorInvoiceNumber, whtAmount |
| `estimates` | Quotes/proposals | estimateNumber, expiryDate, convertedInvoiceId |
| `credit_notes` | Customer returns | creditNoteNumber, reason, linked invoiceId |
| `debit_notes` | Vendor deductions | debitNoteNumber, reason, linked invoiceId |
| `delivery_notes` | Goods delivery | deliveryNoteNumber, deliveredBy, receivedBy, trackingNumber |
| `purchase_orders` | Purchase requisitions | poNumber, expectedDeliveryDate |
| `recurring_invoice_templates` | Auto-generation rules | frequency, nextDueDate, lineItemsJson |

### Line Items
| Table | Purpose |
|-------|---------|
| `invoice_line_items` | Shared line items for sales/purchase invoices, delivery notes |
| `estimate_line_items` | Line items for estimates |
| `credit_note_line_items` | Line items for credit notes |
| `debit_note_line_items` | Line items for debit notes |

### Financial
| Table | Purpose |
|-------|---------|
| `payments` | Payment records (cash, bank, mobile money) |
| `payment_allocations` | Links payments to specific invoices |
| `account_transactions` | Ledger entries (debtor/creditor) with running balance |

### Master Data
| Table | Purpose |
|-------|---------|
| `customers` | Customer directory with billing/shipping addresses |
| `vendors` | Vendor directory with bank details |
| `products` | Product/service catalog with pricing |
| `tax_rates` | Tax rate configurations |

### Configuration
| Table | Purpose |
|-------|---------|
| `invoice_settings` | Per-org settings (branding, templates, payment details) |
| `invoice_sequences` | Auto-incrementing number sequences per document type |

### ZRA Integration
| Table | Purpose |
|-------|---------|
| `zra_vsdc_devices` | Registered VSDC devices (serial, API key, TPIN) |
| `zra_smart_invoice_transmissions` | Transmission log (status, receipt, retry tracking) |

## Invoice Status Lifecycle

```
draft → sent → viewed → partially_paid → paid
  │               │                        │
  └→ void         └→ overdue ──────────────┘
```

## Tax Type Codes (ZRA)

| Code | Name | Rate |
|------|------|------|
| A | Standard Rated | 16% |
| B | Minimum Taxable Value | 16% |
| C1 | Exports | 0% |
| C2 | Zero-rated LPO | 0% |
| C3 | Zero-rated by nature | 0% |
| D | Exempt | 0% |
| TOT | Turnover Tax | 0% |
