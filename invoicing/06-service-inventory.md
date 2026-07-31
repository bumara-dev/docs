---
title: "Invoicing Module — Service & Component Inventory"
---

## Backend Services

| Service File | Functions | Description |
|---|---|---|
| `invoices.service.ts` | getInvoice, getInvoiceWithLineItems, listInvoices, createInvoice, updateInvoice, deleteInvoice | Sales invoice CRUD + ledger entries + inventory link |
| | getPurchaseInvoice, getPurchaseInvoiceWithLineItems, listPurchaseInvoices, createPurchaseInvoice, updatePurchaseInvoice, deletePurchaseInvoice | Purchase invoice CRUD |
| | getCreditNote, getCreditNoteWithLineItems, listCreditNotes, createCreditNote, updateCreditNote | Credit note CRUD |
| | getDebitNote, getDebitNoteWithLineItems, listDebitNotes, createDebitNote, updateDebitNote | Debit note CRUD |
| `estimates.service.ts` | getEstimate, getEstimateWithLineItems, listEstimates, createEstimate, updateEstimate, deleteEstimate, sendEstimate, convertEstimateToInvoice | Quote lifecycle |
| `delivery-notes.service.ts` | getDeliveryNote, getDeliveryNoteWithLineItems, listDeliveryNotes, createDeliveryNote, updateDeliveryNote, deleteDeliveryNote, createDeliveryNoteFromInvoice | Delivery management |
| `purchase-orders.service.ts` | getPurchaseOrder, getPurchaseOrderWithLineItems, listPurchaseOrders, createPurchaseOrder, updatePurchaseOrder, deletePurchaseOrder, convertPurchaseOrderToInvoice | PO lifecycle |
| `payments.service.ts` | getPayment, getPaymentWithAllocations, listPayments, recordPayment, reversePayment | Payment processing |
| `customers.service.ts` | getCustomer, listCustomers, createCustomer, updateCustomer, deleteCustomer, getCustomerStatement, getCustomerAging | Customer management |
| `vendors.service.ts` | getVendor, listVendors, createVendor, updateVendor, deleteVendor, getVendorStatement, getVendorAging | Vendor management |
| `products.service.ts` | getProduct, listProducts, createProduct, updateProduct, deleteProduct + tax rate CRUD | Product catalog |
| `recurring.service.ts` | getRecurringTemplate, listRecurringTemplates, createRecurringTemplate, updateRecurringTemplate, pauseRecurringTemplate, deleteRecurringTemplate | Recurring invoices |
| `reports.service.ts` | getAgingReport, getTaxSummary, getRevenueByPeriod, getDashboardStats | Analytics |
| `settings.service.ts` | getInvoiceSettings, updateInvoiceSettings | Organization config |
| `account-transactions.service.ts` | getAccountTransaction, listAccountTransactions, createAccountTransaction, getAccountBalance | Ledger |
| `inventory-link.service.ts` | deductInventoryForInvoice, receiveInventoryForPurchaseInvoice, receiveInventoryForCreditNote, deductInventoryForDebitNote | Stock movements |

## PDF Services

| Service File | Functions | Event Type |
|---|---|---|
| `invoice-pdf.service.ts` | generateInvoicePdf, sendInvoicePdfEmail | INVOICE_SENT |
| `credit-note-pdf.service.ts` | generateCreditNotePdf, sendCreditNotePdfEmail | CREDIT_NOTE_SENT |
| `debit-note-pdf.service.ts` | generateDebitNotePdf, sendDebitNotePdfEmail | DEBIT_NOTE_SENT |
| `delivery-note-pdf.service.ts` | generateDeliveryNotePdf, sendDeliveryNotePdfEmail | DELIVERY_NOTE_SENT |
| `purchase-invoice-pdf.service.ts` | generatePurchaseInvoicePdf, sendPurchaseInvoicePdfEmail | PURCHASE_INVOICE_SENT |
| `estimate-pdf.service.ts` | generateEstimatePdf, sendEstimatePdfEmail | ESTIMATE_SENT |
| `pdf-settings.ts` | fetchDocumentSettings, sendDocumentEmail | Shared helpers |

## ZRA Services

| Service File | Functions | Description |
|---|---|---|
| `zra-smart-invoice.service.ts` | registerVsdcDevice, initializeVsdcDevice, getVsdcDevices, activateVsdcDevice | Device management |
| | transmitInvoiceToZra, getTransmissionStatus, listPendingTransmissions, retryFailedTransmissions | Invoice transmission |
| | handleZraValidationWebhook | Webhook handling |
| | zraHealthCheck, getZraStatistics | Monitoring |
| `zra/zra-api-client.ts` | ZraApiClient class (initializeDevice, sendSalesData, sendPurchaseData, sendStockData, healthCheck) | HTTP client with wire format transformation |
| `zra/zra-tax.service.ts` | calculateItemTax, calculateInvoiceTax, getTaxCategories, getTaxRateForCode | Tax calculation engine |
| `zra/zra-report.service.ts` | generateXReport, generateZReport, generateDailyReport, generateMonthlyReport | Compliance reporting |

## Frontend Components

### Document Components (per entity: list, detail, form)

