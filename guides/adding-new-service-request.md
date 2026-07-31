---
title: "Quick Start: Adding a New Regulator Service Request"
description: "This guide shows you how to add a new service request type for a regulator (e.g., \"Workers Compensation Fund Registration\") in under 30 minutes."
---

This guide shows you how to add a new service request type for a regulator (e.g., "Workers Compensation Fund Registration") in under 30 minutes.

**Target Audience**: Backend developers familiar with the Bumara codebase.

---

## Prerequisites

- [ ] Regulator already exists in `regulators` table (check via backoffice)
- [ ] Regulator fees defined (if payment required) in `regulator_fees` table
- [ ] You have the service requirements documented (fields, documents, fees)

---

## Step-by-Step Guide

### **1. Create Template Seed** (10 minutes)

Create a new file or add to existing regulator seed file:

```typescript
// packages/database/src/seeds/{regulator}-templates.ts

import type {
  ActivationRules,
  DocRequirementConfig,
  PaymentRuleConfig,
  TaskTemplateConfig,
} from "../schema/compliance/obligation-templates";
import type { IntakeFieldSchema } from "../schema/compliance/service-templates";

export interface ServiceTemplateSeed {
  templateKey: string;
  templateVersion: number;
  name: string;
  description: string;
  regulator: "pacra" | "zra" | "napsa" | "nhima" | "wcf" | "other";
  whatIsThis: string;
  whyItMatters: string;
  consequencesOfDelay: string;
  expectedDueInDays: string;
  defaultPriority: "low" | "normal" | "high" | "urgent" | "critical";
  intakeFieldsSchema: IntakeFieldSchema[];
  activationRules: ActivationRules;
  taskTemplateConfigs: TaskTemplateConfig[];
  docRequirementConfigs: DocRequirementConfig[];
  paymentRuleConfig: PaymentRuleConfig;
  billingTag: "included" | "overage";
  executionMode?: "instant" | "task_based";
}

// Example: Workers Compensation Fund Employer Registration
export const WCF_EMPLOYER_REGISTRATION_V1: ServiceTemplateSeed = {
  templateKey: "WCF_EMPLOYER_REGISTRATION_V1",
  templateVersion: 1,
  name: "Employer Registration",
  description: "Register your business as an employer with Workers Compensation Fund",
  regulator: "wcf", // Ensure this exists in regulatorEnum

  // Plain language explanations (shown to tenants)
  whatIsThis:
    "The Workers Compensation Fund (WCF) requires all employers in Zambia to register and contribute to the fund. This covers employees in case of workplace injuries or accidents.",
  whyItMatters:
    "Registration with WCF is a legal requirement. Without it, you cannot legally employ workers, and you risk penalties and fines.",
  consequencesOfDelay:
    "Operating without WCF registration can result in fines up to ZMW 50,000, backdated contributions, and potential closure of your business.",

  expectedDueInDays: "14", // 14 business days expected turnaround
  defaultPriority: "high",

  // Intake form fields (dynamic form generation)
  intakeFieldsSchema: [
    {
      key: "businessName",
      label: "Business Name",
      type: "text",
      required: true,
      placeholder: "Enter registered business name",
    },
    {
      key: "tpin",
      label: "TPIN",
      type: "text",
      required: true,
      placeholder: "1000000000",
      helpText: "Your ZRA Tax Identification Number",
      validation: {
        pattern: "^\\d{10}$",
      },
    },
    {
      key: "numberOfEmployees",
      label: "Number of Employees",
      type: "number",
      required: true,
      helpText: "Current number of full-time and part-time employees",
    },
    {
      key: "industryCategory",
      label: "Industry Category",
      type: "select",
      required: true,
      options: [
        { value: "manufacturing", label: "Manufacturing" },
        { value: "construction", label: "Construction" },
        { value: "retail", label: "Retail & Wholesale" },
        { value: "services", label: "Professional Services" },
        { value: "agriculture", label: "Agriculture" },
        { value: "mining", label: "Mining & Quarrying" },
        { value: "other", label: "Other" },
      ],
    },
    {
      key: "estimatedAnnualPayroll",
      label: "Estimated Annual Payroll (ZMW)",
      type: "number",
      required: true,
      helpText: "Total estimated annual salary expenses",
    },
  ],

  // Activation rules (who can see/use this template)
  activationRules: {
    // Only show to companies and business names (not individuals)
    entityType: ["company", "business_name"],

    // Must have ZRA connection (TPIN required)
    customConditions: [
      {
        field: "connections.zra.status",
        operator: "eq",
        value: "active",
      },
    ],
  },

  // Task workflow (steps tenant must complete)
  taskTemplateConfigs: [
    {
      key: "upload_tpin_certificate",
      title: "Upload TPIN Certificate",
      description: "Upload a copy of your ZRA TPIN Certificate",
      taskType: "upload_document",
      required: true,
      isBlocking: true,
      sequence: 1,
      actionKind: "doc_upload",
      actionRefTemplate: {
        docRequirementGroup: "tpin_certificate",
      },
    },
    {
      key: "upload_employee_list",
      title: "Upload Employee List",
      description: "Upload a list of all current employees (Excel or PDF format)",
      taskType: "upload_document",
      required: true,
      isBlocking: true,
      sequence: 2,
      actionKind: "doc_upload",
      actionRefTemplate: {
        docRequirementGroup: "employee_list",
      },
    },
    {
      key: "fill_registration_form",
      title: "Complete Registration Form",
      description: "Fill in all required business and employer details",
      taskType: "fill_form",
      required: true,
      isBlocking: true,
      sequence: 3,
      actionKind: "form_section",
      actionRefTemplate: {
        formKey: "wcf_employer_registration",
        section: "employer_details",
        href: "/regulators/wcf/service-requests/{serviceRequestId}",
      },
    },
    {
      key: "review_and_confirm",
      title: "Review and Confirm",
      description: "Review all information and confirm submission readiness",
      taskType: "review_approve",
      required: true,
      isBlocking: true,
      sequence: 4,
      actionKind: "confirmation",
      actionRefTemplate: {
        anchor: "confirm_wcf_registration",
      },
    },
  ],

  // Document requirements
  docRequirementConfigs: [
    {
      key: "tpin_certificate",
      name: "TPIN Certificate",
      description: "Copy of your ZRA Tax Identification Number Certificate",
      kind: "source",
      required: true,
    },
    {
      key: "employee_list",
      name: "Employee List",
      description: "Current list of all employees with names, NRCs, positions, and salaries",
      kind: "source",
      required: true,
    },
    {
      key: "certificate_of_incorporation",
      name: "Certificate of Incorporation",
      description: "Business registration certificate",
      kind: "source",
      required: true,
      conditions: {
        entityType: ["company"], // Only required for companies
      },
    },
    {
      key: "business_name_certificate",
      name: "Business Name Certificate",
      description: "Business name registration certificate from PACRA",
      kind: "source",
      required: true,
      conditions: {
        entityType: ["business_name"], // Only required for business names
      },
    },
  ],

  // Payment configuration
  paymentRuleConfig: {
    paymentRequired: true,
    feeKey: "WCF_EMPLOYER_REGISTRATION", // Matches regulator_fees.feeKey
    serviceFee: 25000, // ZMW 250 Bumara service fee (in minor units)
    notes: "WCF registration fee is calculated based on number of employees and industry risk category. Regulator fee will be calculated and added to invoice.",
  },

  billingTag: "included", // Included in subscription plan
  executionMode: "task_based", // Requires tenant to complete tasks (vs. instant submission)
};

// Export all templates for this regulator
export const WCF_SERVICE_TEMPLATES = [
  WCF_EMPLOYER_REGISTRATION_V1,
  // Add more WCF service templates here
];
```

