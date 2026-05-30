# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Loan Origination System · Created: 2026-05-20

## Philosophy

This model follows the entity-centric architecture pattern where the database is organised around the real actors in a lending transaction — parties (borrowers, guarantors, co-signers), assets (collateral, properties), and products (loan types) — rather than around a monolithic application form. Each entity has its own set of tables, relationships, and lifecycle tracking, and the loan application acts as a container that binds entities together for a specific deal.

The design aligns closely with the MISMO Reference Model v3.4/v3.6, which organises mortgage data into hierarchical containers (DEAL > LOANS > LOAN > PARTIES > PARTY, etc.) with XLink-style relationships connecting entities across the hierarchy. It also draws on Microsoft's Common Data Model for Financial Services (LoanOnboardingDataModel), which defines standard entities for LoanApplication, Employment, Collateral, and Bank. Apache Fineract's open-source loan management schema provides a proven reference for multi-product lending tables.

This is the most traditional approach: every concept gets its own table, foreign keys enforce referential integrity, and the schema is self-documenting. It favours data integrity and complex cross-entity queries at the cost of more tables and more rigid schema evolution.

**Best for:** Regulated institutions (banks, credit unions) that need demonstrable data integrity, standards-aligned reporting (HMDA, TRID), and complex cross-entity queries for portfolio analytics.

**Trade-offs:**
- (+) Strong referential integrity — data inconsistencies are caught at the database level
- (+) Standards-aligned — maps directly to MISMO containers and ULAD/ULDD data points
- (+) Excellent for complex joins and analytical queries
- (+) Self-documenting schema — new developers can understand the domain from the DDL
- (-) High table count (~50-65 tables) increases migration complexity
- (-) Schema changes require migrations even for minor field additions
- (-) Multi-product flexibility requires careful use of type columns and junction tables
- (-) Can be slower for write-heavy workflows due to normalisation overhead

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| MISMO v3.4/v3.6 Reference Model | Table hierarchy mirrors MISMO container structure (DEAL > LOAN > PARTY > ROLE) |
| ULAD (Uniform Loan Application Dataset) | Borrower/application fields map to Form 1003 data points |
| ULDD (Uniform Loan Delivery Dataset) | Loan delivery fields support GSE submission |
| HMDA (Regulation C) | Dedicated `hmda_data` table captures all 110 required data fields |
| TRID (Regulation Z/X) | Disclosure tracking tables with tolerance calculations |
| ISO 3166 | Country and subdivision codes for address and jurisdiction modelling |
| ISO 17442 (LEI) | Legal Entity Identifier field on organisation records |
| ISO 20022 | Payment message structure alignment for disbursement/funding tables |
| ECOA (Regulation B) | Adverse action reason codes stored with decision records |
| FCRA (Regulation V) | Credit report pull tracking with permissible purpose codes |

---

## Party Management

