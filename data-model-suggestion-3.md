# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Loan Origination System · Created: 2026-05-20

## Philosophy

This model uses a compact set of relational tables for the structural backbone — parties, applications, decisions, documents — but relies on JSONB columns to absorb the variability that makes loan origination schemas complex. Different loan products (mortgage, consumer, commercial, SBA), different jurisdictions (state-specific fields), different collateral types (real estate, vehicle, equipment), and different compliance regimes (TRID for mortgages, not for consumer) all produce field sets that vary dramatically. Rather than encoding every possible field as a nullable column or creating per-product table hierarchies, the hybrid model puts stable, queryable fields in typed columns and puts variable, product-specific data in JSONB.

This approach is inspired by how modern fintech LOS platforms (LendFoundry, DigiFi) achieve rapid product iteration: the core data model handles the universal lending workflow, while product-specific configuration and data live in flexible structures. PostgreSQL's JSONB support — GIN indexes, containment operators, JSONPath queries — makes this practical without sacrificing query performance for the fields that matter.

The result is a schema with fewer tables (~20) that can support multiple lending products without migrations, while still enforcing referential integrity on the structural relationships (party-to-application, application-to-decision) and providing indexed access to the most-queried fields.

**Best for:** Fintech lenders and multi-product platforms that need to launch new loan products quickly, support jurisdiction-specific fields without migrations, and iterate rapidly during MVP development without accumulating migration debt.

**Trade-offs:**
- (+) Far fewer tables (~20 vs ~50+) — faster development, simpler operations
- (+) New loan products can be launched without schema migrations
- (+) Jurisdiction-specific and product-specific fields handled without nullable column sprawl
- (+) JSONB fields can be validated at the application layer using JSON Schema
- (+) GIN indexes on JSONB provide fast queries for common access patterns
- (+) Natural fit for API-first design — request/response bodies map directly to JSONB
- (-) No database-level type enforcement on JSONB fields — bugs can insert malformed data
- (-) Complex JSONB queries can be slower than indexed relational columns
- (-) Reporting tools (BI, SQL analysts) find JSONB harder to work with than flat columns
- (-) MISMO/ULAD field mapping requires extraction logic at the application layer
- (-) Schema documentation must be maintained separately (JSON Schema files)
- (-) Risk of "schema soup" if JSONB usage is not disciplined

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| MISMO v3.4/v3.6 | JSONB field names follow MISMO container naming for mortgage products |
| ULAD/ULDD | Mortgage-specific JSONB payloads validated against ULAD JSON Schema |
| JSON Schema (Draft 2020-12) | Product-specific JSONB structures validated using JSON Schema definitions |
| HMDA (Regulation C) | HMDA fields stored in a dedicated JSONB column on the application, validated against FFIEC schema |
| TRID | Disclosure data stored as JSONB with product-specific tolerance rules |
| ISO 3166 | Address jurisdiction codes in relational columns for indexed queries |
| ISO 20022 | Payment/disbursement event data in JSONB follows ISO 20022 message naming |
| ECOA / FCRA | Adverse action reasons and credit data in structured JSONB with schema validation |
| OpenAPI 3.1 | JSONB structures align with API request/response schemas |

---

## Product Configuration

