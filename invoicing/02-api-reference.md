---
title: "Invoicing Module — API Reference"
description: "All endpoints require Authorization: Bearer &lt;token> and X-Organization-Id headers."
---

Base URL: `/api/v1`

All endpoints require `Authorization: Bearer <token>` and `X-Organization-Id` headers.

---

## Sales Invoices

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/invoicing/sales-invoices` | List invoices (filterable, paginated) |
| GET | `/invoicing/sales-invoices/{id}` | Get invoice by ID |
| GET | `/invoicing/sales-invoices/{id}/details` | Get invoice with line items |
| POST | `/invoicing/sales-invoices` | Create invoice with line items |
| PATCH | `/invoicing/sales-invoices/{id}` | Update invoice |
| DELETE | `/invoicing/sales-invoices/{id}` | Delete invoice (draft only) |
| POST | `/invoicing/sales-invoices/{id}/send` | Mark as sent |
| GET | `/invoicing/sales-invoices/{id}/pdf` | Download PDF |
| POST | `/invoicing/sales-invoices/{id}/pdf/send` | Email PDF to recipient |

## Estimates (Quotes)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/invoicing/estimates` | List estimates |
| GET | `/invoicing/estimates/{id}` | Get estimate |
| GET | `/invoicing/estimates/{id}/details` | Get with line items |
| POST | `/invoicing/estimates` | Create estimate |
| PATCH | `/invoicing/estimates/{id}` | Update estimate |
| DELETE | `/invoicing/estimates/{id}` | Delete estimate |
| POST | `/invoicing/estimates/{id}/send` | Mark as sent |
| POST | `/invoicing/estimates/{id}/convert` | Convert to sales invoice |
| GET | `/invoicing/estimates/{id}/pdf` | Download PDF |
| POST | `/invoicing/estimates/{id}/pdf/send` | Email PDF |

## Credit Notes

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/invoicing/credit-notes` | List credit notes |
| GET | `/invoicing/credit-notes/{id}` | Get credit note |
| GET | `/invoicing/credit-notes/{id}/details` | Get with line items |
| POST | `/invoicing/credit-notes` | Create credit note |
| PATCH | `/invoicing/credit-notes/{id}` | Update credit note |
| DELETE | `/invoicing/credit-notes/{id}` | Delete credit note |
| GET | `/invoicing/credit-notes/{id}/pdf` | Download PDF |
| POST | `/invoicing/credit-notes/{id}/pdf/send` | Email PDF |

## Debit Notes

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/invoicing/debit-notes` | List debit notes |
| GET | `/invoicing/debit-notes/{id}` | Get debit note |
| GET | `/invoicing/debit-notes/{id}/details` | Get with line items |
| POST | `/invoicing/debit-notes` | Create debit note |
| PATCH | `/invoicing/debit-notes/{id}` | Update debit note |
| DELETE | `/invoicing/debit-notes/{id}` | Delete debit note |
| GET | `/invoicing/debit-notes/{id}/pdf` | Download PDF |
| POST | `/invoicing/debit-notes/{id}/pdf/send` | Email PDF |

## Delivery Notes

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/invoicing/delivery-notes` | List delivery notes |
| GET | `/invoicing/delivery-notes/{id}` | Get delivery note |
| GET | `/invoicing/delivery-notes/{id}/details` | Get with line items |
| POST | `/invoicing/delivery-notes` | Create delivery note |
| PATCH | `/invoicing/delivery-notes/{id}` | Update delivery note |
| DELETE | `/invoicing/delivery-notes/{id}` | Delete delivery note |
| POST | `/invoicing/delivery-notes/from-invoice/{invoiceId}` | Create from invoice |
| GET | `/invoicing/delivery-notes/{id}/pdf` | Download PDF |
| POST | `/invoicing/delivery-notes/{id}/pdf/send` | Email PDF |

## Purchase Invoices

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/invoicing/purchase-invoices` | List purchase invoices |
| GET | `/invoicing/purchase-invoices/{id}` | Get purchase invoice |
| GET | `/invoicing/purchase-invoices/{id}/details` | Get with line items |
| POST | `/invoicing/purchase-invoices` | Create purchase invoice |
| PATCH | `/invoicing/purchase-invoices/{id}` | Update purchase invoice |
| DELETE | `/invoicing/purchase-invoices/{id}` | Delete purchase invoice |
| GET | `/invoicing/purchase-invoices/{id}/pdf` | Download PDF |
| POST | `/invoicing/purchase-invoices/{id}/pdf/send` | Email PDF |