---

### **2. Seed Template to Database** (2 minutes)

Add seed logic to insert the template:

```typescript
// packages/database/src/seeds/seed-wcf-templates.ts

import { eq } from "drizzle-orm";
import { db } from "../client";
import { regulators, serviceTemplates } from "../schema";
import { WCF_SERVICE_TEMPLATES } from "./wcf-templates";

export async function seedWcfServiceTemplates() {
  console.log("🌱 Seeding WCF service templates...");

  // Get WCF regulator
  const wcfRegulator = await db.query.regulators.findFirst({
    where: eq(regulators.code, "wcf"),
  });

  if (!wcfRegulator) {
    console.error("❌ WCF regulator not found. Please seed regulators first.");
    return;
  }

  for (const template of WCF_SERVICE_TEMPLATES) {
    // Check if template already exists
    const existing = await db.query.serviceTemplates.findFirst({
      where: eq(serviceTemplates.templateKey, template.templateKey),
    });

    if (existing) {
      console.log(`  ↻ Updating ${template.name}...`);
      await db
        .update(serviceTemplates)
        .set({
          name: template.name,
          description: template.description,
          regulatorId: wcfRegulator.id,
          regulator: template.regulator,
          whatIsThis: template.whatIsThis,
          whyItMatters: template.whyItMatters,
          consequencesOfDelay: template.consequencesOfDelay,
          expectedDueInDays: template.expectedDueInDays,
          defaultPriority: template.defaultPriority,
          intakeFieldsSchema: template.intakeFieldsSchema as any,
          activationRules: template.activationRules as any,
          taskTemplateConfigs: template.taskTemplateConfigs as any,
          docRequirementConfigs: template.docRequirementConfigs as any,
          paymentRuleConfig: template.paymentRuleConfig as any,
          billingTag: template.billingTag,
          executionMode: template.executionMode ?? "task_based",
          status: "active",
          updatedAt: new Date(),
        })
        .where(eq(serviceTemplates.id, existing.id));
    } else {
      console.log(`  + Creating ${template.name}...`);
      await db.insert(serviceTemplates).values({
        templateKey: template.templateKey,
        templateVersion: template.templateVersion,
        name: template.name,
        description: template.description,
        regulatorId: wcfRegulator.id,
        regulator: template.regulator,
        whatIsThis: template.whatIsThis,
        whyItMatters: template.whyItMatters,
        consequencesOfDelay: template.consequencesOfDelay,
        expectedDueInDays: template.expectedDueInDays,
        defaultPriority: template.defaultPriority,
        intakeFieldsSchema: template.intakeFieldsSchema as any,
        activationRules: template.activationRules as any,
        taskTemplateConfigs: template.taskTemplateConfigs as any,
        docRequirementConfigs: template.docRequirementConfigs as any,
        paymentRuleConfig: template.paymentRuleConfig as any,
        billingTag: template.billingTag,
        executionMode: template.executionMode ?? "task_based",
        status: "active",
        createdAt: new Date(),
        updatedAt: new Date(),
      });
    }
  }

  console.log("✅ WCF service templates seeded successfully");
}

// If running directly
if (import.meta.url === `file://${process.argv[1]}`) {
  seedWcfServiceTemplates()
    .then(() => process.exit(0))
    .catch((error) => {
      console.error("❌ Seed failed:", error);
      process.exit(1);
    });
}
```

Add to main seed file:

```typescript
// packages/database/seed.ts