```sql
-- Core party table: persons and organisations
CREATE TABLE party (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_type VARCHAR(20) NOT NULL CHECK (party_type IN ('individual', 'organisation')),
    -- Individual fields
    first_name VARCHAR(100),
    middle_name VARCHAR(100),
    last_name VARCHAR(100),
    suffix VARCHAR(20),
    date_of_birth DATE,
    ssn_encrypted BYTEA,               -- encrypted at rest, SSN for credit pulls
    ssn_last_four VARCHAR(4),           -- searchable last 4
    -- Organisation fields
    legal_name VARCHAR(255),
    dba_name VARCHAR(255),
    ein_encrypted BYTEA,
    lei VARCHAR(20),                    -- ISO 17442 Legal Entity Identifier
    entity_type VARCHAR(50),            -- LLC, corporation, sole_proprietorship, trust
    state_of_formation VARCHAR(2),
    date_of_formation DATE,
    -- Contact
    email VARCHAR(255),
    phone_primary VARCHAR(20),
    phone_secondary VARCHAR(20),
    -- Status
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_party_type ON party(party_type);
CREATE INDEX idx_party_last_name ON party(last_name) WHERE party_type = 'individual';
CREATE INDEX idx_party_legal_name ON party(legal_name) WHERE party_type = 'organisation';
CREATE INDEX idx_party_ssn_last_four ON party(ssn_last_four) WHERE ssn_last_four IS NOT NULL;
CREATE INDEX idx_party_email ON party(email);

-- Addresses (multiple per party)
CREATE TABLE party_address (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_id UUID NOT NULL REFERENCES party(id),
    address_type VARCHAR(30) NOT NULL,  -- primary, mailing, previous, employment
    street_line_1 VARCHAR(255) NOT NULL,
    street_line_2 VARCHAR(255),
    city VARCHAR(100) NOT NULL,
    state_code VARCHAR(2) NOT NULL,     -- ISO 3166-2 subdivision
    postal_code VARCHAR(10) NOT NULL,
    country_code VARCHAR(2) NOT NULL DEFAULT 'US',  -- ISO 3166-1 alpha-2
    county_fips VARCHAR(5),             -- FIPS code for HMDA reporting
    census_tract VARCHAR(11),           -- Census tract for HMDA
    residence_type VARCHAR(30),         -- own, rent, living_rent_free
    years_at_address DECIMAL(4,1),
    is_current BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_party_address_party ON party_address(party_id);

-- Employment records (MISMO: EMPLOYMENT container)
CREATE TABLE employment (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_id UUID NOT NULL REFERENCES party(id),
    employer_name VARCHAR(255) NOT NULL,
    employer_phone VARCHAR(20),
    position_title VARCHAR(100),
    employment_type VARCHAR(30) NOT NULL, -- W2, self_employed, 1099, military, retired
    start_date DATE NOT NULL,
    end_date DATE,                       -- NULL = current
    is_current BOOLEAN NOT NULL DEFAULT true,
    monthly_income DECIMAL(12,2),
    years_in_profession DECIMAL(4,1),
    verified BOOLEAN NOT NULL DEFAULT false,
    verified_at TIMESTAMPTZ,
    verification_source VARCHAR(50),     -- payroll_api, employer_letter, tax_return
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_employment_party ON employment(party_id);

-- Income sources (beyond employment)
CREATE TABLE income_source (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_id UUID NOT NULL REFERENCES party(id),
    income_type VARCHAR(50) NOT NULL,    -- rental, investment, alimony, social_security,
                                         -- pension, gig, self_employment, other
    description VARCHAR(255),
    monthly_amount DECIMAL(12,2) NOT NULL,
    is_recurring BOOLEAN NOT NULL DEFAULT true,
    verification_status VARCHAR(20) NOT NULL DEFAULT 'unverified',
    verified_at TIMESTAMPTZ,
    verification_method VARCHAR(50),     -- bank_statement, tax_return, direct_api
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_income_source_party ON income_source(party_id);

-- Assets and liabilities (MISMO: ASSET, LIABILITY containers)
CREATE TABLE party_asset (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_id UUID NOT NULL REFERENCES party(id),
    asset_type VARCHAR(50) NOT NULL,     -- checking, savings, investment, retirement,
                                         -- stock, real_estate, vehicle, other
    institution_name VARCHAR(255),
    account_number_encrypted BYTEA,
    current_value DECIMAL(14,2) NOT NULL,
    verified BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_party_asset_party ON party_asset(party_id);

CREATE TABLE party_liability (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_id UUID NOT NULL REFERENCES party(id),
    liability_type VARCHAR(50) NOT NULL, -- mortgage, auto_loan, student_loan,
                                         -- credit_card, heloc, other
    creditor_name VARCHAR(255),
    account_number_encrypted BYTEA,
    unpaid_balance DECIMAL(14,2),
    monthly_payment DECIMAL(12,2),
    months_remaining INT,
    to_be_paid_off BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_party_liability_party ON party_liability(party_id);
```

---

## Loan Application & Deal Structure