```sql
-- Loan product definitions with JSONB configuration
CREATE TABLE loan_product (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_code VARCHAR(30) NOT NULL UNIQUE,
    product_name VARCHAR(100) NOT NULL,
    product_category VARCHAR(30) NOT NULL, -- mortgage, consumer, commercial, sba, heloc
    -- Relational: universal constraints
    min_amount DECIMAL(14,2),
    max_amount DECIMAL(14,2),
    min_term_months INT,
    max_term_months INT,
    -- JSONB: product-specific configuration
    product_config JSONB NOT NULL DEFAULT '{}',
    -- Example for a mortgage product:
    -- {
    --   "requires_appraisal": true,
    --   "requires_title_search": true,
    --   "trid_applicable": true,
    --   "hmda_reportable": true,
    --   "gse_eligible": true,
    --   "occupancy_types": ["primary_residence", "second_home", "investment"],
    --   "max_ltv": 0.95,
    --   "max_dti": 0.50,
    --   "required_documents": ["pay_stub", "w2", "tax_return", "bank_statement"],
    --   "available_rate_types": ["fixed", "adjustable"],
    --   "aus_systems": ["du", "lpa"]
    -- }
    --
    -- Example for a consumer auto loan:
    -- {
    --   "requires_appraisal": false,
    --   "requires_vehicle_valuation": true,
    --   "trid_applicable": false,
    --   "hmda_reportable": false,
    --   "max_ltv": 1.20,
    --   "max_dti": 0.45,
    --   "required_documents": ["pay_stub", "dl_copy"],
    --   "collateral_types": ["vehicle"],
    --   "available_rate_types": ["fixed"]
    -- }
    -- Schema validation
    field_schema JSONB,                  -- JSON Schema defining required application fields
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Party Management

```sql
-- Parties: core identity fields relational, variable data in JSONB
CREATE TABLE party (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_type VARCHAR(20) NOT NULL CHECK (party_type IN ('individual', 'organisation')),
    -- Relational: always-queried fields
    display_name VARCHAR(255) NOT NULL,  -- computed: "first last" or legal_name
    email VARCHAR(255),
    phone_primary VARCHAR(20),
    ssn_last_four VARCHAR(4),
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    -- JSONB: all party details
    party_data JSONB NOT NULL DEFAULT '{}',
    -- Individual example:
    -- {
    --   "first_name": "Jane",
    --   "middle_name": "Marie",
    --   "last_name": "Smith",
    --   "suffix": null,
    --   "date_of_birth": "1988-03-15",
    --   "ssn_encrypted_ref": "vault://ssn/uuid",
    --   "citizenship": "US",
    --   "marital_status": "married",
    --   "dependents_count": 2,
    --   "addresses": [
    --     {
    --       "type": "current",
    --       "street": "123 Oak Ave",
    --       "city": "Austin",
    --       "state": "TX",
    --       "postal_code": "78701",
    --       "county_fips": "48453",
    --       "census_tract": "48453001400",
    --       "residence_type": "own",
    --       "years_at_address": 4.5
    --     }
    --   ],
    --   "employment": [
    --     {
    --       "employer_name": "Acme Corp",
    --       "position_title": "Software Engineer",
    --       "employment_type": "W2",
    --       "start_date": "2021-06-01",
    --       "is_current": true,
    --       "monthly_income": 9500.00,
    --       "verified": true,
    --       "verification_source": "payroll_api",
    --       "verified_at": "2026-05-18T14:30:00Z"
    --     }
    --   ],
    --   "income_sources": [
    --     {
    --       "type": "rental",
    --       "monthly_amount": 1200.00,
    --       "verified": false
    --     }
    --   ],
    --   "assets": [
    --     {
    --       "type": "checking",
    --       "institution": "Chase",
    --       "balance": 45000.00,
    --       "verified": true
    --     }
    --   ],
    --   "liabilities": [
    --     {
    --       "type": "auto_loan",
    --       "creditor": "Capital One",
    --       "balance": 12000.00,
    --       "monthly_payment": 450.00,
    --       "months_remaining": 28
    --     }
    --   ]
    -- }
    --
    -- Organisation example:
    -- {
    --   "legal_name": "Smith Holdings LLC",
    --   "dba_name": "Smith Properties",
    --   "ein_encrypted_ref": "vault://ein/uuid",
    --   "lei": "5493001KJTIIGC8Y1R12",
    --   "entity_type": "LLC",
    --   "state_of_formation": "DE",
    --   "date_of_formation": "2018-01-15",
    --   "officers": [
    --     {"name": "Jane Smith", "title": "Managing Member", "ownership_pct": 100}
    --   ]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_party_type ON party(party_type);
CREATE INDEX idx_party_name ON party(display_name);
CREATE INDEX idx_party_email ON party(email);
CREATE INDEX idx_party_ssn4 ON party(ssn_last_four) WHERE ssn_last_four IS NOT NULL;
-- GIN index for JSONB queries (e.g., find by employer, by address state)
CREATE INDEX idx_party_data ON party USING GIN (party_data jsonb_path_ops);
```

---

## Loan Application

```sql
CREATE TABLE loan_application (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_number VARCHAR(30) NOT NULL UNIQUE,
    loan_product_id UUID NOT NULL REFERENCES loan_product(id),
    -- Relational: always-queried, always-indexed fields
    status VARCHAR(30) NOT NULL DEFAULT 'draft',
    channel VARCHAR(30) NOT NULL,
    requested_amount DECIMAL(14,2) NOT NULL,
    approved_amount DECIMAL(14,2),
    loan_term_months INT NOT NULL,
    loan_purpose VARCHAR(50) NOT NULL,
    -- Assignment (relational for joins)
    loan_officer_id UUID REFERENCES staff(id),
    processor_id UUID REFERENCES staff(id),
    underwriter_id UUID REFERENCES staff(id),
    branch_id UUID REFERENCES branch(id),
    -- Key dates (relational for filtering)
    submitted_at TIMESTAMPTZ,
    decision_at TIMESTAMPTZ,
    funded_at TIMESTAMPTZ,
    closed_at TIMESTAMPTZ,
    -- JSONB: product-specific application data
    application_data JSONB NOT NULL DEFAULT '{}',
    -- Mortgage example:
    -- {
    --   "occupancy_type": "primary_residence",
    --   "interest_rate_type": "fixed",
    --   "requested_rate": 6.500,
    --   "approved_rate": 6.375,
    --   "property_type": "single_family",
    --   "construction_method": "site_built",
    --   "refinance_type": null,
    --   "subordinate_financing": false,
    --   "seller_concessions": 5000.00,
    --   "aus_results": {
    --     "du": {"case_number": "DU-123456", "recommendation": "approve_eligible"},
    --     "lpa": null
    --   }
    -- }
    --
    -- Auto loan example:
    -- {
    --   "interest_rate_type": "fixed",
    --   "requested_rate": 5.990,
    --   "vehicle_new_or_used": "used",
    --   "vehicle_year": 2024,
    --   "vehicle_make": "Toyota",
    --   "vehicle_model": "Camry",
    --   "vehicle_vin": "4T1BF1FK5FU123456",
    --   "dealer_name": "Austin Toyota",
    --   "dealer_id": "uuid"
    -- }
    -- JSONB: compliance data (product-specific)
    compliance_data JSONB NOT NULL DEFAULT '{}',
    -- Mortgage compliance example:
    -- {
    --   "hmda_reportable": true,
    --   "trid_applicable": true,
    --   "hmda": {
    --     "lei": "5493001KJTIIGC8Y1R12",
    --     "uli": "5493001KJTIIGC8Y1R1220260520001",
    --     "action_taken": 1,
    --     "loan_type": 1,
    --     "preapproval": 2,
    --     "applicant_ethnicity_1": 4,
    --     "applicant_race_1": 5,
    --     "applicant_sex": 2,
    --     "rate_spread": 1.25,
    --     "denial_reasons": []
    --   },
    --   "trid": {
    --     "le_issued_date": "2026-05-18",
    --     "le_received_date": "2026-05-18",
    --     "cd_issued_date": null,
    --     "waiting_period_end": "2026-05-21",
    --     "tolerance_status": "compliant"
    --   }
    -- }
    -- Timestamps
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_app_status ON loan_application(status);
CREATE INDEX idx_app_officer ON loan_application(loan_officer_id);
CREATE INDEX idx_app_submitted ON loan_application(submitted_at);
CREATE INDEX idx_app_product ON loan_application(loan_product_id);
-- GIN index for JSONB queries
CREATE INDEX idx_app_data ON loan_application USING GIN (application_data jsonb_path_ops);
CREATE INDEX idx_app_compliance ON loan_application USING GIN (compliance_data jsonb_path_ops);

-- Junction: parties to applications
CREATE TABLE application_party (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    party_id UUID NOT NULL REFERENCES party(id),
    role VARCHAR(30) NOT NULL,           -- borrower, co_borrower, guarantor, co_signer
    role_order INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (application_id, party_id, role)
);

CREATE INDEX idx_app_party_app ON application_party(application_id);
CREATE INDEX idx_app_party_party ON application_party(party_id);
```

---

## Collateral

```sql
CREATE TABLE collateral (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    collateral_type VARCHAR(30) NOT NULL, -- real_estate, vehicle, equipment, other
    -- Relational: universal valuation fields
    estimated_value DECIMAL(14,2),
    appraised_value DECIMAL(14,2),
    ltv_ratio DECIMAL(6,4),
    lien_position INT DEFAULT 1,
    -- JSONB: type-specific collateral data
    collateral_data JSONB NOT NULL DEFAULT '{}',
    -- Real estate example:
    -- {
    --   "property_type": "single_family",
    --   "address": {
    --     "street": "456 Elm St",
    --     "city": "Austin",
    --     "state": "TX",
    --     "postal_code": "78702",
    --     "county": "Travis"
    --   },
    --   "year_built": 2015,
    --   "square_footage": 2200,
    --   "lot_size_acres": 0.25,
    --   "bedrooms": 4,
    --   "bathrooms": 2.5,
    --   "appraisal": {
    --     "type": "full",
    --     "date": "2026-05-10",
    --     "appraiser": "Smith Appraisal Group",
    --     "report_ref": "doc-uuid"
    --   },
    --   "title": {
    --     "status": "clear",
    --     "company": "First American Title",
    --     "commitment_date": "2026-05-12"
    --   },
    --   "insurance_verified": true,
    --   "flood_zone": "X",
    --   "flood_insurance_required": false
    -- }
    --
    -- Vehicle example:
    -- {
    --   "year": 2024,
    --   "make": "Toyota",
    --   "model": "Camry",
    --   "trim": "SE",
    --   "vin": "4T1BF1FK5FU123456",
    --   "mileage": 12000,
    --   "condition": "excellent",
    --   "nada_value": 28500.00,
    --   "kbb_value": 27800.00
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_collateral_app ON collateral(application_id);
CREATE INDEX idx_collateral_type ON collateral(collateral_type);
CREATE INDEX idx_collateral_data ON collateral USING GIN (collateral_data jsonb_path_ops);
```

---

## Credit & Underwriting

```sql
CREATE TABLE credit_report (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_id UUID NOT NULL REFERENCES party(id),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    bureau VARCHAR(20) NOT NULL,
    report_type VARCHAR(20) NOT NULL,
    permissible_purpose VARCHAR(50) NOT NULL,
    pull_date TIMESTAMPTZ NOT NULL,
    expiration_date DATE,
    credit_score INT,
    score_model VARCHAR(30),
    -- JSONB: full report data (encrypted reference or summary)
    report_summary JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "total_accounts": 12,
    --   "open_accounts": 8,
    --   "total_balance": 45200.00,
    --   "total_monthly_payment": 1850.00,
    --   "collections_count": 0,
    --   "public_records_count": 0,
    --   "inquiries_last_6mo": 2,
    --   "oldest_account_date": "2016-03-01",
    --   "derogatory_marks": 0,
    --   "tradelines": [
    --     {"type": "mortgage", "balance": 180000, "payment": 1200, "status": "current"},
    --     {"type": "auto", "balance": 12000, "payment": 450, "status": "current"}
    --   ]
    -- }
    report_data_encrypted BYTEA,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_credit_party ON credit_report(party_id);
CREATE INDEX idx_credit_app ON credit_report(application_id);

-- Underwriting decisions
CREATE TABLE underwriting_decision (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    decision_type VARCHAR(30) NOT NULL,  -- automated, manual, override
    decision VARCHAR(20) NOT NULL,       -- approve, deny, refer, conditionally_approve
    decided_by_id UUID REFERENCES staff(id),
    decided_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_current BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: decision details (varies by decision type and product)
    decision_data JSONB NOT NULL DEFAULT '{}',
    -- Automated decision example:
    -- {
    --   "model_id": "credit_decision_v3",
    --   "model_version": "v3.2.1",
    --   "risk_metrics": {
    --     "dti_ratio": 0.3850,
    --     "housing_ratio": 0.2800,
    --     "ltv_ratio": 0.8000,
    --     "credit_score": 742,
    --     "residual_income": 3200.00,
    --     "risk_grade": "A2"
    --   },
    --   "conditions": [
    --     {"type": "prior_to_closing", "category": "income",
    --      "description": "Verify most recent pay stub", "status": "outstanding"},
    --     {"type": "prior_to_closing", "category": "title",
    --      "description": "Clear title search", "status": "outstanding"}
    --   ],
    --   "adverse_action_reasons": [],
    --   "explainability": {
    --     "top_factors": [
    --       {"factor": "credit_score", "impact": "positive"},
    --       {"factor": "dti_ratio", "impact": "neutral"}
    --     ]
    --   }
    -- }
    --
    -- Denial decision example:
    -- {
    --   "risk_metrics": { "dti_ratio": 0.55, "credit_score": 580 },
    --   "adverse_action_reasons": [
    --     {"code": "R001", "text": "Amount owed on accounts is too high"},
    --     {"code": "R014", "text": "Length of credit history is too short"}
    --   ],
    --   "natural_language_explanation": "Your application was not approved because...",
    --   "notice_delivery": "mail_and_email"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_uw_decision_app ON underwriting_decision(application_id);
CREATE INDEX idx_uw_decision_current ON underwriting_decision(application_id) WHERE is_current = true;
CREATE INDEX idx_uw_decision_data ON underwriting_decision USING GIN (decision_data jsonb_path_ops);
```

---

## Documents

```sql
CREATE TABLE document (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    party_id UUID REFERENCES party(id),
    document_type VARCHAR(50) NOT NULL,
    document_category VARCHAR(30) NOT NULL,
    file_name VARCHAR(255) NOT NULL,
    mime_type VARCHAR(100),
    storage_path VARCHAR(500) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'uploaded',
    version INT NOT NULL DEFAULT 1,
    supersedes_id UUID REFERENCES document(id),
    -- JSONB: extraction results and metadata
    extraction_data JSONB NOT NULL DEFAULT '{}',
    -- AI extraction example (pay stub):
    -- {
    --   "ocr_processed": true,
    --   "ai_extracted": true,
    --   "confidence": 0.94,
    --   "extracted_fields": {
    --     "employer_name": "Acme Corp",
    --     "employee_name": "Jane Smith",
    --     "pay_period_start": "2026-04-16",
    --     "pay_period_end": "2026-04-30",
    --     "gross_pay": 4750.00,
    --     "net_pay": 3420.00,
    --     "ytd_gross": 42750.00,
    --     "deductions": {
    --       "federal_tax": 712.50,
    --       "state_tax": 285.00,
    --       "health_insurance": 180.00,
    --       "retirement_401k": 152.50
    --     }
    --   },
    --   "validation": {
    --     "name_matches_application": true,
    --     "income_consistent_with_stated": true,
    --     "anomalies_detected": []
    --   }
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_doc_app ON document(application_id);
CREATE INDEX idx_doc_type ON document(document_type);
CREATE INDEX idx_doc_status ON document(status);
```

---

## eSignatures

```sql
CREATE TABLE esignature_envelope (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    provider VARCHAR(20) NOT NULL,
    external_envelope_id VARCHAR(100) NOT NULL,
    status VARCHAR(30) NOT NULL,
    envelope_data JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "recipients": [
    --     {"party_id": "uuid", "role": "borrower", "order": 1,
    --      "status": "signed", "signed_at": "2026-05-19T10:30:00Z",
    --      "ip_address": "192.168.1.50"},
    --     {"party_id": "uuid", "role": "co_borrower", "order": 2,
    --      "status": "pending"}
    --   ],
    --   "documents": ["doc-uuid-1", "doc-uuid-2"],
    --   "sent_at": "2026-05-19T09:00:00Z"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_esig_app ON esignature_envelope(application_id);
```

---

## Workflow & Tasks

```sql
CREATE TABLE workflow_template (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    product_category VARCHAR(30) NOT NULL,
    -- JSONB: the entire workflow definition
    workflow_definition JSONB NOT NULL,
    -- {
    --   "steps": [
    --     {"name": "Application Review", "order": 1, "role": "processor",
    --      "sla_hours": 24, "required": true,
    --      "auto_actions": ["assign_to_processor"]},
    --     {"name": "Credit Pull", "order": 2, "role": "processor",
    --      "sla_hours": 4, "required": true,
    --      "auto_actions": ["pull_credit_tri_merge"]},
    --     {"name": "Document Collection", "order": 3, "role": "processor",
    --      "sla_hours": 72, "required": true},
    --     {"name": "Underwriting", "order": 4, "role": "underwriter",
    --      "sla_hours": 48, "required": true},
    --     {"name": "Closing", "order": 5, "role": "closer",
    --      "sla_hours": 24, "required": true}
    --   ]
    -- }
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE task (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    task_type VARCHAR(50) NOT NULL,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    assigned_to_id UUID REFERENCES staff(id),
    priority VARCHAR(10) NOT NULL DEFAULT 'normal',
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    due_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    -- JSONB: task-specific data
    task_data JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_task_assigned ON task(assigned_to_id, status);
CREATE INDEX idx_task_app ON task(application_id);
CREATE INDEX idx_task_due ON task(due_at) WHERE status IN ('pending', 'in_progress');
```

---

## Fees

```sql
CREATE TABLE loan_fee (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    fee_type VARCHAR(50) NOT NULL,
    fee_name VARCHAR(100) NOT NULL,
    amount DECIMAL(12,2) NOT NULL,
    paid_by VARCHAR(20) NOT NULL,
    tolerance_category VARCHAR(20),      -- TRID tolerance (mortgage only)
    -- JSONB: fee details
    fee_data JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "vendor_name": "Smith Appraisal",
    --   "vendor_id": "uuid",
    --   "is_financed": false,
    --   "poc": true,
    --   "original_amount": 550.00,
    --   "tolerance_cure": 0.00
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fee_app ON loan_fee(application_id);
```

---

## Organisation, Staff & RBAC

```sql
CREATE TABLE organisation (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    lei VARCHAR(20),
    nmls_id VARCHAR(20),
    organisation_type VARCHAR(30) NOT NULL,
    org_data JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE branch (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    branch_name VARCHAR(100) NOT NULL,
    branch_code VARCHAR(20),
    nmls_id VARCHAR(20),
    address JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE staff (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    branch_id UUID REFERENCES branch(id),
    email VARCHAR(255) NOT NULL UNIQUE,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    nmls_id VARCHAR(20),
    role VARCHAR(30) NOT NULL,
    permissions JSONB NOT NULL DEFAULT '[]',
    -- ["view_applications", "edit_applications", "approve_loans", "pull_credit"]
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_staff_org ON staff(organisation_id);
CREATE INDEX idx_staff_role ON staff(role);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    action VARCHAR(20) NOT NULL,
    changes JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "fields_changed": ["status", "approved_amount"],
    --   "before": {"status": "underwriting", "approved_amount": null},
    --   "after": {"status": "approved", "approved_amount": 340000.00}
    -- }
    performed_by_id UUID,
    performed_by_type VARCHAR(20) NOT NULL,
    ip_address INET,
    performed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_time ON audit_log(performed_at);
```

---

## Notifications

```sql
CREATE TABLE notification (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID REFERENCES loan_application(id),
    recipient_id UUID,                   -- party or staff UUID
    recipient_type VARCHAR(10),          -- party, staff
    channel VARCHAR(20) NOT NULL,
    template_code VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    notification_data JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "subject": "Your loan application status update",
    --   "body": "...",
    --   "sent_at": "2026-05-19T10:00:00Z",
    --   "delivered_at": "2026-05-19T10:00:05Z",
    --   "read_at": null
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notif_app ON notification(application_id);
CREATE INDEX idx_notif_status ON notification(status) WHERE status = 'pending';
```

---

## JSONB Query Examples

```sql
-- Find all mortgage applications with LTV > 0.90
SELECT id, application_number, requested_amount,
       application_data->>'occupancy_type' AS occupancy
FROM loan_application la
JOIN collateral c ON c.application_id = la.id
WHERE la.loan_product_id = (SELECT id FROM loan_product WHERE product_code = 'CONV_30_FIXED')
  AND c.ltv_ratio > 0.90;

-- Find all parties with current employment at a specific employer
SELECT id, display_name
FROM party
WHERE party_data @> '{"employment": [{"employer_name": "Acme Corp", "is_current": true}]}';

-- Extract HMDA data for annual reporting (mortgage applications only)
SELECT
    compliance_data->'hmda'->>'lei' AS lei,
    compliance_data->'hmda'->>'uli' AS uli,
    (compliance_data->'hmda'->>'action_taken')::int AS action_taken,
    (compliance_data->'hmda'->>'loan_type')::int AS loan_type,
    requested_amount AS loan_amount
FROM loan_application
WHERE compliance_data @> '{"hmda_reportable": true}'
  AND EXTRACT(YEAR FROM submitted_at) = 2026;

-- Find decisions that used a specific model version
SELECT la.application_number, ud.decision, ud.decided_at,
       ud.decision_data->'risk_metrics'->>'risk_grade' AS risk_grade
FROM underwriting_decision ud
JOIN loan_application la ON la.id = ud.application_id
WHERE ud.decision_data @> '{"model_id": "credit_decision_v3", "model_version": "v3.2.1"}'
  AND ud.is_current = true;

-- Find all documents with low AI extraction confidence
SELECT d.id, d.document_type, d.file_name,
       (d.extraction_data->>'confidence')::decimal AS confidence,
       la.application_number
FROM document d
JOIN loan_application la ON la.id = d.application_id
WHERE (d.extraction_data->>'confidence')::decimal < 0.80
  AND d.extraction_data->>'ai_extracted' = 'true';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Product Configuration | 1 | loan_product (with JSONB config and field_schema) |
| Party Management | 1 | party (addresses, employment, assets, liabilities in JSONB) |
| Loan Application | 2 | loan_application, application_party |
| Collateral | 1 | collateral (type-specific data in JSONB) |
| Credit & Underwriting | 2 | credit_report, underwriting_decision (conditions in JSONB) |
| Documents & eSignatures | 2 | document, esignature_envelope (recipients in JSONB) |
| Workflow & Tasks | 2 | workflow_template, task |
| Fees | 1 | loan_fee |
| Organisation & Staff | 3 | organisation, branch, staff |
| Audit | 1 | audit_log |
| Notifications | 1 | notification |
| **Total** | **17** | Significantly fewer tables than normalised approach |

---

## Key Design Decisions

1. **JSONB for product-specific variability, relational for universal structure**: Fields that exist on every application regardless of product (amount, term, status, dates, assignments) are relational columns. Fields that vary by product (occupancy_type for mortgages, VIN for auto loans, SBA form data for SBA loans) live in JSONB. This eliminates nullable column sprawl and per-product table hierarchies.

2. **Party data consolidated into a single JSONB column**: Addresses, employment, income, assets, and liabilities all live inside the party's `party_data` JSONB. This reduces five normalised tables to zero additional tables, at the cost of requiring application-layer validation and more complex update queries.

3. **Conditions embedded in decision JSONB rather than a separate table**: Underwriting conditions (stipulations) are part of the decision record's JSONB, making the decision self-contained. Condition status changes are tracked by updating the JSONB array. This trades query convenience for simplicity.

4. **GIN indexes on all major JSONB columns**: PostgreSQL GIN indexes with `jsonb_path_ops` enable fast containment queries (`@>`) on JSONB data. This supports queries like "find all applications using model v3.2.1" without full table scans.

5. **JSON Schema validation at the product level**: The `loan_product.field_schema` column stores a JSON Schema definition for what fields are required in `application_data` for that product. The application layer validates incoming data against this schema, compensating for the lack of database-level type enforcement.

6. **Compliance data in a dedicated JSONB column**: HMDA, TRID, and product-specific compliance data live in `compliance_data` rather than separate tables. This keeps compliance data co-located with the application while allowing different compliance regimes per product.

7. **Permissions as a JSONB array on staff**: Rather than a full permission/role_permission table structure, permissions are stored as a simple JSON array on the staff record. This is appropriate for a system with a manageable number of permission types and simplifies the RBAC implementation.

8. **Audit log uses JSONB for before/after snapshots**: Change records capture the full before and after state of changed fields as JSONB, avoiding the field_name/old_value/new_value pattern that requires one row per changed field.

9. **Document extraction results in JSONB**: AI/OCR extraction outputs (extracted fields, confidence scores, validation results) are stored as JSONB on the document record. This allows the extraction schema to evolve as AI models improve without requiring migrations.

10. **17 tables vs 30+ in normalised model**: The reduction in table count directly translates to fewer migrations, simpler ORM mappings, faster development velocity, and easier onboarding for new developers — at the cost of weaker database-level constraints on variable data.
