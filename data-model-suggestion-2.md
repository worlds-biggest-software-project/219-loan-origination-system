# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Loan Origination System · Created: 2026-05-20

## Philosophy

This model treats every change to a loan application as an immutable domain event stored in an append-only event store. The current state of any loan is derived by replaying its event stream. Read-optimised materialised views (projections) serve the operational UI and reporting needs, while the event store remains the single source of truth. This is the CQRS (Command Query Responsibility Segregation) pattern applied to lending.

Financial systems are natural candidates for event sourcing because regulators demand complete, tamper-proof audit trails and the ability to answer temporal queries ("what was the borrower's stated income at the time of the underwriting decision?"). Major investment banks use event sourcing for trade lifecycle management, and the pattern is increasingly adopted by fintechs building on the insight that the audit trail should not be a secondary concern bolted onto a CRUD system, but the primary data structure from which everything else is derived.

The core insight is that in a loan origination workflow, the sequence of events IS the business process: application submitted, document uploaded, credit pulled, income verified, decision rendered, condition cleared, disclosure issued, loan funded. These events have natural causal ordering and regulatory significance. By making them the source of truth, compliance becomes a property of the data model rather than a feature layered on top.

**Best for:** Organisations prioritising regulatory compliance and auditability, AI-powered analytics on historical patterns, and platforms that need to support temporal queries and "point-in-time" reconstructions for examiner inquiries.

**Trade-offs:**
- (+) Complete, immutable audit trail with zero additional effort — every change is an event
- (+) Temporal queries are trivial — replay events up to any timestamp to see state at that moment
- (+) Supports ECOA/FCRA examiner inquiries without complex audit table reconstruction
- (+) Natural fit for AI/ML training — event streams provide rich, sequential training data
- (+) Easy to add new projections (read models) without modifying the write path
- (+) Event replay enables precise compliance investigations and regulatory reporting
- (-) Higher complexity — developers must think in events rather than CRUD
- (-) Eventually consistent read models — brief lag between command and query visibility
- (-) Event schema evolution requires careful versioning (upcasting)
- (-) Storage grows monotonically — need archival strategy for old event streams
- (-) Debugging requires event replay tooling, not simple SELECT queries
- (-) More infrastructure: event store + projection engine + read database

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| MISMO v3.4/v3.6 | Event payloads for loan data changes use MISMO-aligned field names |
| ECOA (Regulation B) | Decision events carry adverse action reasons as first-class event data |
| FCRA (Regulation V) | Credit pull events capture permissible purpose and are immutable |
| TRID (Regulation Z/X) | Disclosure issuance events track timing compliance with built-in waiting periods |
| HMDA (Regulation C) | HMDA projection materialised from event stream at reporting time |
| FFIEC SR 11-7 | Model decisioning events include model version, inputs, and outputs for validation |
| NIST AI RMF | AI scoring events capture model metadata for governance and explainability |
| OCSF (Open Cybersecurity Schema Framework) | Event envelope structure follows OCSF-inspired patterns for structured logging |
| ISO 8601 | All event timestamps use ISO 8601 with timezone (TIMESTAMPTZ) |

---

## Event Store (Source of Truth)

```sql
-- The immutable event store — append-only, never updated or deleted
CREATE TABLE event_store (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- Stream identification
    stream_type VARCHAR(50) NOT NULL,    -- loan_application, party, collateral, document
    stream_id UUID NOT NULL,             -- the aggregate root ID
    -- Event metadata
    event_type VARCHAR(100) NOT NULL,    -- e.g., 'application.submitted',
                                          --       'credit.pulled',
                                          --       'decision.rendered'
    event_version INT NOT NULL,          -- monotonically increasing per stream
    -- Event data
    event_data JSONB NOT NULL,           -- the event payload (see examples below)
    event_metadata JSONB NOT NULL DEFAULT '{}',
    -- Example metadata:
    -- {
    --   "correlation_id": "uuid",      -- links related events across streams
    --   "causation_id": "uuid",        -- the event that caused this event
    --   "actor_type": "staff|system|api|borrower",
    --   "actor_id": "uuid",
    --   "ip_address": "192.168.1.1",
    --   "user_agent": "...",
    --   "source": "web_portal|api|batch"
    -- }
    -- Timestamps
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(), -- when the event happened in the domain
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT now(), -- when it was persisted (always >= occurred_at)

    -- Optimistic concurrency: no two events at the same version in a stream
    UNIQUE (stream_id, event_version)
);

-- Primary query: replay a stream's events in order
CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);

-- Query by event type (for building specific projections)
CREATE INDEX idx_event_type ON event_store(event_type, recorded_at);

-- Query by time range (for compliance investigations)
CREATE INDEX idx_event_recorded ON event_store(recorded_at);

-- Correlation queries (linking events across streams)
CREATE INDEX idx_event_correlation ON event_store((event_metadata->>'correlation_id'))
    WHERE event_metadata->>'correlation_id' IS NOT NULL;

-- Partition by month for performance at scale
-- CREATE TABLE event_store_2026_05 PARTITION OF event_store
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

### Event Type Taxonomy

```
-- Loan Application Lifecycle Events
application.created
application.submitted
application.assigned
application.status_changed
application.terms_updated
application.withdrawn
application.funded
application.closed