```sql
-- Product catalogue
CREATE TABLE loan_product (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_code VARCHAR(30) NOT NULL UNIQUE,
    product_name VARCHAR(100) NOT NULL,
    product_category VARCHAR(30) NOT NULL, -- mortgage, consumer, commercial, sba, heloc
    loan_purpose_type VARCHAR(50),         -- purchase, refinance, cash_out, construction
    min_amount DECIMAL(14,2),
    max_amount DECIMAL(14,2),
    min_term_months INT,
    max_term_months INT,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- The deal: top-level container (MISMO: DEAL)
CREATE TABLE loan_application (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_number VARCHAR(30) NOT NULL UNIQUE, -- human-readable identifier
    loan_product_id UUID NOT NULL REFERENCES loan_product(id),
    -- Loan terms
    requested_amount DECIMAL(14,2) NOT NULL,
    approved_amount DECIMAL(14,2),
    loan_term_months INT NOT NULL,
    interest_rate_type VARCHAR(20),      -- fixed, adjustable, variable
    requested_rate DECIMAL(6,4),
    approved_rate DECIMAL(6,4),
    loan_purpose VARCHAR(50) NOT NULL,   -- purchase, refinance, cash_out_refinance,
                                          -- home_improvement, debt_consolidation
    occupancy_type VARCHAR(30),          -- primary_residence, second_home, investment
    -- Channel
    channel VARCHAR(30) NOT NULL,        -- web, mobile, branch, call_center, broker
    referral_source VARCHAR(100),
    -- Assignment
    loan_officer_id UUID REFERENCES staff(id),
    processor_id UUID REFERENCES staff(id),
    underwriter_id UUID REFERENCES staff(id),
    branch_id UUID REFERENCES branch(id),
    -- Status
    status VARCHAR(30) NOT NULL DEFAULT 'draft',
    -- draft, submitted, processing, underwriting, approved,
    -- conditionally_approved, denied, withdrawn, funded, closed
    status_changed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    submitted_at TIMESTAMPTZ,
    decision_at TIMESTAMPTZ,
    funded_at TIMESTAMPTZ,
    closed_at TIMESTAMPTZ,
    -- Compliance
    hmda_reportable BOOLEAN NOT NULL DEFAULT false,
    trid_applicable BOOLEAN NOT NULL DEFAULT false,
    -- Timestamps
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_loan_app_status ON loan_application(status);
CREATE INDEX idx_loan_app_officer ON loan_application(loan_officer_id);
CREATE INDEX idx_loan_app_submitted ON loan_application(submitted_at);
CREATE INDEX idx_loan_app_number ON loan_application(application_number);

-- Junction: parties to applications with roles
CREATE TABLE application_party (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    party_id UUID NOT NULL REFERENCES party(id),
    role VARCHAR(30) NOT NULL,           -- borrower, co_borrower, guarantor, co_signer
    role_order INT NOT NULL DEFAULT 1,   -- primary = 1
    signing_status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (application_id, party_id, role)
);

CREATE INDEX idx_app_party_app ON application_party(application_id);
CREATE INDEX idx_app_party_party ON application_party(party_id);
```

---

## Collateral & Property

```sql
-- Collateral / subject property (MISMO: COLLATERAL > SUBJECT_PROPERTY)
CREATE TABLE collateral (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    collateral_type VARCHAR(30) NOT NULL, -- real_estate, vehicle, equipment,
                                           -- accounts_receivable, inventory
    -- Real estate fields
    property_type VARCHAR(30),            -- single_family, condo, multi_unit, commercial
    street_address VARCHAR(255),
    city VARCHAR(100),
    state_code VARCHAR(2),
    postal_code VARCHAR(10),
    county VARCHAR(100),
    -- Valuation
    estimated_value DECIMAL(14,2),
    appraised_value DECIMAL(14,2),
    appraisal_date DATE,
    appraisal_type VARCHAR(30),          -- full, desktop, waiver, avm
    ltv_ratio DECIMAL(6,4),              -- loan-to-value calculated
    cltv_ratio DECIMAL(6,4),             -- combined LTV
    -- Vehicle / equipment fields
    make VARCHAR(100),
    model VARCHAR(100),
    year INT,
    vin VARCHAR(20),
    -- Status
    lien_position INT DEFAULT 1,
    title_status VARCHAR(30),            -- clear, existing_lien, pending
    insurance_verified BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_collateral_app ON collateral(application_id);
CREATE INDEX idx_collateral_type ON collateral(collateral_type);
```