import { seedWcfServiceTemplates } from "./src/seeds/seed-wcf-templates";

async function main() {
  // ... existing seeds
  await seedWcfServiceTemplates();
}
```

Run the seed:

```bash
pnpm --filter @repo/database db:seed
```

---

### **3. (Optional) Add Custom Handler** (10 minutes)

Only needed if you have complex validation or custom business logic.

```typescript
// packages/api-services/src/domains/regulators/wcf/wcf-employer-registration.handler.ts

import type {
  ServiceContext,
  ServiceDependencies,
} from "../../core/context";
import type {
  ServiceRequestHandler,
  ValidationResult,
  FeeCalculation,
} from "../handler-interface";
import { ServiceError } from "../../core/errors";

export class WcfEmployerRegistrationHandler implements ServiceRequestHandler {
  readonly handlerKey = "WCF_EMPLOYER_REGISTRATION_V1";
  readonly regulatorCode = "wcf";

  /**
   * Custom validation: Check TPIN format and employee count
   */
  async validateIntakeData(
    ctx: ServiceContext,
    deps: ServiceDependencies,
    data: Record<string, unknown>
  ): Promise<ValidationResult> {
    const errors: Array<{ field: string; message: string }> = [];

    // Validate TPIN format (10 digits)
    const tpin = data.tpin as string;
    if (!tpin || !/^\d{10}$/.test(tpin)) {
      errors.push({
        field: "tpin",
        message: "TPIN must be exactly 10 digits",
      });
    }

    // Validate employee count
    const employeeCount = data.numberOfEmployees as number;
    if (!employeeCount || employeeCount < 1) {
      errors.push({
        field: "numberOfEmployees",
        message: "Number of employees must be at least 1",
      });
    }

    // Validate payroll estimate
    const payroll = data.estimatedAnnualPayroll as number;
    if (!payroll || payroll < 0) {
      errors.push({
        field: "estimatedAnnualPayroll",
        message: "Annual payroll must be a positive number",
      });
    }

    if (errors.length > 0) {
      return { valid: false, errors };
    }

    return { valid: true };
  }