-- Party Events
party.created
party.updated
party.address_added
party.address_updated
party.employment_added
party.employment_updated
party.income_added
party.asset_added
party.liability_added

-- Credit Events
credit.report_ordered
credit.report_received
credit.score_recorded

-- Document Events
document.uploaded
document.ocr_processed
document.ai_extracted
document.verified
document.rejected
document.esignature_sent
document.esignature_completed

-- Underwriting Events
underwriting.automated_decision_rendered
underwriting.manual_review_started
underwriting.manual_decision_rendered
underwriting.decision_overridden
underwriting.condition_added
underwriting.condition_cleared
underwriting.condition_waived

-- Compliance Events
compliance.trid_le_issued
compliance.trid_cd_issued
compliance.trid_waiting_period_started
compliance.trid_waiting_period_cleared
compliance.hmda_data_captured
compliance.adverse_action_notice_generated
compliance.adverse_action_notice_sent

-- Collateral Events
collateral.added
collateral.appraisal_ordered
collateral.appraisal_received
collateral.valuation_updated
collateral.title_cleared
collateral.insurance_verified

-- Fee Events
fee.added
fee.updated
fee.tolerance_checked

-- Fraud Events
fraud.identity_check_initiated
fraud.identity_check_completed
fraud.alert_raised
fraud.alert_resolved

-- Communication Events
communication.notification_sent
communication.notification_delivered
communication.notification_read
```

### Example Event Payloads

```sql
-- Example: application.submitted
-- {
--   "application_number": "LN-2026-00042",
--   "product_code": "CONV_30_FIXED",
--   "requested_amount": 350000.00,
--   "loan_term_months": 360,
--   "interest_rate_type": "fixed",
--   "loan_purpose": "purchase",
--   "occupancy_type": "primary_residence",
--   "channel": "web",
--   "borrower_party_id": "uuid",
--   "co_borrower_party_id": "uuid"
-- }

-- Example: underwriting.automated_decision_rendered
-- {
--   "decision": "conditionally_approve",
--   "aus_system": "internal_engine",
--   "model_version": "v3.2.1",
--   "model_id": "credit_decision_v3",
--   "inputs": {
--     "credit_score": 742,
--     "dti_ratio": 0.3850,
--     "ltv_ratio": 0.8000,
--     "employment_months": 48,
--     "monthly_income": 8500.00,
--     "monthly_debt": 3272.50
--   },
--   "outputs": {
--     "risk_grade": "A2",
--     "approval_probability": 0.94,
--     "recommended_rate": 6.375,
--     "conditions_required": ["verify_income", "clear_title"]
--   },
--   "explainability": {
--     "top_factors": [
--       {"factor": "credit_score", "impact": "positive", "weight": 0.35},
--       {"factor": "dti_ratio", "impact": "neutral", "weight": 0.25},
--       {"factor": "employment_stability", "impact": "positive", "weight": 0.20}
--     ],
--     "shap_values_ref": "s3://models/shap/run-2026-05-20-uuid.json"
--   }
-- }

-- Example: compliance.adverse_action_notice_generated
-- {
--   "decision_event_id": "uuid",
--   "reason_codes": ["R001", "R014", "R032"],
--   "reasons": [
--     "Amount owed on accounts is too high",
--     "Length of credit history is too short",
--     "Insufficient income for the requested loan amount"
--   ],
--   "natural_language_explanation": "Your application was not approved primarily
--     because your current outstanding debt of $45,200 results in a debt-to-income
--     ratio above our guidelines. Additionally, your credit history of 3 years is
--     shorter than typically required for this loan type.",
--   "credit_score_used": 682,
--   "score_model": "fico9",
--   "notice_template": "adverse_action_v2",
--   "delivery_method": "mail_and_email"
-- }
```

---

## Snapshot Store (Performance Optimisation)

```sql
-- Periodic snapshots to avoid replaying full event streams
CREATE TABLE event_snapshot (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type VARCHAR(50) NOT NULL,
    stream_id UUID NOT NULL,
    snapshot_version INT NOT NULL,       -- the event_version this snapshot reflects
    snapshot_data JSONB NOT NULL,        -- the aggregate state at this version
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, snapshot_version)
);

