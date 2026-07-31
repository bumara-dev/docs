---
title: "PACRA Integration Specification"
description: "Technical specification for PACRA (Patents and Companies Registration Agency) integration in Bumara."
---

This document details the PACRA connection flow, templates, rules, and fee structures.

---

## 1. Overview

PACRA is Zambia's company and business name registry. Bumara integrates with PACRA to help businesses manage their compliance requirements:

- **Annual Returns**: Recurring annual filings for companies and business names
- **Service Requests**: One-off filings like name clearance, director changes

### Submission Model

PACRA filings are **not automated** - Bumara's backoffice submits on behalf of tenants:

1. Tenant prepares data and documents in Bumara
2. Tenant requests submission and pays fees
3. Backoffice submits to PACRA portal manually
4. Backoffice uploads evidence (screenshots, certificates)
5. Filing status updated to ACCEPTED

---

## 2. Connection Flow

### 2.1 Connect Endpoint

```
POST /api/pacra/connect
```

**Input:**

```typescript
{
  entityType: "company" | "business_name",
  companyType?: "private" | "public" | ..., // if company
  registrationNumber?: string,
  entityName: string,
  tradingName?: string,
  financialYearEndDate: string, // YYYY-MM-DD
  contactEmail: string,
  contactPhone?: string,
  address?: string,
  city?: string,
  province?: string,
  managedByBumara: boolean
}
```

**Processing:**

1. Validate input against `pacraConnectConfigSchema`
2. Upsert PACRA profile (`pacra_profiles` table)
3. Upsert regulator connection (`organization_regulator_connections` table)
4. Activate templates for the entity type
5. Generate initial filings and tasks
6. Record audit events

**Output:**

```typescript
{
  success: true,
  profileId: "uuid",
  connectionId: "uuid",
  activation: {
    success: true,
    connectionId: "uuid",
    activatedObligations: [
      {
        obligationId: "uuid",
        templateKey: "PACRA_ANNUAL_RETURN_COMPANY_V1",
        name: "PACRA Annual Return",
        filingId: "uuid",
        filingDueOn: "2025-03-31",
        tasksCreated: 6
      }
    ],
    availableServices: [
      { templateId: "uuid", templateKey: "PACRA_NAME_CLEARANCE_V1", name: "Name Clearance" }
    ],
    totalTasksCreated: 6,
    totalFilingsCreated: 1,
    auditEventIds: ["uuid", ...]
  }
}
```

### 2.2 Idempotency

The connect flow is fully idempotent:

- **Profile**: Created once per organization
- **Connection**: Created once per organization + regulator
- **Obligations**: Unique constraint on `(org_id, template_key)`
- **Filings**: Unique constraint on `(org_id, obligation_id, period_key)`
- **Tasks**: Unique constraint on `(filing_id, template_key)`

Re-calling `/connect` with the same config will return existing records without creating duplicates.

---

## 3. Obligation Templates

### 3.1 Annual Return — Company

**Template Key:** `PACRA_ANNUAL_RETURN_COMPANY_V1`

| Field | Value |
|-------|-------|
| Frequency | Annually |
| Due Date Rule | FY_END + 3 months |
| Activation Rule | `entityType = ["company"]` |

**Tasks:**

| Key | Title | Type | Required |
|-----|-------|------|----------|
| `confirm_company_details` | Confirm Company Details | review_approve | Yes |
| `confirm_directors_shareholders` | Confirm Directors & Shareholders | review_approve | Yes |
| `confirm_registered_office` | Confirm Registered Office | review_approve | Yes |
| `upload_financial_statements` | Upload Financial Statements | upload_document | No |
| `confirm_financial_year` | Confirm Financial Year | review_approve | Yes |
| `confirm_submission` | Review & Confirm Submission | review_approve | Yes |

**Documents:**

- Certificate of Incorporation (optional)
- Financial Statements (optional)

**Payment:**

- Regulator fee: Varies by company type and share capital
- Service fee: ZMW 150

---

### 3.2 Annual Return — Business Name

**Template Key:** `PACRA_ANNUAL_RETURN_BUSINESS_NAME_V1`

| Field | Value |
|-------|-------|
| Frequency | Annually |
| Due Date Rule | FY_END + 3 months |
| Activation Rule | `entityType = ["business_name"]` |

**Tasks:**

| Key | Title | Type | Required |
|-----|-------|------|----------|
| `confirm_business_details` | Confirm Business Details | review_approve | Yes |
| `confirm_proprietor_details` | Confirm Proprietor Details | review_approve | Yes |
| `confirm_business_address` | Confirm Business Address | review_approve | Yes |
| `confirm_submission` | Review & Confirm Submission | review_approve | Yes |

**Payment:**

- Regulator fee: Fixed amount (lower than company)
- Service fee: ZMW 100

---

## 4. Service Templates

### 4.1 Name Clearance

**Template Key:** `PACRA_NAME_CLEARANCE_V1`

Used before registering a new company or business name. Checks if the proposed name is available.

**Intake Fields:**