  /**
   * Dynamic fee calculation based on employee count and industry
   */
  async calculateFees(
    ctx: ServiceContext,
    deps: ServiceDependencies,
    intakeData: Record<string, unknown>
  ): Promise<FeeCalculation> {
    const employeeCount = intakeData.numberOfEmployees as number;
    const industry = intakeData.industryCategory as string;

    // WCF fee structure (example rates)
    const baseRate = 5000; // ZMW 50 base
    const perEmployeeFee = 2000; // ZMW 20 per employee

    // Risk multiplier by industry
    const riskMultipliers: Record<string, number> = {
      construction: 2.0,
      mining: 2.5,
      manufacturing: 1.5,
      agriculture: 1.3,
      retail: 1.0,
      services: 1.0,
      other: 1.0,
    };

    const multiplier = riskMultipliers[industry] ?? 1.0;
    const regulatorFee = Math.round((baseRate + (employeeCount * perEmployeeFee)) * multiplier);

    const serviceFee = 25000; // ZMW 250 (from template)

    return {
      serviceFee,
      regulatorFee,
      total: serviceFee + regulatorFee,
      breakdown: {
        baseRate,
        perEmployeeFee,
        employeeCount,
        industryMultiplier: multiplier,
      },
    };
  }

  /**
   * Pre-submission: Validate TPIN with ZRA (if API available)
   */
  async beforeSubmission(
    ctx: ServiceContext,
    deps: ServiceDependencies,
    requestId: string
  ): Promise<void> {
    // Example: Call ZRA API to verify TPIN validity
    // const request = await getServiceRequest(ctx, deps, requestId);
    // const tpin = (request.payload as any).tpin;
    //
    // const isValid = await zraApi.verifyTpin(tpin);
    // if (!isValid) {
    //   throw new ServiceError("INVALID_TPIN", "The provided TPIN is not registered with ZRA");
    // }

    deps.logger?.info?.({ requestId }, "Pre-submission validation passed for WCF registration");
  }
}
```

Register the handler:

```typescript
// packages/api-services/src/domains/regulators/handler-registry.ts

import { WcfEmployerRegistrationHandler } from "./wcf/wcf-employer-registration.handler";

// ... existing handlers

export function registerAllHandlers() {
  // PACRA handlers
  registerHandler(new PacraNameClearanceHandler());
  registerHandler(new PacraCompanyRegistrationHandler());

  // WCF handlers
  registerHandler(new WcfEmployerRegistrationHandler());

  // ... other handlers
}
```

Call registration on app startup:

```typescript
// packages/backend/src/index.ts

import { registerAllHandlers } from "@repo/api-services/domains/regulators/handler-registry";

// ... app setup

registerAllHandlers();

// ... start server
```

---

### **4. Frontend Integration** (Auto-generated!)

The service request is now available in the catalog. The frontend will automatically:

1. **Show in catalog** (filtered by activation rules):
   ```tsx
   // apps/app/features/regulators/components/service-catalog.tsx
   // No changes needed - uses useAvailableServiceTemplates hook
   ```

2. **Generate dynamic intake form** (from `intakeFieldsSchema`):
   ```tsx
   // apps/app/features/compliance/components/dynamic-intake-form.tsx
   // Automatically renders fields based on schema
   ```

3. **Create task checklist** (from `taskTemplateConfigs`):
   ```tsx
   // apps/app/features/compliance/components/task-list.tsx
   // Auto-generated from task templates
   ```

**Optional**: Add regulator-specific UI page:

```tsx
// apps/app/app/(authenticated)/(general)/regulators/wcf/employer-registration/page.tsx

import { DynamicIntakeForm } from "@/features/compliance/components/dynamic-intake-form";
import { useCreateServiceRequest } from "@/lib/queries/service-requests";