## Purchase Orders

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/invoicing/purchase-orders` | List purchase orders |
| GET | `/invoicing/purchase-orders/{id}` | Get purchase order |
| GET | `/invoicing/purchase-orders/{id}/details` | Get with line items |
| POST | `/invoicing/purchase-orders` | Create purchase order |
| PATCH | `/invoicing/purchase-orders/{id}` | Update purchase order |
| DELETE | `/invoicing/purchase-orders/{id}` | Delete (draft only) |
| POST | `/invoicing/purchase-orders/{id}/convert` | Convert to purchase invoice |

## Payments

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/invoicing/payments` | List payments |
| GET | `/invoicing/payments/{id}` | Get payment |
| GET | `/invoicing/payments/{id}/allocations` | Get with allocations |
| POST | `/invoicing/payments` | Record payment |
| POST | `/invoicing/payments/{id}/allocate` | Allocate to invoices |
| POST | `/invoicing/payments/{id}/void` | Void payment |

## Customers

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/invoicing/customers` | List customers |
| GET | `/invoicing/customers/{id}` | Get customer |
| POST | `/invoicing/customers` | Create customer |
| PATCH | `/invoicing/customers/{id}` | Update customer |
| DELETE | `/invoicing/customers/{id}` | Delete customer |
| GET | `/invoicing/customers/{id}/statement` | Customer statement |
| GET | `/invoicing/customers/{id}/statement/pdf` | Statement PDF |
| GET | `/invoicing/customers/{id}/aging` | Aging report |

## Vendors

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/invoicing/vendors` | List vendors |
| GET | `/invoicing/vendors/{id}` | Get vendor |
| POST | `/invoicing/vendors` | Create vendor |
| PATCH | `/invoicing/vendors/{id}` | Update vendor |
| DELETE | `/invoicing/vendors/{id}` | Delete vendor |
| GET | `/invoicing/vendors/{id}/statement` | Vendor statement |
| GET | `/invoicing/vendors/{id}/statement/pdf` | Statement PDF |
| GET | `/invoicing/vendors/{id}/aging` | Aging report |

## Products & Tax Rates

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/invoicing/products` | List products |
| GET | `/invoicing/products/{id}` | Get product |
| POST | `/invoicing/products` | Create product |
| PATCH | `/invoicing/products/{id}` | Update product |
| DELETE | `/invoicing/products/{id}` | Delete product |
| GET | `/invoicing/tax-rates` | List tax rates |
| POST | `/invoicing/tax-rates` | Create tax rate |

## Reports

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/invoicing/reports/dashboard` | Dashboard KPIs |
| GET | `/invoicing/reports/aging` | Accounts receivable aging |
| GET | `/invoicing/reports/tax-summary` | Tax breakdown by period |
| GET | `/invoicing/reports/revenue` | Revenue by period |

## Settings

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/invoicing/settings` | Get organization settings |
| PUT | `/invoicing/settings` | Update settings |

## ZRA Smart Invoice

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/invoicing/zra/devices/register` | Register VSDC device |
| POST | `/invoicing/zra/devices/{id}/initialize` | Initialize with ZRA |
| GET | `/invoicing/zra/devices` | List devices |
| POST | `/invoicing/zra/transmit/{invoiceId}` | Transmit invoice to ZRA |
| GET | `/invoicing/zra/transmissions/{invoiceId}` | Transmission status |
| GET | `/invoicing/zra/transmissions/pending` | Pending transmissions |
| POST | `/invoicing/zra/transmissions/retry` | Retry all failed |
| POST | `/invoicing/zra/webhook/validation` | ZRA callback |
| POST | `/invoicing/zra/tax/calculate` | Calculate ZRA tax |
| GET | `/invoicing/zra/tax/categories` | Tax type codes |
| GET | `/invoicing/zra/reports/x` | X Report (interim) |
| GET | `/invoicing/zra/reports/z` | Z Report (end-of-day) |
| GET | `/invoicing/zra/reports/daily` | Daily summary |
| GET | `/invoicing/zra/reports/monthly` | Monthly summary |
| GET | `/invoicing/zra/health` | API health check |
| GET | `/invoicing/zra/statistics` | Transmission statistics |

---

## Common Query Parameters

All list endpoints support:
- `search` — free-text search
- `status` — filter by status
- `fromDate` / `toDate` — date range
- `limit` — page size (default: 20, max: 100)
- `page` — page number (default: 1)

## Email Send Body

```json
{ "recipientEmail": "customer@example.com" }
```

Field is optional — falls back to customer/vendor email on file.