- First Choice Name (required)
- Second Choice Name (optional)
- Third Choice Name (optional)
- Type of Entity (required)
- Reason for Name (optional)

**Payment:**

- Regulator fee: Fixed
- Service fee: ZMW 50

---

### 4.2 Change of Directors/Secretary

**Template Key:** `PACRA_CHANGE_DIRECTORS_V1`

Notify PACRA when directors or company secretary change.

**Activation Rule:** `entityType = ["company"]`

**Intake Fields:**

- Type of Change (appointment/resignation/removal/secretary/multiple)
- Effective Date
- Details of Change

**Tasks:**

- Upload Board Resolution
- Upload Consent Letter (for appointments)
- Upload Resignation Letter (for resignations)
- Provide Director Details
- Review & Confirm

**Documents:**

- Board Resolution (required)
- Consent Letter (conditional)
- Resignation Letter (conditional)
- NRC/ID Copy (optional)

**Payment:**

- Service fee: ZMW 100

---

### 4.3 Change of Registered Office

**Template Key:** `PACRA_CHANGE_REGISTERED_OFFICE_V1`

Notify PACRA of address change.

**Activation Rule:** `entityType = ["company"]`

**Intake Fields:**

- New Address
- Effective Date
- City
- Province

**Tasks:**

- Upload Board Resolution
- Provide Complete Address
- Review & Confirm

**Payment:**

- Service fee: ZMW 75

---

## 5. Due Date Computation

### 5.1 Annual Return Due Date

Due date = Financial Year End + 3 months

**Examples:**

| FY End | Due Date |
|--------|----------|
| December 31, 2024 | March 31, 2025 |
| June 30, 2024 | September 30, 2024 |
| March 31, 2025 | June 30, 2025 |

### 5.2 Period Key Format

| Frequency | Format | Example |
|-----------|--------|---------|
| Annual | `FY{YYYY}` | FY2024 |
| Quarterly | `{YYYY}-Q{N}` | 2024-Q1 |
| Monthly | `{YYYY}-{MM}` | 2024-01 |

---

## 6. Fee Structure

Regulator fees are centrally managed in the `regulator_fees` table. Backoffice updates these when PACRA publishes fee changes.

### 6.1 Regulator Fees

| Fee Key | Service | Amount (ZMW) |
|---------|---------|--------------|
| `PACRA_ANNUAL_RETURN_COMPANY` | Annual Return (Private Company) | 250.00 |
| `PACRA_ANNUAL_RETURN_COMPANY_PUBLIC` | Annual Return (Public Company) | 500.00 |
| `PACRA_ANNUAL_RETURN_BUSINESS_NAME` | Annual Return (Business Name) | 150.00 |
| `PACRA_NAME_CLEARANCE` | Name Clearance | 100.00 |
| `PACRA_CHANGE_DIRECTORS` | Change of Directors | 200.00 |
| `PACRA_CHANGE_REGISTERED_OFFICE` | Change of Registered Office | 150.00 |

### 6.2 Service Fees (Bumara)

| Service | Fee (ZMW) |
|---------|-----------|
| Annual Return (Company) | 150.00 |
| Annual Return (Business Name) | 100.00 |
| Name Clearance | 50.00 |
| Change of Directors | 100.00 |
| Change of Registered Office | 75.00 |

> **Note:** All amounts stored in minor units (ngwee). Display as `amount / 100` for ZMW.

---

## 7. Audit Trail

All PACRA operations create audit log entries:

| Event | Entity Type | Logged Fields |
|-------|-------------|---------------|
| Profile Created/Updated | `pacra_profile` | entityType, entityName |
| Connection Created | `regulator_connection` | regulatorKey, entityType |
| Obligation Activated | `org_obligation` | templateKey, name |
| Filing Generated | `filing` | periodKey, dueOn |
| Template Activation | `regulator_activation` | activatedObligations, availableServices |

---

## 8. API Reference

### Connect to PACRA

```http
POST /api/pacra/connect
Authorization: Bearer {token}
Content-Type: application/json

{
  "entityType": "company",
  "entityName": "My Company Ltd",
  "financialYearEndDate": "2024-12-31",
  "contactEmail": "admin@company.com",
  "managedByBumara": true
}
```

### Get Activation Status

```http
GET /api/pacra/activation-status
Authorization: Bearer {token}
```

Returns current connection status and activated templates.

---

## 9. Error Handling

| Error Code | Meaning |
|------------|---------|
| `BAD_REQUEST` | Invalid input (e.g., missing required fields) |
| `UNAUTHORIZED` | Not authenticated |
| `FORBIDDEN` | Not authorized for this organization |
| `CONFLICT` | Connection already exists (for non-idempotent operations) |
| `INTERNAL` | System error (e.g., PACRA regulator not seeded) |

---

## 10. Future Enhancements

- [ ] Auto-generate next period filing on completion
- [ ] Email reminders for upcoming due dates
- [ ] Direct PACRA portal integration (if API available)
- [ ] Bulk operations for multi-company accounts