---

## Credit & Underwriting

```sql
-- Credit report pulls (FCRA compliant)
CREATE TABLE credit_report (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_id UUID NOT NULL REFERENCES party(id),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    bureau VARCHAR(20) NOT NULL,         -- equifax, experian, transunion
    report_type VARCHAR(20) NOT NULL,    -- individual, joint, tri_merge
    permissible_purpose VARCHAR(50) NOT NULL, -- FCRA permissible purpose code
    pull_date TIMESTAMPTZ NOT NULL,
    expiration_date DATE,                -- typically 120 days
    credit_score INT,
    score_model VARCHAR(30),             -- fico8, fico9, vantage3, vantage4
    report_reference VARCHAR(100),       -- bureau reference number
    report_data_encrypted BYTEA,         -- full report encrypted at rest
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_credit_report_party ON credit_report(party_id);
CREATE INDEX idx_credit_report_app ON credit_report(application_id);

-- Underwriting decision record
CREATE TABLE underwriting_decision (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    decision_type VARCHAR(30) NOT NULL,  -- automated, manual, override
    decision VARCHAR(20) NOT NULL,       -- approve, deny, refer, suspend,
                                          -- conditionally_approve
    decided_by_id UUID REFERENCES staff(id), -- NULL for automated
    decided_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Automated underwriting
    aus_system VARCHAR(30),              -- du (Desktop Underwriter), lpa (Loan Product Advisor)
    aus_recommendation VARCHAR(30),      -- approve_eligible, refer, caution
    aus_case_number VARCHAR(50),
    -- Risk metrics
    dti_ratio DECIMAL(6,4),             -- debt-to-income
    housing_ratio DECIMAL(6,4),         -- front-end DTI
    residual_income DECIMAL(12,2),
    representative_credit_score INT,
    risk_grade VARCHAR(10),
    -- Decision rationale
    decision_notes TEXT,
    is_current BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_uw_decision_app ON underwriting_decision(application_id);
CREATE INDEX idx_uw_decision_current ON underwriting_decision(application_id) WHERE is_current = true;

-- Adverse action reasons (ECOA / Regulation B)
CREATE TABLE adverse_action_reason (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    decision_id UUID NOT NULL REFERENCES underwriting_decision(id),
    reason_code VARCHAR(10) NOT NULL,    -- standardised reason code
    reason_text VARCHAR(255) NOT NULL,   -- human-readable explanation
    sort_order INT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_adverse_action_decision ON adverse_action_reason(decision_id);

-- Conditions / stipulations
CREATE TABLE underwriting_condition (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    decision_id UUID REFERENCES underwriting_decision(id),
    condition_type VARCHAR(30) NOT NULL, -- prior_to_closing, prior_to_funding,
                                          -- prior_to_docs
    category VARCHAR(50) NOT NULL,       -- income, asset, credit, title, appraisal,
                                          -- insurance, legal
    description TEXT NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'outstanding',
    -- outstanding, received, reviewed, cleared, waived
    assigned_to_id UUID REFERENCES staff(id),
    due_date DATE,
    cleared_by_id UUID REFERENCES staff(id),
    cleared_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_uw_condition_app ON underwriting_condition(application_id);
CREATE INDEX idx_uw_condition_status ON underwriting_condition(status);
```

---

## Document Management

