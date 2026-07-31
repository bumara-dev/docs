---
title: "Bumara Database Architecture"
description: "Bumara is a business compliance platform for Zambian businesses, designed to centralize and automate regulatory obligations across multiple government..."
---

## Table of Contents
1. [Business Overview](#business-overview)
2. [Architecture Principles](#architecture-principles)
3. [Core Schema Design](#core-schema-design)
4. [Table Descriptions](#table-descriptions)
5. [Relations & Data Flow](#relations--data-flow)
6. [Use Cases & Queries](#use-cases--queries)
7. [Scalability & Future Considerations](#scalability--future-considerations)

---

## Business Overview`

### What is Bumara?

Bumara is a **business compliance platform** for Zambian businesses, designed to centralize and automate regulatory obligations across multiple government agencies and regulatory bodies.

### Core Regulators

1. **ZRA (Zambia Revenue Authority)** - Tax compliance, Smart Invoicing
2. **NAPSA (National Pension Scheme Authority)** - Social security contributions
3. **PACRA (Patents and Companies Registration Agency)** - Business registration and corporate filings

### Expandable to Include

- **NIHMA** - National Institute for Health Management (health sector)
- **NCC** - National Construction Council (construction sector)
- **ECZ** - Energy Regulation Board (energy sector)
- **ZICTA** - Zambia Information and Communications Technology Authority
- **Local Councils** - Local government permits and licenses

### Key Business Requirements

1. **Multi-tenant architecture** - Support multiple organizations
2. **Flexible regulator management** - Easy to add new regulators without code changes
3. **Plan-based access control** - Different subscription tiers (Starter, Standard, Premium)
4. **Compliance tracking** - Monitor deadlines, submissions, and statuses
5. **Task management** - Centralized workflow for compliance obligations
6. **Specialized schemas** - Major regulators have dedicated tables for complex data
7. **Generic connections** - All regulators connect through a unified interface

---

## Architecture Principles

### 1. Hybrid Schema Approach

```
┌─────────────────────────────────────────────────────────────┐
│                    CORE ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────┐         ┌────────────────────┐       │
│  │  organizations   │◄───────►│  subscriptions     │       │
│  └────────┬─────────┘         └────────────────────┘       │
│           │                                                  │
│           │ 1:N                                             │
│           ▼                                                  │
│  ┌─────────────────────────────────────────────┐           │
│  │ organization_regulator_connections          │           │
│  │ (Universal junction for all regulators)     │           │
│  └────────┬─────────────────────────┬──────────┘           │
│           │                          │                      │
│           │                          │                      │
│  ┌────────▼─────────┐       ┌───────▼──────────┐          │
│  │   regulators     │       │ obligation_tasks │          │
│  │  (Registry)      │       │                  │          │
│  └──────────────────┘       └──────────────────┘          │
│                                                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              SPECIALIZED SCHEMAS (Major Regulators)         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────┐ │
│  │  zra_profiles  │  │ pacra_profiles │  │napsa_profiles│ │
│  │  ├─paye_returns│  │  ├─filings     │  │├─employees    │ │
│  │  ├─wht_returns │  │  ├─directors   │  │├─contributions│ │
│  │  └─tot_returns │  │  └─changes     │  │├─returns      │ │
│  │                │  │                │  │└─schedules    │ │
│  └────────────────┘  └────────────────┘  └──────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Why Hybrid?**
- **Specialized schemas** for major regulators (ZRA, PACRA, NAPSA) with complex, domain-specific data
- **Generic connections** table for all regulators (including major ones) to provide unified access control and compliance tracking
- **Best of both worlds**: Rich domain models + flexible extensibility

### 2. Registry Pattern for Regulators

Instead of hardcoding regulator information, we use a **registry table** (`regulators`) that acts as a central catalog of all regulatory bodies.

**Benefits:**
- Add new regulators through data entry (no code changes)
- Centralized configuration and access control
- Dynamic task type definitions per regulator
- Easy to enable/disable regulators

### 3. Plan-Based Access Control

```
Subscription Plans (subscriptions table)
  ├─ Starter: Access to ZRA, PACRA, NAPSA
  ├─ Standard: + Smart Invoicing
  └─ Premium: + Industry regulators (add-ons)

Access Control Flow:
  User creates task → Check regulator → Check subscription
    ├─ regulators.minimumPlanRequired ≤ user's plan? ✓
    ├─ regulators.requiresAddOn? Check add-ons array ✓
    └─ regulators.isAvailableInTrial? Allow trial users ✓
```

---

## Core Schema Design

### Entity Relationship Diagram

```
organizations (1) ─────► (*) subscriptions
     │
     │
     ├──► (*) organization_members
     │
     ├──► (*) zra_profiles ────────┐
     │    ├─► paye_returns          │
     │    ├─► wht_returns           │
     │    └─► tot_returns           │
     │                              │
     ├──► (*) pacra_profiles ───────┼─ Specialized Data
     │    ├─► filings               │
     │    └─► directors             │
     │                              │
     ├──► (*) napsa_profiles ───────┘
     │    ├─► napsa_employees
     │    ├─► napsa_contributions
     │    ├─► napsa_returns
     │    └─► napsa_contribution_schedule
     │
     │
     └──► (*) organization_regulator_connections
              │
              ├──► (1) regulators ──► defines → task types
              │
              └──► (*) obligation_tasks
                   │
                   └──► (*) regulator_submissions
                        │
                        ├──► (*) task_documents
                        ├──► (*) task_comments
                        └──► (*) task_checklist_items
```

---

## Table Descriptions

### 1. `organizations` - Core Tenant Table

**Purpose**: Represents a business/company using Bumara.

**Key Fields**:
```typescript
{
  id: text (Clerk organization ID)
  name: text
  slug: text (unique URL identifier)
  legalName: text
  tradingName: text
  businessType: enum ('business_name' | 'local_company')
  
  // Business identifiers
  tpin: text (ZRA Tax ID - unique)
  incorporationNumber: text (PACRA registration - unique)
  
  // Contact & Address
  email, phone, address, city, province, country
  
  // Financial
  financialYearEnd: text ('MM-DD' format)
  baseCurrency: text (default 'ZMW')
  
  isActive: boolean
  createdAt, updatedAt: timestamps
}
```

**Indexes**:
- `id` (primary)
- `slug` (unique, for URL routing)
- `isActive + createdAt` (for dashboard queries)

**Relations**:
- `1:N` → subscriptions (billing)
- `1:N` → organization_members (users)
- `1:N` → organization_regulator_connections (regulator subscriptions)
- `1:N` → zra_profiles, pacra_profiles (specialized data)
- `1:N` → obligation_tasks (compliance tasks)

---

### 2. `regulators` - Registry of All Regulatory Bodies

**Purpose**: Central catalog of all government agencies and regulatory bodies. This is the **source of truth** for what regulators exist and how to access them.

**Key Fields**:
```typescript
{
  id: uuid (primary key)
  code: varchar(50) UNIQUE (e.g., 'zra', 'nihma', 'ncc')
  name: text ('Zambia Revenue Authority')
  shortName: varchar(100) ('ZRA')
  category: varchar(100) ('taxation', 'health', 'construction')
  industry: text (sector classification)
  
  // === ACCESS CONTROL ===
  minimumPlanRequired: enum ('starter' | 'standard' | 'premium')
  requiresAddOn: boolean (premium feature?)
  requiredAddOnId: varchar(100) (e.g., 'regulator_nihma')
  isPublicAccess: boolean (free for all?)
  isAvailableInTrial: boolean (trial access?)
  trialFeatureLimits: jsonb {
    maxSubmissionsPerMonth?: number
    maxDocuments?: number
    featuresDisabled?: string[]
  }
  
  // PRICING
  monthlyFeeInZMW: integer (subscription fee)
  perSubmissionFeeInZMW: integer (per-transaction fee)
  
  // FEATURE FLAGS
  hasSpecializedSchema: boolean (has zra_profiles-like table?)
  hasApiIntegration: boolean (automated submission?)
  
  // CONFIGURATION
  config: jsonb {
    portalUrl?: string (regulator's website)
    supportEmail?: string
    supportPhone?: string
    requiresRegistration?: boolean
    registrationFields?: string[] (required data)
    defaultComplianceSchedule?: string (filing frequency)
  }
  
  // TASK TYPES (Dynamic!)
  availableTaskTypes: jsonb [{
    code: string ('vat_return', 'license_renewal')
    name: string ('VAT Return')
    description?: string
    category?: string ('tax_filing', 'license')
    isRecurring?: boolean
    defaultPriority?: 'low' | 'normal' | 'high' | 'urgent' | 'critical'
    estimatedHours?: number
    defaultDocuments?: string[] (['financial_statements', 'receipt'])
    serviceFee?: number (Bumara's fee in ZMW)
    regulatoryFee?: number (government fee in ZMW)
    totalFee?: number (total in ZMW)
  }]
  
  // STATUS
  isActive: boolean (enabled?)
  isNational: boolean (vs provincial)
  province?: text (if provincial regulator)
  
  // METADATA
  description: text
  website: text
  createdAt, updatedAt: timestamps
}
```

**Example Data**:
```typescript
// ZRA Record
{
  code: 'zra',
  name: 'Zambia Revenue Authority',
  shortName: 'ZRA',
  category: 'taxation',
  minimumPlanRequired: 'starter',
  isPublicAccess: true,
  isAvailableInTrial: true,
  hasSpecializedSchema: true,
  hasApiIntegration: true,
  availableTaskTypes: [
    {
      code: 'vat_return',
      name: 'VAT Return',
      category: 'tax_filing',
      isRecurring: true,
      defaultPriority: 'high',
      serviceFee: 100, // ZMW 100
      regulatoryFee: 0
    },
    {
      code: 'paye_return',
      name: 'PAYE Monthly Return',
      category: 'tax_filing',
      isRecurring: true,
      serviceFee: 150
    }
  ]
}

// NIHMA Record
{
  code: 'nihma',
  name: 'National Institute for Health Management of Africa',
  shortName: 'NIHMA',
  category: 'health',
  industry: 'healthcare',
  minimumPlanRequired: 'premium',
  requiresAddOn: true,
  requiredAddOnId: 'regulator_nihma',
  isAvailableInTrial: false,
  hasSpecializedSchema: false,
  availableTaskTypes: [
    {
      code: 'facility_inspection',
      name: 'Health Facility Inspection',
      category: 'inspection',
      isRecurring: true,
      serviceFee: 500,
      regulatoryFee: 1000
    },
    {
      code: 'license_renewal',
      name: 'Practice License Renewal',
      category: 'license',
      isRecurring: true,
      serviceFee: 300,
      regulatoryFee: 5000
    }
  ]
}
```

**Why This Design is Powerful**:
1. **No code changes** to add NIHMA, NCC, or any new regulator
2. **Dynamic task types** - each regulator defines its own tasks
3. **Flexible pricing** - per-regulator or per-submission fees
4. **Access control** - enforced at data level
5. **Configuration** - portal URLs, support contacts stored per regulator

**Indexes**:
- `code` (unique, for lookups)
- `category + isActive` (filter by type)
- `minimumPlanRequired + isPublicAccess` (access control queries)

---

### 3. `organization_regulator_connections` - Universal Junction Table

**Purpose**: Tracks which regulators each organization is **registered with and actively using**. This is the **universal connection point** for all regulators (including ZRA, PACRA, NAPSA).

**Key Fields**:
```typescript
{
  id: uuid
  organizationId: text (FK → organizations)
  regulatorId: uuid (FK → regulators)
  
  // REGISTRATION DETAILS
  registrationNumber: text (e.g., TPIN, license number)
  registrationDate: timestamp
  registrationStatus: enum (
    'not_started' | 'pending' | 'pending_approval' | 
    'active' | 'suspended' | 'expired' | 
    'renewal_pending' | 'cancelled' | 'rejected' | 'under_review'
  )
  
  // COMPLIANCE TRACKING
  complianceStatus: enum (
    'compliant' | 'overdue' | 'not_required' | 
    'pending_review' | 'at_risk' | 'partial' | 
    'non_compliant' | 'remediation' | 
    'grace_period' | 'penalty_applied'
  )
  nextDueDate: timestamp (next filing/payment deadline)
  lastSubmissionDate: timestamp (last successful submission)
  
  // CREDENTIALS (Encrypted!)
  credentials: jsonb {
    username?: string
    password?: string // MUST be encrypted
    apiKey?: string // MUST be encrypted
    portalId?: string
    accessToken?: string // MUST be encrypted
    refreshToken?: string // MUST be encrypted
    [key: string]: any // Flexible for different regulators
  }
  
  // METADATA (Flexible per regulator)
  metadata: jsonb {
    licenseNumber?: string
    permitNumber?: string
    certificateNumber?: string
    localCouncilWard?: string (for local councils)
    industryCategory?: string
    [key: string]: any
  }
  
  // PREFERENCES
  isActive: boolean (currently using?)
  autoCompliance: boolean (enable automated submissions?)
  
  createdAt, updatedAt: timestamps
}
```

**Unique Constraint**: `(organizationId, regulatorId)` - An organization can only connect to each regulator ONCE.

**Example Data**:
```typescript
// Organization ABC connected to ZRA
{
  organizationId: 'org_abc123',
  regulatorId: 'uuid-of-zra',
  registrationNumber: '1234567890', // TPIN
  registrationStatus: 'active',
  complianceStatus: 'compliant',
  nextDueDate: '2025-11-15',
  lastSubmissionDate: '2025-10-15',
  credentials: {
    // Encrypted credentials for Smart Invoicing API
    apiKey: 'encrypted_api_key',
    portalId: 'ZRA123'
  },
  isActive: true,
  autoCompliance: true
}

// Organization ABC connected to NIHMA
{
  organizationId: 'org_abc123',
  regulatorId: 'uuid-of-nihma',
  registrationNumber: 'NIHMA/2024/001',
  registrationStatus: 'active',
  complianceStatus: 'at_risk', // Inspection due soon
  nextDueDate: '2025-11-30',
  metadata: {
    licenseNumber: 'MED-LIC-12345',
    facilityType: 'clinic',
    numberOfBeds: 10
  },
  isActive: true
}
```

**Indexes**:
- `organizationId + isActive` (get active connections for an org)
- `regulatorId` (find all orgs using a regulator)
- `nextDueDate` (upcoming deadlines)
- `complianceStatus` (filter by compliance state)

**Relations**:
- `N:1` → organizations (belongs to org)
- `N:1` → regulators (connects to regulator)
- `1:N` → obligation_tasks (tasks for this connection)
- `1:N` → regulator_submissions (submissions for this connection)

---

### 4. `subscriptions` - Billing & Access Control

**Purpose**: Manages organization's subscription plan and add-ons, which control access to regulators and features.

**Key Fields**:
```typescript
{
  id: uuid
  organizationId: text (FK → organizations)
  
  plan: enum ('starter' | 'standard' | 'premium')
  status: enum ('trial' | 'active' | 'cancelled')
  
  trialStartsAt: timestamp
  trialEndsAt: timestamp
  
  // FEATURE TOGGLES
  aiEnabled: boolean
  addOns: jsonb [{
    id: // Core add-ons
      'inventory' | 'sms_pack' | 'ai_credits' | 
      'extra_storage' | 'smart_einvoice' |
      // Regulator add-ons
      'regulator_nihma' | 'regulator_ncc' | 
      'regulator_ecz' | 'regulator_zicta' |
      'industry_regulators_bundle' | 
      'local_councils_access'
    qty?: number
    metadata?: {
      regulatorId?: string
      expiresAt?: string
      [key: string]: any
    }
  }]
  
  currentPeriodStart: timestamp
  currentPeriodEnd: timestamp
  createdAt, updatedAt: timestamps
}
```

**Access Control Logic**:
```typescript
async function canAccessRegulator(
  organizationId: string, 
  regulatorCode: string
): Promise<boolean> {
  const regulator = await getRegulatorByCode(regulatorCode);
  const subscription = await getSubscription(organizationId);
  
  // Public access regulators
  if (regulator.isPublicAccess) return true;
  
  // Trial access
  if (subscription.status === 'trial' && regulator.isAvailableInTrial) {
    return checkTrialLimits(subscription, regulator);
  }
  
  // Check minimum plan
  const planHierarchy = { starter: 1, standard: 2, premium: 3 };
  if (planHierarchy[subscription.plan] < planHierarchy[regulator.minimumPlanRequired]) {
    return false;
  }
  
  // Check add-on requirement
  if (regulator.requiresAddOn) {
    return subscription.addOns.some(
      addon => addon.id === regulator.requiredAddOnId
    );
  }
  
  return true;
}
```

---

### 5. `obligation_tasks` - Compliance Workflow Engine

**Purpose**: Central task management system for all compliance obligations across all regulators.

**Key Fields**:
```typescript
{
  id: uuid
  organizationId: text (FK → organizations)
  taskNumber: text UNIQUE (e.g., 'TASK-2024-001')
  
  // TASK TYPE (Dynamic - references regulator.availableTaskTypes)
  taskType: varchar(100) (e.g., 'vat_return', 'facility_inspection')
  
  // REGULATOR LINK
  regulatorId: uuid (FK → regulators)
  category: varchar(100) ('tax_filing', 'license', 'inspection')
  
  // DETAILS
  title: varchar(255) ('VAT Return - October 2024')
  description: text
  
  // PRIORITY & DEADLINES
  priority: enum ('low' | 'normal' | 'high' | 'urgent' | 'critical')
  dueDate: timestamp (Bumara internal deadline)
  regulatorDeadline: timestamp (actual regulator deadline)
  
  // STATUS TRACKING
  status: enum (
    'draft' | 'submitted' | 'acknowledged' | 'assigned' | 
    'in_progress' | 'pending_documents' | 'pending_approval' | 
    'ready_for_submission' | 'submitted_to_regulator' | 
    'regulator_processing' | 'regulator_approved' | 
    'regulator_rejected' | 'completed' | 'cancelled' | 
    'failed' | 'on_hold'
  )
  previousStatus: enum (status history)
  statusChangedAt: timestamp
  
  // ASSIGNMENT
  assignedTo: uuid (FK → back_office_agents)
  assignedAt: timestamp
  assignedBy: uuid (FK → back_office_agents)
  
  // SUBMISSION
  submittedBy: text (FK → organization_members)
  submittedAt: timestamp
  acknowledgedBy: uuid (FK → back_office_agents)
  acknowledgedAt: timestamp
  
  // COMPLETION
  completedBy: uuid (FK → back_office_agents)
  completedAt: timestamp
  
  // TIME TRACKING
  estimatedHours: decimal(5,2)
  actualHours: decimal(5,2)
  
  // RELATED ENTITIES (Polymorphic)
  relatedEntityType: varchar(50) ('paye_return', 'vat_return')
  relatedEntityId: uuid (FK to specific table)
  
  // FINANCIAL
  serviceFee: decimal(15,2) (Bumara's fee)
  governmentFee: decimal(15,2) (regulator's fee)
  totalCost: decimal(15,2)
  isPaid: boolean
  paidAt: timestamp
  
  // METADATA
  tags: jsonb string[]
  isUrgent: boolean
  isRecurring: boolean
  recurringSchedule: varchar(50) ('monthly', 'quarterly', 'annually')
  
  // SLA
  slaDeadline: timestamp
  isOverdue: boolean
  
  createdAt, updatedAt: timestamps
}
```

**Composite Relation**: `(organizationId, regulatorId)` maps to `organization_regulator_connections`, allowing you to fetch the connection details for any task.

**Example Task**:
```typescript
{
  organizationId: 'org_abc123',
  taskNumber: 'TASK-2024-0523',
  taskType: 'facility_inspection', // From NIHMA's availableTaskTypes
  regulatorId: 'uuid-of-nihma',
  category: 'inspection',
  title: 'Annual Health Facility Inspection - 2024',
  description: 'Mandatory annual inspection for clinic operations',
  priority: 'high',
  dueDate: '2024-11-20T00:00:00Z',
  regulatorDeadline: '2024-11-30T00:00:00Z',
  status: 'assigned',
  assignedTo: 'agent_123',
  submittedBy: 'member_456',
  serviceFee: 500.00,
  governmentFee: 1000.00,
  totalCost: 1500.00,
  isRecurring: true,
  recurringSchedule: 'annually'
}
```

**Indexes**:
- `taskNumber` (unique)
- `organizationId + status` (org's task list)
- `assignedTo + status` (agent's queue)
- `status + priority + dueDate` (prioritization)
- `regulatorId + status` (regulator-specific tasks)
- `dueDate` WHERE status NOT IN ('completed', 'cancelled') (upcoming tasks)
- `relatedEntityType + relatedEntityId` (polymorphic lookup)

---

### 6. `regulator_submissions` - Submission Log

**Purpose**: Tracks every submission made to any regulator, with status tracking and response logging.

**Key Fields**:
```typescript
{
  id: uuid
  taskId: uuid (FK → obligation_tasks)
  organizationId: text (FK → organizations)
  regulatorId: uuid (FK → regulators)
  
  // COMPLIANCE ENTITY (what was submitted)
  complianceEntityType: text ('paye_return', 'vat_return', 'license_application')
  complianceEntityId: uuid (FK to specific table)
  
  // SUBMISSION DETAILS
  submissionMethod: text ('api' | 'web_portal' | 'manual' | 'email')
  submittedAt: timestamp
  submittedBy: uuid (FK → back_office_agents)
  
  // REGULATOR RESPONSE
  regulatorReferenceNumber: text (confirmation/tracking number)
  regulatorStatus: enum (
    'draft' | 'pending' | 'submitted' | 'received' | 
    'processing' | 'approved' | 'rejected' | 
    'requires_clarification' | 'resubmitted' | 
    'cancelled' | 'error' | 'expired'
  )
  regulatorResponse: text (response message)
  regulatorResponseDate: timestamp
  
  // OUTCOME
  isSuccessful: boolean
  
  // DOCUMENTS
  confirmationDocumentUrl: text (S3 URL)
  receiptDocumentUrl: text (payment receipt URL)
  
  createdAt, updatedAt: timestamps
}
```

**Composite Relation**: `(organizationId, regulatorId)` maps to `organization_regulator_connections`.

**Indexes**:
- `taskId` (submissions for a task)
- `regulatorId + regulatorStatus` (filter by regulator and status)
- `organizationId + submittedAt` (org's submission history)
- `regulatorReferenceNumber` (lookup by tracking number)
- `complianceEntityType + complianceEntityId` (polymorphic lookup)

---

### 7. Specialized Schemas (ZRA, PACRA, NAPSA)

#### `zra_profiles` - ZRA Tax Profile

**Purpose**: Stores ZRA-specific tax registration information that's too complex for the generic connections table.

```typescript
{
  id: uuid
  organizationId: text (FK → organizations)
  tpin: varchar(10) UNIQUE
  businessName: text
  
  // Tax Type Registrations
  isTurnOverTaxRegistered: boolean
  isVatRegistered: boolean
  isPayeRegistered: boolean
  isWhtRegistered: boolean
  isTaxRegistrationCompleted: boolean
  
  nextReturnDueDate: timestamp
  createdAt, updatedAt: timestamps
}
```

**Relations**: Also has `organization_regulator_connections` entry with `regulatorId` pointing to ZRA.

**Child Tables**:
- `paye_returns` - PAYE tax returns
- `turnover_tax_returns` - ToT returns
- `wht_returns` - Withholding tax returns
- `employee_tax_profiles` - Employee PAYE data
- `supplier_tax_profiles` - Supplier WHT data
- `payroll_runs` - Monthly payroll calculations
- `payslips` - Employee payslips

#### `pacra_profiles` - PACRA Registration Profile

**Purpose**: Stores PACRA-specific company registration and incorporation data.

```typescript
{
  id: uuid
  organizationId: text (FK → organizations)
  registrationNumber: text UNIQUE
  businessName: text
  tradingName: varchar(255)
  
  entityType: enum ('company' | 'business_name' | 'partnership' | 'trust' | 'ngo')
  
  registrationDate: timestamp
  incorporationDate: timestamp
  status: enum ('active' | 'suspended' | 'dissolved')
  
  businessActivity: text
  address, city, province, postalCode, country: text
  
  annualReturnDueDate: timestamp
  lastAnnualReturnFiled: timestamp
  
  createdAt, updatedAt: timestamps
}
```

**Child Tables**:
- `pacra_filings` - Annual returns, amendments
- `pacra_directors` - Company directors
- `pacra_change_history` - Change logs

#### `napsa_profiles` - NAPSA Employer Profile

**Purpose**: ✅ **FULLY IMPLEMENTED** - Stores NAPSA employer registration and contribution information.

**Structure**:
```typescript
{
  id: uuid
  organizationId: text (FK → organizations)
  employerNumber: text UNIQUE (NAPSA employer registration number)
  employerName: text
  
  registrationDate: timestamp
  status: varchar(255) (default: 'active')
  
  // Contact details
  contactPerson: text
  contactEmail: text
  contactPhone: text
  
  // Portal credentials (encrypted)
  portalCredentials: jsonb {
    username?: string
    password?: string // MUST be encrypted
  }
  
  createdAt, updatedAt: timestamps
}
```

**Indexes**:
- `employerNumber` (unique, for lookups)
- `organizationId` (for org queries)
- `status` (for filtering active employers)

**Child Tables**:
- ✅ `napsa_employees` - Employee records
- ✅ `napsa_contributions` - Individual contributions
- ✅ `napsa_returns` - Monthly returns
- ✅ `napsa_contribution_schedule` - Detailed contribution schedule

---

#### `napsa_employees` - NAPSA Employee Records

**Purpose**: ✅ **FULLY IMPLEMENTED** - Stores employee information for NAPSA contributions.

**Structure**:
```typescript
{
  id: uuid
  organizationId: text (FK → organizations)
  napsaProfileId: uuid (FK → napsa_profiles)
  employeeTaxProfileId: uuid (FK → employee_tax_profiles, nullable)
  
  // NAPSA identification
  napsaNumber: text (employee's NAPSA number)
  nrcNumber: text (National Registration Card)
  
  // Personal details
  firstName: text
  lastName: text
  dateOfBirth: timestamp
  gender: varchar(20)
  
  // Employment
  employmentStartDate: timestamp
  employmentEndDate: timestamp
  isActive: boolean (default: true)
  
  basicSalary: decimal(18,2)
  
  createdAt, updatedAt: timestamps
}
```

**Unique Constraint**: `(organizationId, napsaNumber)` - Prevents duplicate employee records

**Indexes**:
- `napsaProfileId` (for profile queries)
- `organizationId` (for org queries)
- `isActive` (for filtering active employees)

**Cross-Integration**: Links to ZRA's `employee_tax_profiles` for unified employee management

---

#### `napsa_contributions` - Individual Contributions

**Purpose**: ✅ **FULLY IMPLEMENTED** - Tracks monthly contributions for each employee.

**Structure**:
```typescript
{
  id: uuid
  organizationId: text (FK → organizations)
  napsaProfileId: uuid (FK → napsa_profiles)
  employeeId: uuid (FK → napsa_employees)
  payslipId: uuid (FK → payslips, nullable)
  
  contributionMonth: varchar(7) ('YYYY-MM' format)
  
  // Contribution breakdown (5% employee + 5% employer = 10% total)
  grossSalary: decimal(18,2)
  employeeContribution: decimal(18,2) // 5% of gross
  employerContribution: decimal(18,2) // 5% of gross
  totalContribution: decimal(18,2) // 10% of gross
  
  // Status tracking
  status: varchar(20) (default: 'draft')
  submittedAt: timestamp
  paidAt: timestamp
  
  napsaReferenceNumber: varchar(50)
  
  createdAt, updatedAt: timestamps
}
```

**Unique Constraint**: `(organizationId, contributionMonth, employeeId)` - One contribution per employee per month

**Indexes**:
- `napsaProfileId` (for profile queries)
- `status` (for filtering by status)

**Cross-Integration**: Links to ZRA's `payslips` for automated contribution calculation

---

#### `napsa_returns` - Monthly Returns

**Purpose**: ✅ **FULLY IMPLEMENTED** - Aggregate monthly returns submitted to NAPSA.

**Structure**:
```typescript
{
  id: uuid
  organizationId: text (FK → organizations)
  napsaProfileId: uuid (FK → napsa_profiles)
  
  contributionMonth: varchar(7) ('YYYY-MM' format)
  periodStartDate: date
  periodEndDate: date
  
  // Summary totals
  totalEmployees: integer
  totalGrossSalary: decimal(18,2)
  totalEmployeeContribution: decimal(18,2)
  totalEmployerContribution: decimal(18,2)
  totalContribution: decimal(18,2)
  
  // Penalties
  lateFees: decimal(18,2) (default: 0.00)
  penalties: decimal(18,2) (default: 0.00)
  
  // Submission tracking
  status: returnStatusEnum (default: 'draft')
  submittedAt: timestamp
  submittedBy: text (FK → organization_members)
  napsaReferenceNumber: varchar(50)
  napsaResponse: text
  
  // Payment tracking
  paidAt: timestamp
  paymentReference: varchar(50)
  dueDate: timestamp
  filedOnTime: boolean
  
  createdAt, updatedAt: timestamps
}
```

**Unique Constraint**: `(organizationId, contributionMonth)` - One return per org per month

**Indexes**:
- `dueDate` (for deadline queries)
- `status` (for filtering by status)

---

#### `napsa_contribution_schedule` - Detailed Contribution Schedule

**Purpose**: ✅ **FULLY IMPLEMENTED** - Line-item detail of contributions for each return.

**Structure**:
```typescript
{
  id: uuid
  napsaReturnId: uuid (FK → napsa_returns)
  contributionId: uuid (FK → napsa_contributions, nullable)
  
  // Employee identification (denormalized for reporting)
  employeeNumber: varchar(50)
  employeeName: varchar(255)
  employeeSsn: varchar(50) // Social Security Number
  employeeTpin: varchar(10)
  employeeNRC: varchar(50)
  
  // Contribution amounts
  grossSalary: decimal(18,2)
  employeeContribution: decimal(18,2)
  employerContribution: decimal(18,2)
  totalContribution: decimal(18,2)
  
  createdAt: timestamp
}
```

**Indexes**:
- `napsaReturnId` (for return queries)
- `contributionId` (for linking to contributions)

**Purpose**: This table provides the detailed breakdown that appears on the official NAPSA return form, with each row representing one employee's contribution for that month.

---

## Relations & Data Flow

### 1. Organization Setup Flow

```
1. User signs up → Organization created in Clerk
   ↓
2. Organization record created (id = Clerk org ID)
   ↓
3. Subscription created (status = 'trial', plan = 'starter')
   ↓
4. Onboarding: User provides TPIN & Incorporation Number
   ↓
5. System creates:
   - zra_profiles (if TPIN provided)
   - pacra_profiles (if incorporation number provided)
   - organization_regulator_connections for ZRA
   - organization_regulator_connections for PACRA
```

### 2. Adding a New Regulator (NIHMA Example)

```
Admin creates regulator record:
  ↓
INSERT INTO regulators (
  code = 'nihma',
  name = 'National Institute for Health Management',
  category = 'health',
  minimumPlanRequired = 'premium',
  requiresAddOn = true,
  requiredAddOnId = 'regulator_nihma',
  availableTaskTypes = [
    { code: 'facility_inspection', name: 'Facility Inspection', ... },
    { code: 'license_renewal', name: 'License Renewal', ... }
  ]
)
  ↓
No code changes needed! System immediately supports:
  - Checking access (plan + add-on)
  - Creating tasks with NIHMA task types
  - Tracking compliance for NIHMA
  - Displaying NIHMA in UI (via regulators query)
```

### 3. Task Creation & Execution Flow

```
User initiates compliance task:
  ↓
1. Check access: canAccessRegulator(orgId, regulatorCode)
   ├─ Check subscription plan
   ├─ Check required add-ons
   └─ Verify org has connection to regulator
  ↓
2. Fetch available task types from regulator.availableTaskTypes
  ↓
3. Create task:
   INSERT INTO obligation_tasks (
     organizationId,
     regulatorId,
     taskType = selected task type code,
     ...
   )
  ↓
4. Task assigned to back office agent
  ↓
5. Agent processes task, uploads documents
  ↓
6. Submit to regulator:
   INSERT INTO regulator_submissions (
     taskId,
     organizationId,
     regulatorId,
     complianceEntityType,
     complianceEntityId,
     submissionMethod
   )
  ↓
7. Update organization_regulator_connections:
   - lastSubmissionDate = now()
   - complianceStatus = 'compliant'
   - nextDueDate = calculated based on schedule
  ↓
8. Task status = 'completed'
```

### 4. Query Patterns

#### Get all regulators an org has access to:

```typescript
const accessibleRegulators = await db
  .select()
  .from(regulators)
  .where(
    or(
      eq(regulators.isPublicAccess, true),
      and(
        lte(regulators.minimumPlanRequired, subscription.plan),
        or(
          eq(regulators.requiresAddOn, false),
          inArray(regulators.requiredAddOnId, userAddOnIds)
        )
      )
    )
  )
  .where(eq(regulators.isActive, true));
```

#### Get org's active regulator connections with compliance status:

```typescript
const connections = await db.query.organizationRegulatorConnections.findMany({
  where: and(
    eq(organizationRegulatorConnections.organizationId, orgId),
    eq(organizationRegulatorConnections.isActive, true)
  ),
  with: {
    regulator: true,
  },
  orderBy: [asc(organizationRegulatorConnections.nextDueDate)],
});

// Result:
[
  {
    id: 'conn_1',
    registrationNumber: '1234567890',
    complianceStatus: 'compliant',
    nextDueDate: '2024-11-15',
    regulator: {
      code: 'zra',
      name: 'Zambia Revenue Authority',
      category: 'taxation'
    }
  },
  {
    id: 'conn_2',
    registrationNumber: 'NIHMA/2024/001',
    complianceStatus: 'at_risk',
    nextDueDate: '2024-11-30',
    regulator: {
      code: 'nihma',
      name: 'National Institute for Health Management',
      category: 'health'
    }
  }
]
```

#### Get all tasks for a specific regulator connection:

```typescript
const tasks = await db.query.obligationTasks.findMany({
  where: and(
    eq(obligationTasks.organizationId, orgId),
    eq(obligationTasks.regulatorId, regulatorId)
  ),
  with: {
    regulatorConnection: {
      with: {
        regulator: true,
      },
    },
    submissions: {
      orderBy: [desc(regulatorSubmissions.submittedAt)],
      limit: 1,
    },
  },
  orderBy: [desc(obligationTasks.createdAt)],
});
```

#### Get upcoming deadlines across all regulators:

```typescript
const upcomingDeadlines = await db
  .select({
    regulator: regulators.name,
    taskTitle: obligationTasks.title,
    dueDate: obligationTasks.dueDate,
    complianceStatus: organizationRegulatorConnections.complianceStatus,
  })
  .from(obligationTasks)
  .innerJoin(
    regulators,
    eq(obligationTasks.regulatorId, regulators.id)
  )
  .innerJoin(
    organizationRegulatorConnections,
    and(
      eq(organizationRegulatorConnections.organizationId, obligationTasks.organizationId),
      eq(organizationRegulatorConnections.regulatorId, obligationTasks.regulatorId)
    )
  )
  .where(
    and(
      eq(obligationTasks.organizationId, orgId),
      gte(obligationTasks.dueDate, new Date()),
      notInArray(obligationTasks.status, ['completed', 'cancelled'])
    )
  )
  .orderBy(asc(obligationTasks.dueDate))
  .limit(10);
```

---

## Use Cases & Queries

### Use Case 1: Healthcare Business Needs NIHMA Compliance

**Scenario**: A private clinic wants to use Bumara for NIHMA compliance.

**Flow**:
1. Clinic signs up, creates organization
2. Subscription created with plan = 'premium' or purchases 'regulator_nihma' add-on
3. User navigates to "Add Regulator" → sees NIHMA (because they have access)
4. User clicks "Connect to NIHMA"
   ```typescript
   // System creates connection
   await db.insert(organizationRegulatorConnections).values({
     organizationId: 'org_clinic123',
     regulatorId: nihmaRegulatorId,
     registrationNumber: null, // Will be filled during registration
     registrationStatus: 'not_started',
     complianceStatus: 'not_required',
     isActive: true,
   });
   ```
5. User fills in NIHMA registration details:
   ```typescript
   await db.update(organizationRegulatorConnections)
     .set({
       registrationNumber: 'NIHMA/2024/001',
       registrationStatus: 'active',
       registrationDate: new Date(),
       metadata: {
         licenseNumber: 'MED-LIC-12345',
         facilityType: 'clinic',
       },
     })
     .where(eq(id, connectionId));
   ```
6. System now shows NIHMA tasks (from `regulators.availableTaskTypes`)
7. User creates task: "Annual Facility Inspection"
8. Task flows through workflow → submitted to NIHMA
9. Compliance tracked automatically

### Use Case 2: Dashboard - Compliance Overview

**Query**: Get compliance status across all connected regulators

```typescript
const complianceOverview = await db
  .select({
    regulator: {
      name: regulators.name,
      code: regulators.code,
      category: regulators.category,
    },
    connection: {
      complianceStatus: organizationRegulatorConnections.complianceStatus,
      nextDueDate: organizationRegulatorConnections.nextDueDate,
      lastSubmissionDate: organizationRegulatorConnections.lastSubmissionDate,
    },
    pendingTasksCount: sql<number>`
      COUNT(CASE WHEN ${obligationTasks.status} NOT IN ('completed', 'cancelled') 
      THEN 1 END)
    `,
    overdueTasksCount: sql<number>`
      COUNT(CASE WHEN ${obligationTasks.dueDate} < NOW() 
      AND ${obligationTasks.status} NOT IN ('completed', 'cancelled') 
      THEN 1 END)
    `,
  })
  .from(organizationRegulatorConnections)
  .innerJoin(
    regulators,
    eq(organizationRegulatorConnections.regulatorId, regulators.id)
  )
  .leftJoin(
    obligationTasks,
    and(
      eq(obligationTasks.organizationId, organizationRegulatorConnections.organizationId),
      eq(obligationTasks.regulatorId, organizationRegulatorConnections.regulatorId)
    )
  )
  .where(
    and(
      eq(organizationRegulatorConnections.organizationId, orgId),
      eq(organizationRegulatorConnections.isActive, true)
    )
  )
  .groupBy(
    regulators.id,
    organizationRegulatorConnections.id
  );

// Result:
[
  {
    regulator: { name: 'ZRA', code: 'zra', category: 'taxation' },
    connection: {
      complianceStatus: 'compliant',
      nextDueDate: '2024-12-15',
      lastSubmissionDate: '2024-10-15'
    },
    pendingTasksCount: 2,
    overdueTasksCount: 0
  },
  {
    regulator: { name: 'NIHMA', code: 'nihma', category: 'health' },
    connection: {
      complianceStatus: 'at_risk',
      nextDueDate: '2024-11-30',
      lastSubmissionDate: '2023-11-20'
    },
    pendingTasksCount: 1,
    overdueTasksCount: 0
  },
  {
    regulator: { name: 'PACRA', code: 'pacra', category: 'registration' },
    connection: {
      complianceStatus: 'overdue',
      nextDueDate: '2024-10-31',
      lastSubmissionDate: '2023-10-20'
    },
    pendingTasksCount: 1,
    overdueTasksCount: 1
  }
]
```

### Use Case 3: Admin Panel - Regulator Management

**Query**: Get all regulators with connection statistics

```typescript
const regulatorStats = await db
  .select({
    regulator: regulators,
    connectionsCount: sql<number>`COUNT(DISTINCT ${organizationRegulatorConnections.id})`,
    activeConnectionsCount: sql<number>`
      COUNT(DISTINCT CASE WHEN ${organizationRegulatorConnections.isActive} = true 
      THEN ${organizationRegulatorConnections.id} END)
    `,
    totalTasksCount: sql<number>`COUNT(DISTINCT ${obligationTasks.id})`,
    completedTasksCount: sql<number>`
      COUNT(DISTINCT CASE WHEN ${obligationTasks.status} = 'completed' 
      THEN ${obligationTasks.id} END)
    `,
    avgCompletionTime: sql<number>`
      AVG(EXTRACT(EPOCH FROM (${obligationTasks.completedAt} - ${obligationTasks.createdAt})) / 3600)
    `,
  })
  .from(regulators)
  .leftJoin(
    organizationRegulatorConnections,
    eq(organizationRegulatorConnections.regulatorId, regulators.id)
  )
  .leftJoin(
    obligationTasks,
    eq(obligationTasks.regulatorId, regulators.id)
  )
  .groupBy(regulators.id)
  .orderBy(desc(sql`COUNT(DISTINCT ${organizationRegulatorConnections.id})`));
```

---

## Scalability & Future Considerations

### 1. Performance Optimization

**Current Indexes Cover**:
- ✅ Org-specific queries (by organizationId)
- ✅ Regulator lookups (by code, category)
- ✅ Access control (plan + public access)
- ✅ Compliance tracking (by status, due date)
- ✅ Task management (status, priority, deadlines)

**Future Optimizations**:
- Materialized views for dashboard queries
- Partial indexes on active records only
- JSONB indexes on commonly queried metadata fields
- Read replicas for reporting queries

### 2. Adding New Regulators

**Current State**: ✅ Fully Dynamic
- No code changes needed
- Add regulator via admin panel or SQL insert
- Define task types in `availableTaskTypes` JSONB
- System immediately supports new regulator

**Example**: Adding NCC (National Construction Council)
```sql
INSERT INTO regulators (
  code, name, short_name, category, industry,
  minimum_plan_required, requires_add_on, required_add_on_id,
  available_task_types, is_active
) VALUES (
  'ncc',
  'National Construction Council',
  'NCC',
  'construction',
  'construction',
  'premium',
  true,
  'regulator_ncc',
  '[
    {
      "code": "contractor_registration",
      "name": "Contractor Registration",
      "category": "registration",
      "serviceFee": 1000,
      "regulatoryFee": 5000
    },
    {
      "code": "annual_renewal",
      "name": "Annual License Renewal",
      "category": "license",
      "isRecurring": true,
      "serviceFee": 500,
      "regulatoryFee": 3000
    }
  ]'::jsonb,
  true
);
```

Done! NCC is now available in the system.

### 3. Specialized Schema Decision Tree

**When to create a specialized schema** (like `zra_profiles`, `pacra_profiles`):

```
Is the regulator's data model complex?
  ├─ YES: Has many entity types (returns, employees, transactions)
  │   └─ Create specialized schema with multiple tables
  │
  └─ NO: Simple registration + compliance tracking
      └─ Use organization_regulator_connections only
```

**Examples**:
- **ZRA**: ✅ Specialized (PAYE, VAT, ToT, WHT - each with multiple tables)
- **PACRA**: ✅ Specialized (Directors, filings, change history)
- **NAPSA**: ✅ Specialized (Employees, contributions, returns, schedules - 5 tables total)
- **NIHMA**: ❌ Generic (Simple license + inspections - use connections table)
- **NCC**: ❌ Generic (Contractor registration + renewals)
- **Local Councils**: ❌ Generic (Permits + licenses)

### 4. Multi-Tenancy & Isolation

**Current Design**: ✅ Fully Multi-Tenant
- All tables have `organizationId` foreign key
- Row-level security possible via Postgres RLS
- Queries filtered by organization context
- No cross-organization data leakage

**Future**: Implement RLS policies
```sql
-- Example RLS policy
CREATE POLICY org_isolation_policy ON obligation_tasks
  FOR ALL
  TO authenticated_users
  USING (organization_id = current_setting('app.current_org_id')::text);
```

### 5. Audit & Compliance Tracking

**Current**: Partial (created_at, updated_at)

**Future Enhancements**:
- Full audit log table (who changed what, when)
- Status transition history
- Document version control
- Compliance certificate generation
- Regulatory report exports

### 6. API Integrations

**Planned**:
- ZRA Smart Invoicing API (already flagged with `hasApiIntegration`)
- PACRA company search API
- Automated submission via regulator APIs
- Webhook support for status updates

**Design Pattern**:
```typescript
// regulators.hasApiIntegration = true
if (regulator.hasApiIntegration) {
  const apiClient = getRegulatorApiClient(regulator.code);
  const result = await apiClient.submit(submissionData);
  
  await db.insert(regulatorSubmissions).values({
    submissionMethod: 'api',
    regulatorReferenceNumber: result.referenceNumber,
    regulatorStatus: result.status,
  });
}
```

---

## Design Patterns & Best Practices

### 1. JSONB for Flexibility

**Used In**:
- `regulators.config` - Regulator-specific configuration
- `regulators.availableTaskTypes` - Dynamic task definitions
- `regulators.trialFeatureLimits` - Trial restrictions
- `organizationRegulatorConnections.credentials` - Encrypted credentials
- `organizationRegulatorConnections.metadata` - Flexible per-regulator data
- `subscriptions.addOns` - Subscription add-ons

**Benefits**:
- No schema migrations for new fields
- Regulator-specific customization
- Easy to extend

**Caution**:
- Use sparingly (only for truly flexible data)
- Add JSONB indexes for commonly queried fields
- Validate JSONB structure in application layer

### 2. Enum Expansion Strategy

**Problem**: Adding new enum values requires migration

**Solution**: Expanded enums upfront with comprehensive values

Example:
```typescript
// registrationConnectionStatusEnum - covers entire lifecycle
'not_started' | 'pending' | 'pending_approval' | 'active' | 
'suspended' | 'expired' | 'renewal_pending' | 'cancelled' | 
'rejected' | 'under_review'

// complianceStatusEnum - covers all compliance states
'compliant' | 'overdue' | 'not_required' | 'pending_review' | 
'at_risk' | 'partial' | 'non_compliant' | 'remediation' | 
'grace_period' | 'penalty_applied'
```

### 3. Polymorphic Relations

**Pattern**: `complianceEntityType` + `complianceEntityId`

```typescript
// Task can relate to any entity type
{
  taskId: 'task_123',
  relatedEntityType: 'paye_return',
  relatedEntityId: 'return_456'
}

// Submission can relate to any compliance entity
{
  submissionId: 'sub_789',
  complianceEntityType: 'facility_inspection',
  complianceEntityId: 'inspection_101'
}
```

**Benefits**:
- Single table for all submissions (regardless of regulator)
- Easy to add new entity types
- Flexible querying

**Query Pattern**:
```typescript
const entity = await getEntityByTypeAndId(
  task.relatedEntityType,
  task.relatedEntityId
);
```

### 4. Composite Relations

**Pattern**: Foreign key on multiple fields

```typescript
// Task relates to connection via (organizationId, regulatorId)
obligationTasks.organizationId + obligationTasks.regulatorId
  → organizationRegulatorConnections.organizationId + organizationRegulatorConnections.regulatorId
```

**Benefit**: Fetch connection context for any task without separate query

```typescript
const task = await db.query.obligationTasks.findFirst({
  where: eq(obligationTasks.id, taskId),
  with: {
    regulatorConnection: true, // Automatic join
  },
});

// task.regulatorConnection contains registration number, credentials, etc.
```

---

## Summary

The Bumara database schema is designed with three core principles:

1. **Flexibility**: Add new regulators without code changes via the registry pattern
2. **Scalability**: Hybrid approach balances specialized schemas (for complex regulators) with generic connections (for simple ones)
3. **Control**: Plan-based access control and comprehensive compliance tracking

**Key Innovation**: The `regulators` table acts as a **catalog** that defines:
- What regulators exist
- Who can access them (plan + add-on requirements)
- What tasks they support (dynamic task types)
- How to interact with them (config, API flags)

This architecture enables Bumara to scale from 3 major regulators (ZRA, PACRA, NAPSA) to dozens of specialized regulators (NIHMA, NCC, local councils, etc.) **without significant schema changes**.

The `organization_regulator_connections` table provides a **universal interface** for compliance tracking, while specialized schemas (like `zra_profiles`, `pacra_profiles`) handle complex domain-specific data where needed.

Together, this creates a robust, extensible platform for business compliance management in Zambia.