| Entity | List | Detail | Form |
|---|---|---|---|
| Sales Invoices | sales-invoices-list | sales-invoice-detail | sales-invoice-form |
| Purchase Invoices | purchase-invoices-list | purchase-invoice-detail | purchase-invoice-form |
| Estimates | estimates-list | estimate-detail | estimate-form |
| Credit Notes | credit-notes-list | credit-note-detail | credit-note-form |
| Debit Notes | debit-notes-list | debit-note-detail | debit-note-form |
| Delivery Notes | delivery-notes-list | delivery-note-detail | delivery-note-form |
| Purchase Orders | purchase-orders-list | purchase-order-detail | purchase-order-form |
| Payments | payments-list | payment-detail | payment-form |
| Customers | customers-list | customer-detail | customer-form |
| Vendors | vendors-list | vendor-detail | vendor-form |
| Products | products-list | product-detail | product-form |
| Recurring | recurring-list | recurring-detail | recurring-form |

### Shared Components

| Component | Purpose |
|---|---|
| `line-items-editor.tsx` | Inline line item editor with product selection, qty, price, tax |
| `customer-select.tsx` | Customer dropdown with search |
| `vendor-select.tsx` | Vendor dropdown with search |
| `product-select.tsx` | Product dropdown with search |
| `record-payment-dialog.tsx` | Modal for recording payment against an invoice |
| `send-email-dialog.tsx` | Modal with email input for sending document PDFs |
| `status-badge.tsx` | Color-coded status indicator |
| `invoice-preview.tsx` | Document preview modal |

### Settings Components

| Component | Purpose |
|---|---|
| `invoice-settings-form.tsx` | Company info, defaults, bank details, mobile money |
| `invoice-template-settings.tsx` | PDF template picker (Classic/Modern/Minimal), section toggles, branding colors, default content |

### ZRA Components

| Component | Purpose |
|---|---|
| `zra-settings.tsx` | ZRA dashboard with health, stats, retry actions |
| `devices-list.tsx` | VSDC device table with register + initialize dialogs |
| `pending-transmissions.tsx` | Transmission queue with transmit/retry actions |

### Report Components

| Component | Purpose |
|---|---|
| `aging-report.tsx` | AR aging analysis by customer |
| `revenue-report.tsx` | Revenue trend chart |
| `tax-summary-report.tsx` | Tax breakdown by rate |
| `invoicing-dashboard.tsx` | KPI overview |

## Frontend Hooks

| Hook File | Key Hooks | API Base |
|---|---|---|
| `use-sales-invoices.ts` | useSalesInvoices, useSalesInvoiceDetails, useCreateSalesInvoice, useSendSalesInvoice | /invoicing/sales-invoices |
| `use-purchase-invoices.ts` | usePurchaseInvoices, usePurchaseInvoiceDetails, useCreatePurchaseInvoice | /invoicing/purchase-invoices |
| `use-estimates.ts` | useEstimates, useEstimateDetails, useSendEstimate, useConvertEstimate | /invoicing/estimates |
| `use-credit-notes.ts` | useCreditNotes, useCreditNoteDetails, useCreateCreditNote | /invoicing/credit-notes |
| `use-debit-notes.ts` | useDebitNotes, useDebitNoteDetails, useCreateDebitNote | /invoicing/debit-notes |
| `use-delivery-notes.ts` | useDeliveryNotes, useDeliveryNoteDetails, useCreateDeliveryNoteFromInvoice | /invoicing/delivery-notes |
| `use-purchase-orders.ts` | usePurchaseOrders, usePurchaseOrderDetails, useConvertPurchaseOrderToInvoice | /invoicing/purchase-orders |
| `use-payments.ts` | usePayments, usePaymentAllocations, useCreatePayment, useVoidPayment | /invoicing/payments |
| `use-customers.ts` | useCustomers, useCreateCustomer | /invoicing/customers |
| `use-vendors.ts` | useVendors, useCreateVendor | /invoicing/vendors |
| `use-products.ts` | useProducts, useCreateProduct | /invoicing/products |
| `use-recurring.ts` | useRecurringTemplates, useCreateRecurringTemplate, usePauseRecurringTemplate | /invoicing/recurring |
| `use-reports.ts` | useDashboardStats, useAgingReport, useTaxSummary, useRevenueReport | /invoicing/reports |
| `use-settings.ts` | useInvoiceSettings, useUpdateInvoiceSettings | /invoicing/settings |
| `use-zra.ts` | useVsdcDevices, useInitializeVsdcDevice, useTransmitInvoice, useZraHealth, useZraStatistics | /invoicing/zra |
| `use-statements.ts` | useCustomerStatement, useVendorStatement | /invoicing/statements |

## Route Counts Summary

| Category | Routes | Pages | Components |
|---|---|---|---|
| Sales Invoices | 9 | 4 | 3 |
| Estimates | 10 | 4 | 3 |
| Credit Notes | 8 | 4 | 3 |
| Debit Notes | 8 | 4 | 3 |
| Delivery Notes | 9 | 4 | 3 |
| Purchase Invoices | 8 | 4 | 3 |
| Purchase Orders | 7 | 4 | 3 |
| Payments | 6 | 3 | 4 |
| Customers | 8 | 4 | 3 |
| Vendors | 8 | 4 | 3 |
| Products | 5 | 4 | 3 |
| Recurring | 5+ | 4 | 3 |
| Reports | 4 | 4 | 4 |
| Settings | 2 | 3 | 3 |
| ZRA | 17 | 1 | 3 |
| **Total** | **~114** | **~55** | **~49** |