```sql
CREATE TABLE document (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    party_id UUID REFERENCES party(id),  -- NULL if application-level document
    document_type VARCHAR(50) NOT NULL,  -- pay_stub, w2, tax_return, bank_statement,
                                          -- appraisal, title_report, insurance,
                                          -- loan_estimate, closing_disclosure, note
    document_category VARCHAR(30) NOT NULL, -- income, asset, credit, compliance,
                                              -- closing, servicing
    file_name VARCHAR(255) NOT NULL,
    file_size_bytes BIGINT,
    mime_type VARCHAR(100),
    storage_path VARCHAR(500) NOT NULL,  -- S3 or equivalent path
    storage_bucket VARCHAR(100),
    -- Processing
    ocr_processed BOOLEAN NOT NULL DEFAULT false,
    ai_extracted BOOLEAN NOT NULL DEFAULT false,
    extraction_confidence DECIMAL(5,4),
    -- Status
    status VARCHAR(20) NOT NULL DEFAULT 'uploaded',
    -- uploaded, processing, verified, rejected, archived
    verified_by_id UUID REFERENCES staff(id),
    verified_at TIMESTAMPTZ,
    -- Versioning
    version INT NOT NULL DEFAULT 1,
    supersedes_id UUID REFERENCES document(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_document_app ON document(application_id);
CREATE INDEX idx_document_party ON document(party_id);
CREATE INDEX idx_document_type ON document(document_type);
CREATE INDEX idx_document_status ON document(status);

-- eSignature envelopes
CREATE TABLE esignature_envelope (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    provider VARCHAR(20) NOT NULL,       -- docusign, hellosign
    external_envelope_id VARCHAR(100) NOT NULL,
    status VARCHAR(30) NOT NULL,         -- created, sent, delivered, signed,
                                          -- declined, voided
    sent_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    voided_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_esig_app ON esignature_envelope(application_id);

CREATE TABLE esignature_recipient (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    envelope_id UUID NOT NULL REFERENCES esignature_envelope(id),
    party_id UUID NOT NULL REFERENCES party(id),
    role VARCHAR(30) NOT NULL,           -- borrower, co_borrower, notary
    signing_order INT NOT NULL DEFAULT 1,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    signed_at TIMESTAMPTZ,
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_esig_recipient_envelope ON esignature_recipient(envelope_id);
```

---

## Compliance & Disclosures

```sql
-- TRID disclosure tracking
CREATE TABLE trid_disclosure (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    disclosure_type VARCHAR(30) NOT NULL, -- loan_estimate, closing_disclosure,
                                           -- revised_le, revised_cd
    version_number INT NOT NULL DEFAULT 1,
    issued_date DATE NOT NULL,
    received_date DATE,
    -- Tolerance tracking
    tolerance_category VARCHAR(20),      -- zero, ten_percent, unlimited
    fee_total DECIMAL(12,2),
    tolerance_cure_amount DECIMAL(12,2),
    -- Timing compliance
    waiting_period_end DATE,             -- 3 business day waiting period
    is_compliant BOOLEAN,
    compliance_notes TEXT,
    document_id UUID REFERENCES document(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_trid_app ON trid_disclosure(application_id);

-- HMDA data (all 110 fields mapped to FFIEC LAR specification)
CREATE TABLE hmda_data (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id) UNIQUE,
    -- Identifiers
    lei VARCHAR(20) NOT NULL,            -- ISO 17442
    uli VARCHAR(45) NOT NULL,            -- Universal Loan Identifier
    -- Application data
    action_taken INT NOT NULL,           -- 1-8 per HMDA codes
    action_taken_date DATE,
    loan_type INT NOT NULL,              -- 1=conventional, 2=FHA, 3=VA, 4=USDA
    loan_purpose INT NOT NULL,           -- 1=purchase, 2=refinance, 31=cash-out, etc.
    preapproval INT,
    construction_method INT,
    occupancy_type INT,
    loan_amount DECIMAL(14,2) NOT NULL,
    -- Property
    state_code VARCHAR(2),
    county_code VARCHAR(3),
    census_tract VARCHAR(11),
    -- Applicant demographics (collected per Regulation C)
    applicant_ethnicity_1 INT,
    applicant_race_1 INT,
    applicant_sex INT,
    co_applicant_ethnicity_1 INT,
    co_applicant_race_1 INT,
    co_applicant_sex INT,
    applicant_age INT,
    -- Pricing
    interest_rate DECIMAL(6,4),
    rate_spread DECIMAL(6,4),
    hoepa_status INT,
    total_loan_costs DECIMAL(12,2),
    origination_charges DECIMAL(12,2),
    -- Underwriting
    dti_ratio VARCHAR(10),
    cltv_ratio VARCHAR(10),
    -- Denial reasons
    denial_reason_1 INT,
    denial_reason_2 INT,
    denial_reason_3 INT,
    denial_reason_4 INT,
    -- AUS
    aus_1 INT,
    aus_result_1 INT,
    -- Submission
    reporting_year INT NOT NULL,
    submitted_to_ffiec BOOLEAN NOT NULL DEFAULT false,
    submitted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_hmda_year ON hmda_data(reporting_year);
CREATE INDEX idx_hmda_lei ON hmda_data(lei);
```