export default function WcfEmployerRegistrationPage() {
  const createMutation = useCreateServiceRequest();

  const handleSubmit = async (data: Record<string, unknown>) => {
    await createMutation.mutateAsync({
      templateKey: "WCF_EMPLOYER_REGISTRATION_V1",
      intakeData: data,
    });
  };

  return (
    <div>
      <h1>WCF Employer Registration</h1>
      <DynamicIntakeForm
        templateKey="WCF_EMPLOYER_REGISTRATION_V1"
        onSubmit={handleSubmit}
      />
    </div>
  );
}
```

---

### **5. Test the Flow** (5 minutes)

1. **Verify template appears in catalog**:
   - Login as tenant with company entity type
   - Navigate to `/regulators/wcf/services` (or generic `/services` page)
   - Confirm "Employer Registration" appears

2. **Create a service request**:
   - Fill in intake form
   - Submit
   - Verify service request created with status `pending_data`

3. **Complete tasks**:
   - Upload TPIN certificate
   - Upload employee list
   - Fill registration form
   - Confirm submission

4. **Verify status transitions**:
   - Should move to `ready_for_submission` when all tasks done
   - If payment required: moves to `awaiting_payment`

5. **Test payment flow** (if applicable):
   - Pay via Bumara payment gateway
   - Verify status moves to `ready_for_submission`

---

## Advanced: Complex Scenarios

### **Scenario 1: Conditional Fields**

Show/hide fields based on previous answers:

```typescript
intakeFieldsSchema: [
  {
    key: "hasExistingRegistration",
    label: "Do you have an existing WCF registration?",
    type: "boolean",
    required: true,
  },
  {
    key: "existingRegistrationNumber",
    label: "Existing Registration Number",
    type: "text",
    required: false,
    conditions: {
      dependsOn: "hasExistingRegistration",
      showIf: { operator: "eq", value: true },
    },
  },
]
```

### **Scenario 2: Multi-Step Wizard**

For complex forms, use intake form sections:

```typescript
intakeFieldsSchema: [
  // Section 1: Business Details
  { key: "_section_business", label: "Business Details", type: "section" },
  { key: "businessName", label: "Business Name", type: "text", required: true },
  { key: "tpin", label: "TPIN", type: "text", required: true },

  // Section 2: Employee Information
  { key: "_section_employees", label: "Employee Information", type: "section" },
  { key: "numberOfEmployees", label: "Number of Employees", type: "number", required: true },
  { key: "employeeList", label: "Employee List", type: "table", required: true },
]
```

### **Scenario 3: Dynamic Task Generation**

Generate tasks based on intake data (e.g., one task per director):

```typescript
// In custom handler
async generateDynamicTasks(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  request: ServiceRequest,
  intakeData: Record<string, unknown>
): Promise<TaskTemplateConfig[]> {
  const directors = (intakeData.directors as any[]) ?? [];

  return directors.map((director, index) => ({
    key: `upload_director_id_${index}`,
    title: `Upload ID for ${director.name}`,
    description: `Upload NRC or passport for ${director.name}`,
    taskType: "upload_document",
    required: true,
    sequence: 10 + index,
    actionKind: "doc_upload",
    actionRefTemplate: {
      docRequirementGroup: `director_${index}_id`,
    },
  }));
}
```

---

## Troubleshooting

### **Template not showing in catalog**

**Cause**: Activation rules not met
**Fix**: Check `activationRules` in template. Verify org meets conditions:
```sql
SELECT
  o.id,
  o.entity_type,
  s.plan_tier,
  rc.status AS connection_status
FROM organizations o
LEFT JOIN subscriptions s ON o.id = s.organization_id
LEFT JOIN regulator_connections rc ON o.id = rc.organization_id AND rc.regulator_id = 'wcf-uuid'
WHERE o.id = 'your-org-id';
```

### **Tasks not auto-completing**

**Cause**: `completionTrigger` mismatch
**Fix**: Ensure `actionKind` and `actionRefTemplate` match task completion events:
- `doc_upload` → Matches when document with `requirementKey` is uploaded
- `form_section` → Matches when form section data is saved
- `confirmation` → Requires manual completion

### **Payment not blocking submission**

**Cause**: `paymentRuleConfig.paymentRequired` is `false`
**Fix**: Set to `true` and specify `feeKey` or `serviceFee`

---

## Next Steps

- [ ] Add integration tests for the new template
- [ ] Update docs with service request guide for end users
- [ ] Add Slack notification when service request is created
- [ ] Monitor usage metrics for the new service

---

## Resources

- [Service Request System Architecture](/ARCHITECTURE/service-request-system)
- Catalog Module Patterns
- [CLAUDE.md Guidelines](https://github.com/bumara-dev/bumara/tree/main/.claude/CLAUDE.md)