CREATE INDEX idx_snapshot_stream ON event_snapshot(stream_id, snapshot_version DESC);

-- Snapshots are taken every N events (e.g., every 50)
-- To rebuild state: load latest snapshot, then replay events from snapshot_version+1
```

---

## Read Models (Materialised Projections)

These tables are derived from the event store. They can be rebuilt from scratch at any time by replaying events. They are the "query side" of CQRS.

```sql
-- Projection: current loan application state (denormalised for fast reads)
CREATE TABLE v_loan_application (
    id UUID PRIMARY KEY,                 -- same as stream_id
    application_number VARCHAR(30) NOT NULL UNIQUE,
    -- Product
    product_code VARCHAR(30) NOT NULL,
    product_name VARCHAR(100),
    product_category VARCHAR(30),
    -- Borrower (denormalised)
    primary_borrower_id UUID,
    primary_borrower_name VARCHAR(200),
    co_borrower_id UUID,
    co_borrower_name VARCHAR(200),
    -- Loan terms
    requested_amount DECIMAL(14,2),
    approved_amount DECIMAL(14,2),
    loan_term_months INT,
    interest_rate_type VARCHAR(20),
    current_rate DECIMAL(6,4),
    loan_purpose VARCHAR(50),
    occupancy_type VARCHAR(30),
    -- Risk metrics
    representative_credit_score INT,
    dti_ratio DECIMAL(6,4),
    ltv_ratio DECIMAL(6,4),
    risk_grade VARCHAR(10),
    -- Assignment
    loan_officer_name VARCHAR(200),
    processor_name VARCHAR(200),
    underwriter_name VARCHAR(200),
    branch_name VARCHAR(100),
    channel VARCHAR(30),
    -- Status
    status VARCHAR(30) NOT NULL,
    status_changed_at TIMESTAMPTZ,
    submitted_at TIMESTAMPTZ,
    decision VARCHAR(20),
    decision_at TIMESTAMPTZ,
    funded_at TIMESTAMPTZ,
    -- Conditions
    total_conditions INT DEFAULT 0,
    outstanding_conditions INT DEFAULT 0,
    -- Compliance
    hmda_reportable BOOLEAN DEFAULT false,
    trid_applicable BOOLEAN DEFAULT false,
    -- Projection metadata
    last_event_version INT NOT NULL,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_loan_status ON v_loan_application(status);
CREATE INDEX idx_v_loan_officer ON v_loan_application(loan_officer_name);
CREATE INDEX idx_v_loan_submitted ON v_loan_application(submitted_at);
CREATE INDEX idx_v_loan_borrower ON v_loan_application(primary_borrower_id);

-- Projection: party directory
CREATE TABLE v_party (
    id UUID PRIMARY KEY,
    party_type VARCHAR(20) NOT NULL,
    display_name VARCHAR(255) NOT NULL,  -- computed: first + last or legal_name
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    legal_name VARCHAR(255),
    email VARCHAR(255),
    phone_primary VARCHAR(20),
    ssn_last_four VARCHAR(4),
    current_address_line VARCHAR(255),
    current_city VARCHAR(100),
    current_state VARCHAR(2),
    active_application_count INT DEFAULT 0,
    total_application_count INT DEFAULT 0,
    last_event_version INT NOT NULL,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_party_name ON v_party(display_name);
CREATE INDEX idx_v_party_email ON v_party(email);

-- Projection: document tracker
CREATE TABLE v_document (
    id UUID PRIMARY KEY,
    application_id UUID NOT NULL,
    party_id UUID,
    document_type VARCHAR(50) NOT NULL,
    file_name VARCHAR(255) NOT NULL,
    status VARCHAR(20) NOT NULL,
    uploaded_at TIMESTAMPTZ,
    verified_at TIMESTAMPTZ,
    verified_by VARCHAR(200),
    ai_extraction_confidence DECIMAL(5,4),
    last_event_version INT NOT NULL,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_doc_app ON v_document(application_id);
CREATE INDEX idx_v_doc_status ON v_document(status);

-- Projection: pipeline dashboard (aggregated)
CREATE TABLE v_pipeline_summary (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    loan_officer_id UUID,
    branch_id UUID,
    period_date DATE NOT NULL,
    -- Counts by status
    draft_count INT DEFAULT 0,
    submitted_count INT DEFAULT 0,
    processing_count INT DEFAULT 0,
    underwriting_count INT DEFAULT 0,
    approved_count INT DEFAULT 0,
    denied_count INT DEFAULT 0,
    funded_count INT DEFAULT 0,
    withdrawn_count INT DEFAULT 0,
    -- Volume
    total_requested_amount DECIMAL(16,2) DEFAULT 0,
    total_funded_amount DECIMAL(16,2) DEFAULT 0,
    -- Performance
    avg_days_to_decision DECIMAL(6,1),
    avg_days_to_funding DECIMAL(6,1),
    -- Metadata
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_pipeline_officer ON v_pipeline_summary(loan_officer_id, period_date);
CREATE INDEX idx_v_pipeline_date ON v_pipeline_summary(period_date);

-- Projection: HMDA annual report (built from events)
CREATE TABLE v_hmda_lar (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL UNIQUE,
    reporting_year INT NOT NULL,
    lei VARCHAR(20) NOT NULL,
    uli VARCHAR(45) NOT NULL,
    action_taken INT NOT NULL,
    action_taken_date DATE,
    loan_type INT,
    loan_purpose INT,
    loan_amount DECIMAL(14,2),
    state_code VARCHAR(2),
    county_code VARCHAR(3),
    census_tract VARCHAR(11),
    applicant_ethnicity_1 INT,
    applicant_race_1 INT,
    applicant_sex INT,
    interest_rate DECIMAL(6,4),
    dti_ratio VARCHAR(10),
    denial_reason_1 INT,
    aus_1 INT,
    aus_result_1 INT,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_hmda_year ON v_hmda_lar(reporting_year);

-- Projection: compliance timeline (for examiner inquiries)
CREATE TABLE v_compliance_timeline (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    event_summary VARCHAR(500) NOT NULL,  -- human-readable description
    compliance_relevant BOOLEAN NOT NULL DEFAULT true,
    occurred_at TIMESTAMPTZ NOT NULL,
    actor_name VARCHAR(200),
    actor_type VARCHAR(20),
    event_id UUID NOT NULL,              -- back-reference to event_store
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_compliance_app ON v_compliance_timeline(application_id, occurred_at);
```

---

## Projection Management

```sql
-- Tracks the last event processed by each projection
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id UUID NOT NULL,
    last_event_recorded_at TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    last_run_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    -- active, paused, rebuilding, error
    error_message TEXT,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example projections:
-- INSERT INTO projection_checkpoint VALUES
--   ('v_loan_application', ..., ..., 0, now(), 'active', NULL, now()),
--   ('v_party', ..., ..., 0, now(), 'active', NULL, now()),
--   ('v_document', ..., ..., 0, now(), 'active', NULL, now()),
--   ('v_pipeline_summary', ..., ..., 0, now(), 'active', NULL, now()),
--   ('v_hmda_lar', ..., ..., 0, now(), 'active', NULL, now()),
--   ('v_compliance_timeline', ..., ..., 0, now(), 'active', NULL, now());
```

---

## Command & Aggregate Reference Tables

These are the only mutable tables — they support the command side (write path) with reference data and idempotency.

```sql
-- Loan product catalogue (reference data, rarely changes)
CREATE TABLE loan_product (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_code VARCHAR(30) NOT NULL UNIQUE,
    product_name VARCHAR(100) NOT NULL,
    product_category VARCHAR(30) NOT NULL,
    min_amount DECIMAL(14,2),
    max_amount DECIMAL(14,2),
    min_term_months INT,
    max_term_months INT,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Credit decision model registry (for FFIEC SR 11-7 compliance)
CREATE TABLE decision_model_registry (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id VARCHAR(50) NOT NULL,
    model_version VARCHAR(20) NOT NULL,
    model_type VARCHAR(30) NOT NULL,     -- credit_scoring, fraud_detection, income_analysis
    description TEXT,
    training_data_period VARCHAR(50),    -- e.g., "2023-01 to 2025-12"
    validation_date DATE,
    validation_status VARCHAR(20),       -- validated, pending_validation, retired
    fairness_metrics JSONB,              -- disparate impact ratios by protected class
    deployed_at TIMESTAMPTZ,
    retired_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (model_id, model_version)
);

-- Idempotency keys for command deduplication
CREATE TABLE command_idempotency (
    idempotency_key VARCHAR(100) PRIMARY KEY,
    command_type VARCHAR(100) NOT NULL,
    result_event_id UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at TIMESTAMPTZ NOT NULL DEFAULT (now() + interval '7 days')
);

CREATE INDEX idx_idempotency_expires ON command_idempotency(expires_at);

-- Organisation and staff (reference data for the command side)
CREATE TABLE organisation (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    lei VARCHAR(20),
    nmls_id VARCHAR(20),
    organisation_type VARCHAR(30) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE staff (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email VARCHAR(255) NOT NULL UNIQUE,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    nmls_id VARCHAR(20),
    role VARCHAR(30) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Temporal Query Examples

```sql
-- "What was the borrower's stated income at the time of underwriting decision?"
-- Replay party events up to the decision timestamp
SELECT event_data
FROM event_store
WHERE stream_id = '<party_uuid>'
  AND event_type IN ('party.income_added', 'party.employment_added', 'party.employment_updated')
  AND occurred_at <= (
      SELECT occurred_at
      FROM event_store
      WHERE stream_id = '<application_uuid>'
        AND event_type = 'underwriting.automated_decision_rendered'
      ORDER BY event_version DESC
      LIMIT 1
  )
ORDER BY event_version;

-- "Show me every change to this loan application in chronological order"
-- (the compliance timeline projection handles this, but raw query is also simple)
SELECT event_type, event_data, event_metadata, occurred_at
FROM event_store
WHERE stream_id = '<application_uuid>'
ORDER BY event_version;

-- "What was the LTV when the loan estimate was issued?"
SELECT event_data->>'ltv_ratio' AS ltv_at_le_time
FROM event_store
WHERE stream_id = '<application_uuid>'
  AND event_type = 'compliance.trid_le_issued'
ORDER BY event_version DESC
LIMIT 1;

-- "How many loans were approved by model version v3.2.1?"
SELECT COUNT(*)
FROM event_store
WHERE event_type = 'underwriting.automated_decision_rendered'
  AND event_data->>'decision' = 'approve'
  AND event_data->>'model_version' = 'v3.2.1'
  AND occurred_at >= '2026-01-01'
  AND occurred_at < '2026-06-01';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | event_store (partitioned by month) |
| Snapshots | 1 | event_snapshot |
| Projections (Read Models) | 7 | v_loan_application, v_party, v_document, v_pipeline_summary, v_hmda_lar, v_compliance_timeline, projection_checkpoint |
| Reference / Command | 5 | loan_product, decision_model_registry, command_idempotency, organisation, staff |
| **Total** | **14** | Plus additional projections as needed |

---

## Key Design Decisions

1. **Single event_store table as the source of truth**: All domain events across all aggregate types live in one table, partitioned by time. This simplifies infrastructure (one append-only table) while supporting cross-aggregate correlation queries for compliance investigations.

2. **JSONB event payloads with typed event names**: Event data is stored as JSONB for schema flexibility, while the `event_type` column provides a typed taxonomy for building projections. New event types can be added without schema migrations.

3. **Event metadata separates "what happened" from "who/how"**: The `event_data` captures domain facts (income amount, decision outcome), while `event_metadata` captures operational context (actor, IP, correlation ID). This separation supports both business analysis and security audit.

4. **Optimistic concurrency via stream version**: The UNIQUE constraint on `(stream_id, event_version)` prevents concurrent writes to the same aggregate, ensuring consistency without distributed locks.

5. **Projections are disposable**: Every `v_*` table can be dropped and rebuilt by replaying events from the event store. This makes schema evolution on the read side trivial — change the projection logic and rebuild.

6. **Decision model registry for FFIEC compliance**: AI/ML model metadata is stored as reference data, and every automated decision event references the model version used. This satisfies SR 11-7 model risk management requirements for model inventory and traceability.

7. **Snapshots for performance**: Rather than replaying thousands of events for a long-lived loan, periodic snapshots cache the aggregate state. Rebuild requires only loading the latest snapshot and replaying subsequent events.

8. **Command idempotency table**: Prevents duplicate event creation from retried API calls or webhook deliveries, which is critical in financial workflows where duplicate credit pulls or duplicate funding events would be operationally harmful.

9. **Compliance timeline as a first-class projection**: Rather than assembling audit trails from scattered tables, the `v_compliance_timeline` projection provides a pre-built, human-readable chronological view of every compliance-relevant event — ready for examiner review.

10. **Event store partitioning by time**: Monthly partitions enable efficient archival of old events (move to cold storage) while keeping recent events on fast storage, managing the monotonically growing storage requirement.