---

## Workflow & Task Management

```sql
-- Workflow definitions
CREATE TABLE workflow_template (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    product_category VARCHAR(30) NOT NULL, -- mortgage, consumer, commercial
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE workflow_step_template (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_template_id UUID NOT NULL REFERENCES workflow_template(id),
    step_name VARCHAR(100) NOT NULL,
    step_order INT NOT NULL,
    assigned_role VARCHAR(30) NOT NULL,  -- loan_officer, processor, underwriter,
                                          -- closer, compliance
    sla_hours INT,
    is_required BOOLEAN NOT NULL DEFAULT true,
    auto_action VARCHAR(50),             -- auto_assign, auto_approve, notify
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Active workflow instances
CREATE TABLE workflow_instance (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    workflow_template_id UUID NOT NULL REFERENCES workflow_template(id),
    current_step_id UUID REFERENCES workflow_step_template(id),
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_workflow_app ON workflow_instance(application_id);

-- Tasks assigned to staff
CREATE TABLE task (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    workflow_instance_id UUID REFERENCES workflow_instance(id),
    task_type VARCHAR(50) NOT NULL,      -- review_document, verify_income,
                                          -- order_appraisal, clear_condition
    title VARCHAR(255) NOT NULL,
    description TEXT,
    assigned_to_id UUID REFERENCES staff(id),
    assigned_role VARCHAR(30),
    priority VARCHAR(10) NOT NULL DEFAULT 'normal', -- low, normal, high, urgent
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    -- pending, in_progress, completed, cancelled, escalated
    due_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    completed_by_id UUID REFERENCES staff(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_task_assigned ON task(assigned_to_id, status);
CREATE INDEX idx_task_app ON task(application_id);
CREATE INDEX idx_task_due ON task(due_at) WHERE status IN ('pending', 'in_progress');
```

---

## Organisation & Staff

```sql
CREATE TABLE organisation (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    lei VARCHAR(20),                     -- ISO 17442
    nmls_id VARCHAR(20),                 -- NMLS for mortgage institutions
    organisation_type VARCHAR(30) NOT NULL, -- bank, credit_union, fintech, broker
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
    street_address VARCHAR(255),
    city VARCHAR(100),
    state_code VARCHAR(2),
    postal_code VARCHAR(10),
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
    nmls_id VARCHAR(20),                -- for licensed loan officers
    role VARCHAR(30) NOT NULL,           -- loan_officer, processor, underwriter,
                                          -- closer, compliance_officer, admin
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_staff_org ON staff(organisation_id);
CREATE INDEX idx_staff_role ON staff(role);

-- RBAC
CREATE TABLE permission (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    permission_code VARCHAR(50) NOT NULL UNIQUE,
    description VARCHAR(255)
);

CREATE TABLE role_permission (
    role VARCHAR(30) NOT NULL,
    permission_id UUID NOT NULL REFERENCES permission(id),
    PRIMARY KEY (role, permission_id)
);
```

---

## Audit Trail

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type VARCHAR(50) NOT NULL,    -- loan_application, party, document, etc.
    entity_id UUID NOT NULL,
    action VARCHAR(20) NOT NULL,         -- create, update, delete, view, export
    field_name VARCHAR(100),             -- specific field changed (NULL for create)
    old_value TEXT,
    new_value TEXT,
    performed_by_id UUID REFERENCES staff(id),
    performed_by_type VARCHAR(20) NOT NULL, -- staff, system, api
    ip_address INET,
    user_agent VARCHAR(500),
    performed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_performed_at ON audit_log(performed_at);
CREATE INDEX idx_audit_performed_by ON audit_log(performed_by_id);

-- Partition by month for performance
-- CREATE TABLE audit_log_2026_01 PARTITION OF audit_log
--     FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

---

## Fees & Pricing

```sql
CREATE TABLE loan_fee (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    fee_type VARCHAR(50) NOT NULL,       -- origination, appraisal, credit_report,
                                          -- title_search, recording, flood_cert,
                                          -- tax_service, government
    fee_name VARCHAR(100) NOT NULL,
    amount DECIMAL(12,2) NOT NULL,
    paid_by VARCHAR(20) NOT NULL,        -- borrower, seller, lender, third_party
    tolerance_category VARCHAR(20),      -- zero, ten_percent, unlimited (TRID)
    vendor_name VARCHAR(255),
    is_financed BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_loan_fee_app ON loan_fee(application_id);
```

---

## Notifications & Communications

```sql
CREATE TABLE notification (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID REFERENCES loan_application(id),
    recipient_party_id UUID REFERENCES party(id),
    recipient_staff_id UUID REFERENCES staff(id),
    channel VARCHAR(20) NOT NULL,        -- email, sms, push, in_app
    template_code VARCHAR(50) NOT NULL,
    subject VARCHAR(255),
    body TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    -- pending, sent, delivered, failed, read
    sent_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ,
    read_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notification_app ON notification(application_id);
CREATE INDEX idx_notification_status ON notification(status) WHERE status = 'pending';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Party Management | 5 | party, party_address, employment, income_source, party_asset, party_liability |
| Loan Application & Deal | 3 | loan_product, loan_application, application_party |
| Collateral | 1 | collateral |
| Credit & Underwriting | 4 | credit_report, underwriting_decision, adverse_action_reason, underwriting_condition |
| Document Management | 3 | document, esignature_envelope, esignature_recipient |
| Compliance | 2 | trid_disclosure, hmda_data |
| Workflow & Tasks | 4 | workflow_template, workflow_step_template, workflow_instance, task |
| Organisation & RBAC | 5 | organisation, branch, staff, permission, role_permission |
| Audit | 1 | audit_log (partitioned) |
| Fees | 1 | loan_fee |
| Notifications | 1 | notification |
| **Total** | **30** | Core tables; additional reference/lookup tables expected |

---

## Key Design Decisions

1. **Party-first design over application-first**: The `party` table is independent of any application, enabling the same borrower record to be reused across multiple loan applications and supporting CRM-like relationship management.

2. **MISMO-aligned container hierarchy**: Tables mirror the DEAL > LOAN > PARTY > ROLE structure from the MISMO Reference Model, making data export to ULAD/ULDD formats straightforward with direct field mapping rather than complex transformations.

3. **Encrypted PII with searchable tokens**: SSN and EIN are stored encrypted (BYTEA) with a cleartext last-four column for search, balancing security with operational needs. Credit report data is stored encrypted at rest.

4. **Separate underwriting_decision table with history**: Multiple decisions per application are supported (automated, manual, override) with an `is_current` flag, providing a complete audit trail of the decision chain without overwriting previous records.

5. **HMDA as a dedicated table**: All 110 HMDA fields are captured in a single table keyed to the application, making annual LAR file generation a straightforward SELECT rather than a complex multi-join reconstruction.

6. **Workflow as template/instance pattern**: Workflow templates define the process for each product type; instances track progress on individual applications. This avoids hard-coding workflow logic while keeping the data model explicit.

7. **Partitioned audit log**: The audit_log table is designed for monthly partitioning to manage the volume of change records without sacrificing query performance for compliance investigations.

8. **TRID tolerance tracking in the data model**: Fee tolerance categories (zero, ten_percent, unlimited) are stored on both fee and disclosure records, enabling automated tolerance violation detection at the database level.

9. **Multi-product support via loan_product catalogue**: A single schema handles mortgage, consumer, commercial, and SBA loans by varying the product configuration rather than requiring separate table sets per loan type.

10. **County FIPS and census tract on addresses**: These geographic identifiers support HMDA reporting and Community Reinvestment Act (CRA) analysis directly from the address record without external geocoding at report time.
