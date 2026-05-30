# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Loan Origination System · Created: 2026-05-20

## Philosophy

This model adds a property graph layer on top of relational operational tables to handle the relationship-heavy queries that loan origination systems increasingly need: party-to-party relationships (borrower/co-borrower/guarantor chains across multiple loans), collateral cross-collateralisation, conflict-of-interest detection, fraud ring identification, and portfolio concentration analysis. The operational CRUD workflow runs against conventional relational tables, while a graph overlay — implemented using PostgreSQL's `graph_node` / `graph_edge` tables (or Apache AGE extension for Cypher queries) — captures the rich web of relationships.

The insight is that lending is fundamentally a relationship business. A single borrower may have multiple applications; a guarantor may guarantee loans across different entities they control; a property may serve as collateral for multiple facilities; a loan officer may have referral relationships with real estate agents; and fraud networks connect identities through shared addresses, phone numbers, employers, and devices. Traditional relational models can answer these questions, but they require complex multi-join queries or recursive CTEs that become unwieldy as relationship depth grows. A graph model makes these queries natural: "find all entities within 3 hops of this borrower" or "show all loans collateralised by properties in this same development."

This approach is used by major banks for anti-money laundering (AML) and Know Your Customer (KYC) workflows, where the graph of relationships between parties, accounts, and transactions is the primary analytical structure. It is also increasingly used in commercial lending for corporate ownership chain analysis and concentration risk assessment.

**Best for:** Institutions handling commercial lending with complex ownership structures, banks needing fraud ring detection and AML compliance, lenders managing cross-collateralised portfolios, and platforms requiring conflict-of-interest checks across loan officers and borrower relationships.

**Trade-offs:**
- (+) Relationship queries (ownership chains, fraud rings, collateral networks) are orders of magnitude faster than relational JOINs
- (+) Natural model for corporate hierarchies, guarantor chains, and beneficial ownership
- (+) Enables AI-powered fraud detection by analysing graph topology (clustering coefficients, shortest paths, community detection)
- (+) Portfolio concentration analysis across any relationship dimension (geography, industry, guarantor, collateral)
- (+) Conflict-of-interest detection becomes a simple path-finding query
- (-) Additional infrastructure complexity (graph engine + relational DB, or PostgreSQL with AGE extension)
- (-) Developers must learn graph query languages (Cypher, GQL, or graph SQL)
- (-) Graph synchronisation with relational tables requires careful consistency management
- (-) Overkill for simple consumer lending with independent applications
- (-) Graph databases have less mature tooling for backup, monitoring, and BI integration
- (-) Write performance on graph edges can be a bottleneck under high transaction volume

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| MISMO v3.4/v3.6 | Relational tables follow MISMO container naming; graph extends relationships |
| GQL (ISO/IEC 39075) | Graph queries use the emerging SQL/PGQ standard where supported |
| ISO 17442 (LEI) | Legal Entity Identifier used as a node property for organisation matching |
| ISO 3166 | Jurisdiction nodes enable geographic concentration analysis |
| FinCEN Beneficial Ownership | Ownership graph edges capture 25%+ beneficial ownership per FinCEN rules |
| ECOA / FCRA | Decision nodes in the graph carry adverse action data for relationship-aware analysis |
| HMDA | HMDA data on application nodes enables geographic disparity analysis via graph traversal |
| Open Cap Table Format | Ownership percentage edges align with equity/ownership data models |
| FFIEC BSA/AML | Graph topology supports Suspicious Activity Report (SAR) relationship analysis |

---

## Relational Layer (Operational CRUD)

The relational tables handle the day-to-day loan origination workflow. They are simpler than the fully normalised model because the graph layer handles relationship complexity.

```sql
-- Parties: persons and organisations
CREATE TABLE party (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_type VARCHAR(20) NOT NULL CHECK (party_type IN ('individual', 'organisation')),
    display_name VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    legal_name VARCHAR(255),
    email VARCHAR(255),
    phone_primary VARCHAR(20),
    ssn_last_four VARCHAR(4),
    lei VARCHAR(20),                     -- ISO 17442 for organisations
    ein_encrypted BYTEA,
    ssn_encrypted BYTEA,
    entity_type VARCHAR(50),             -- LLC, corporation, sole_proprietorship
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_party_name ON party(display_name);
CREATE INDEX idx_party_email ON party(email);
CREATE INDEX idx_party_lei ON party(lei) WHERE lei IS NOT NULL;

-- Addresses (also graph nodes for shared-address detection)
CREATE TABLE address (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    street_line_1 VARCHAR(255) NOT NULL,
    street_line_2 VARCHAR(255),
    city VARCHAR(100) NOT NULL,
    state_code VARCHAR(2) NOT NULL,
    postal_code VARCHAR(10) NOT NULL,
    country_code VARCHAR(2) NOT NULL DEFAULT 'US',
    county_fips VARCHAR(5),
    census_tract VARCHAR(11),
    -- Normalised address for deduplication
    normalised_hash VARCHAR(64),         -- SHA-256 of normalised address string
    latitude DECIMAL(10,7),
    longitude DECIMAL(10,7),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_address_hash ON address(normalised_hash);
CREATE INDEX idx_address_postal ON address(postal_code);

-- Loan applications
CREATE TABLE loan_application (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_number VARCHAR(30) NOT NULL UNIQUE,
    loan_product_id UUID NOT NULL REFERENCES loan_product(id),
    requested_amount DECIMAL(14,2) NOT NULL,
    approved_amount DECIMAL(14,2),
    loan_term_months INT NOT NULL,
    interest_rate_type VARCHAR(20),
    current_rate DECIMAL(6,4),
    loan_purpose VARCHAR(50) NOT NULL,
    occupancy_type VARCHAR(30),
    channel VARCHAR(30) NOT NULL,
    -- Assignment
    loan_officer_id UUID REFERENCES staff(id),
    processor_id UUID REFERENCES staff(id),
    underwriter_id UUID REFERENCES staff(id),
    branch_id UUID REFERENCES branch(id),
    -- Status
    status VARCHAR(30) NOT NULL DEFAULT 'draft',
    submitted_at TIMESTAMPTZ,
    decision_at TIMESTAMPTZ,
    funded_at TIMESTAMPTZ,
    -- Risk
    representative_credit_score INT,
    dti_ratio DECIMAL(6,4),
    ltv_ratio DECIMAL(6,4),
    risk_grade VARCHAR(10),
    -- Compliance
    hmda_reportable BOOLEAN DEFAULT false,
    trid_applicable BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_app_status ON loan_application(status);
CREATE INDEX idx_app_officer ON loan_application(loan_officer_id);
CREATE INDEX idx_app_submitted ON loan_application(submitted_at);

-- Loan product catalogue
CREATE TABLE loan_product (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_code VARCHAR(30) NOT NULL UNIQUE,
    product_name VARCHAR(100) NOT NULL,
    product_category VARCHAR(30) NOT NULL,
    min_amount DECIMAL(14,2),
    max_amount DECIMAL(14,2),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Collateral
CREATE TABLE collateral (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    collateral_type VARCHAR(30) NOT NULL,
    estimated_value DECIMAL(14,2),
    appraised_value DECIMAL(14,2),
    lien_position INT DEFAULT 1,
    -- Property fields
    property_address_id UUID REFERENCES address(id),
    property_type VARCHAR(30),
    -- Vehicle/equipment fields
    description VARCHAR(255),
    vin VARCHAR(20),
    serial_number VARCHAR(50),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_collateral_app ON collateral(application_id);
CREATE INDEX idx_collateral_address ON collateral(property_address_id) WHERE property_address_id IS NOT NULL;

-- Underwriting decisions
CREATE TABLE underwriting_decision (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    decision_type VARCHAR(30) NOT NULL,
    decision VARCHAR(20) NOT NULL,
    decided_by_id UUID REFERENCES staff(id),
    decided_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    model_id VARCHAR(50),
    model_version VARCHAR(20),
    dti_ratio DECIMAL(6,4),
    ltv_ratio DECIMAL(6,4),
    risk_grade VARCHAR(10),
    decision_notes TEXT,
    is_current BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_uw_app ON underwriting_decision(application_id);

-- Documents
CREATE TABLE document (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    party_id UUID REFERENCES party(id),
    document_type VARCHAR(50) NOT NULL,
    file_name VARCHAR(255) NOT NULL,
    storage_path VARCHAR(500) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'uploaded',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_doc_app ON document(application_id);

-- Organisation & Staff
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

CREATE TABLE branch (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    branch_name VARCHAR(100) NOT NULL,
    branch_code VARCHAR(20),
    address_id UUID REFERENCES address(id),
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
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Audit log
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    action VARCHAR(20) NOT NULL,
    field_name VARCHAR(100),
    old_value TEXT,
    new_value TEXT,
    performed_by_id UUID,
    performed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_time ON audit_log(performed_at);
```

---

## Graph Layer

The graph layer models relationships between entities using a generic node/edge pattern implemented in PostgreSQL. Each node references a row in the relational layer; edges carry typed, weighted relationships with temporal validity.

```sql
-- Graph nodes: every entity that participates in relationships
CREATE TABLE graph_node (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type VARCHAR(30) NOT NULL,      -- party, application, collateral, address,
                                          -- employer, branch, staff, organisation
    entity_id UUID NOT NULL,             -- FK to the relational table row
    -- Denormalised display fields for fast graph traversal
    label VARCHAR(255) NOT NULL,         -- human-readable label
    properties JSONB NOT NULL DEFAULT '{}',
    -- Example for a party node:
    -- {
    --   "party_type": "individual",
    --   "state": "TX",
    --   "risk_grade": "A2",
    --   "total_exposure": 450000.00,
    --   "active_loans": 2
    -- }
    --
    -- Example for an address node:
    -- {
    --   "postal_code": "78701",
    --   "county_fips": "48453",
    --   "census_tract": "48453001400",
    --   "latitude": 30.2672,
    --   "longitude": -97.7431
    -- }
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (node_type, entity_id)
);

CREATE INDEX idx_gnode_type ON graph_node(node_type);
CREATE INDEX idx_gnode_entity ON graph_node(entity_id);
CREATE INDEX idx_gnode_properties ON graph_node USING GIN (properties jsonb_path_ops);

-- Graph edges: typed, weighted, temporal relationships
CREATE TABLE graph_edge (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_node_id UUID NOT NULL REFERENCES graph_node(id),
    target_node_id UUID NOT NULL REFERENCES graph_node(id),
    edge_type VARCHAR(50) NOT NULL,
    -- Relationship types:
    -- Party relationships:
    --   'borrower_on'      party -> application
    --   'co_borrower_on'   party -> application
    --   'guarantor_for'    party -> application
    --   'owns'             party -> organisation (with ownership_pct)
    --   'officer_of'       party -> organisation
    --   'related_to'       party -> party (family, business partner)
    --   'employed_by'      party -> employer_node
    --   'resides_at'       party -> address
    --   'previously_at'    party -> address
    --
    -- Collateral relationships:
    --   'secured_by'       application -> collateral
    --   'located_at'       collateral -> address
    --   'cross_collateral' collateral -> collateral
    --
    -- Staff relationships:
    --   'originated_by'    application -> staff
    --   'processed_by'     application -> staff
    --   'underwritten_by'  application -> staff
    --   'works_at'         staff -> branch
    --   'referred_by'      application -> external_referrer
    --
    -- Fraud detection edges:
    --   'shares_ssn'       party -> party (synthetic identity)
    --   'shares_phone'     party -> party
    --   'shares_email'     party -> party
    --   'shares_address'   party -> party (via address nodes)
    --   'shares_employer'  party -> party (via employer nodes)
    --   'shares_device'    party -> party (same device fingerprint)

    -- Edge properties
    weight DECIMAL(8,4) DEFAULT 1.0,     -- relationship strength/significance
    properties JSONB NOT NULL DEFAULT '{}',
    -- Examples:
    -- ownership edge: {"ownership_pct": 51.0, "ownership_type": "direct"}
    -- borrower edge: {"role_order": 1, "signing_status": "signed"}
    -- guarantor edge: {"guarantee_amount": 200000.00, "guarantee_type": "limited"}
    -- fraud edge: {"confidence": 0.87, "detection_method": "address_matching"}

    -- Temporal validity
    valid_from TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to TIMESTAMPTZ,                -- NULL = currently active
    is_active BOOLEAN NOT NULL DEFAULT true,

    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gedge_source ON graph_edge(source_node_id, edge_type) WHERE is_active = true;
CREATE INDEX idx_gedge_target ON graph_edge(target_node_id, edge_type) WHERE is_active = true;
CREATE INDEX idx_gedge_type ON graph_edge(edge_type) WHERE is_active = true;
CREATE INDEX idx_gedge_properties ON graph_edge USING GIN (properties jsonb_path_ops);
CREATE INDEX idx_gedge_temporal ON graph_edge(valid_from, valid_to) WHERE is_active = true;
```

---

## Graph Query Examples

### Ownership Chain Traversal (Commercial Lending)

```sql
-- Find the full ownership chain for a commercial borrower
-- "Who ultimately controls this entity?"
WITH RECURSIVE ownership_chain AS (
    -- Start from the borrower entity
    SELECT
        gn.id AS node_id,
        gn.label,
        gn.node_type,
        ge.edge_type,
        (ge.properties->>'ownership_pct')::decimal AS ownership_pct,
        1 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_node gn
    JOIN graph_edge ge ON ge.target_node_id = gn.id
    WHERE ge.source_node_id = (
        SELECT id FROM graph_node
        WHERE node_type = 'party' AND entity_id = '<borrower_org_uuid>'
    )
    AND ge.edge_type = 'owns'
    AND ge.is_active = true

    UNION ALL

    -- Traverse upward through ownership
    SELECT
        gn2.id,
        gn2.label,
        gn2.node_type,
        ge2.edge_type,
        (ge2.properties->>'ownership_pct')::decimal,
        oc.depth + 1,
        oc.path || gn2.id
    FROM ownership_chain oc
    JOIN graph_edge ge2 ON ge2.target_node_id = oc.node_id
    JOIN graph_node gn2 ON gn2.id = ge2.source_node_id
    WHERE ge2.edge_type = 'owns'
      AND ge2.is_active = true
      AND gn2.id != ALL(oc.path)  -- prevent cycles
      AND oc.depth < 10
)
SELECT label, node_type, ownership_pct, depth
FROM ownership_chain
ORDER BY depth;
```

### Fraud Ring Detection

```sql
-- Find all parties connected to a suspect within 3 hops via shared identifiers
WITH RECURSIVE fraud_network AS (
    -- Start from suspect party
    SELECT
        gn.id AS node_id,
        gn.label,
        gn.entity_id,
        0 AS depth,
        ARRAY[gn.id] AS path,
        NULL::VARCHAR AS connection_type
    FROM graph_node gn
    WHERE gn.node_type = 'party' AND gn.entity_id = '<suspect_party_uuid>'

    UNION ALL

    -- Traverse through shared identifiers
    SELECT
        gn2.id,
        gn2.label,
        gn2.entity_id,
        fn.depth + 1,
        fn.path || gn2.id,
        ge.edge_type
    FROM fraud_network fn
    -- Outgoing edges from current node
    JOIN graph_edge ge ON ge.source_node_id = fn.node_id
    JOIN graph_node gn2 ON gn2.id = ge.target_node_id
    WHERE ge.edge_type IN ('shares_address', 'shares_phone', 'shares_email',
                           'shares_employer', 'shares_device', 'shares_ssn')
      AND ge.is_active = true
      AND gn2.id != ALL(fn.path)
      AND fn.depth < 3

    UNION ALL

    -- Incoming edges to current node
    SELECT
        gn3.id,
        gn3.label,
        gn3.entity_id,
        fn2.depth + 1,
        fn2.path || gn3.id,
        ge2.edge_type
    FROM fraud_network fn2
    JOIN graph_edge ge2 ON ge2.target_node_id = fn2.node_id
    JOIN graph_node gn3 ON gn3.id = ge2.source_node_id
    WHERE ge2.edge_type IN ('shares_address', 'shares_phone', 'shares_email',
                            'shares_employer', 'shares_device', 'shares_ssn')
      AND ge2.is_active = true
      AND gn3.id != ALL(fn2.path)
      AND fn2.depth < 3
)
SELECT DISTINCT entity_id, label, depth,
       array_agg(DISTINCT connection_type) AS connection_types
FROM fraud_network
WHERE node_id != (SELECT id FROM graph_node WHERE entity_id = '<suspect_party_uuid>' AND node_type = 'party')
GROUP BY entity_id, label, depth
ORDER BY depth;
```

### Portfolio Concentration Analysis

```sql
-- Total exposure by guarantor across all guaranteed loans
SELECT
    gn_guarantor.label AS guarantor_name,
    gn_guarantor.entity_id AS guarantor_id,
    COUNT(DISTINCT gn_app.entity_id) AS guaranteed_loan_count,
    SUM(la.approved_amount) AS total_guaranteed_exposure,
    SUM((ge.properties->>'guarantee_amount')::decimal) AS total_guarantee_amount
FROM graph_node gn_guarantor
JOIN graph_edge ge ON ge.source_node_id = gn_guarantor.id
    AND ge.edge_type = 'guarantor_for' AND ge.is_active = true
JOIN graph_node gn_app ON gn_app.id = ge.target_node_id
    AND gn_app.node_type = 'application'
JOIN loan_application la ON la.id = gn_app.entity_id
    AND la.status IN ('approved', 'funded')
WHERE gn_guarantor.node_type = 'party'
GROUP BY gn_guarantor.label, gn_guarantor.entity_id
HAVING SUM(la.approved_amount) > 1000000
ORDER BY total_guaranteed_exposure DESC;

-- Geographic concentration: total exposure by census tract
SELECT
    gn_addr.properties->>'census_tract' AS census_tract,
    gn_addr.properties->>'county_fips' AS county,
    COUNT(DISTINCT gn_app.entity_id) AS loan_count,
    SUM(la.approved_amount) AS total_exposure
FROM graph_node gn_addr
JOIN graph_edge ge_loc ON ge_loc.target_node_id = gn_addr.id
    AND ge_loc.edge_type = 'located_at' AND ge_loc.is_active = true
JOIN graph_node gn_coll ON gn_coll.id = ge_loc.source_node_id
    AND gn_coll.node_type = 'collateral'
JOIN graph_edge ge_sec ON ge_sec.target_node_id = gn_coll.id
    AND ge_sec.edge_type = 'secured_by' AND ge_sec.is_active = true
JOIN graph_node gn_app ON gn_app.id = ge_sec.source_node_id
    AND gn_app.node_type = 'application'
JOIN loan_application la ON la.id = gn_app.entity_id
    AND la.status IN ('approved', 'funded')
WHERE gn_addr.node_type = 'address'
GROUP BY gn_addr.properties->>'census_tract', gn_addr.properties->>'county_fips'
ORDER BY total_exposure DESC
LIMIT 20;
```

### Conflict of Interest Detection

```sql
-- Check if a loan officer has any relationship to the borrower
-- (shared address, shared employer, family connection)
SELECT
    ge.edge_type AS relationship,
    ge.properties,
    gn_intermediate.label AS connection_through
FROM graph_node gn_officer
JOIN graph_edge ge ON (ge.source_node_id = gn_officer.id OR ge.target_node_id = gn_officer.id)
    AND ge.is_active = true
JOIN graph_node gn_intermediate ON gn_intermediate.id =
    CASE WHEN ge.source_node_id = gn_officer.id THEN ge.target_node_id
         ELSE ge.source_node_id END
WHERE gn_officer.node_type = 'staff'
  AND gn_officer.entity_id = '<loan_officer_uuid>'
  AND gn_intermediate.id IN (
      -- All nodes connected to the borrower
      SELECT CASE WHEN ge2.source_node_id = gn_borrower.id THEN ge2.target_node_id
                  ELSE ge2.source_node_id END
      FROM graph_node gn_borrower
      JOIN graph_edge ge2 ON (ge2.source_node_id = gn_borrower.id OR ge2.target_node_id = gn_borrower.id)
          AND ge2.is_active = true
      WHERE gn_borrower.node_type = 'party'
        AND gn_borrower.entity_id = '<borrower_party_uuid>'
  );
```

### Cross-Collateralisation Detection

```sql
-- Find all loans sharing collateral with a given application
SELECT
    la2.application_number,
    la2.status,
    la2.approved_amount,
    gn_coll.label AS shared_collateral,
    ge1.properties->>'lien_position' AS lien_position_this_loan,
    ge2.properties->>'lien_position' AS lien_position_other_loan
FROM graph_node gn_app1
JOIN graph_edge ge1 ON ge1.source_node_id = gn_app1.id
    AND ge1.edge_type = 'secured_by' AND ge1.is_active = true
JOIN graph_node gn_coll ON gn_coll.id = ge1.target_node_id
    AND gn_coll.node_type = 'collateral'
JOIN graph_edge ge2 ON ge2.target_node_id = gn_coll.id
    AND ge2.edge_type = 'secured_by' AND ge2.is_active = true
    AND ge2.source_node_id != gn_app1.id
JOIN graph_node gn_app2 ON gn_app2.id = ge2.source_node_id
    AND gn_app2.node_type = 'application'
JOIN loan_application la2 ON la2.id = gn_app2.entity_id
WHERE gn_app1.node_type = 'application'
  AND gn_app1.entity_id = '<application_uuid>';
```

---

## Graph Synchronisation

```sql
-- Trigger function to keep graph nodes in sync with relational tables
-- (simplified example for the party table)
CREATE OR REPLACE FUNCTION sync_party_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO graph_node (node_type, entity_id, label, properties)
        VALUES (
            'party',
            NEW.id,
            NEW.display_name,
            jsonb_build_object(
                'party_type', NEW.party_type,
                'status', NEW.status,
                'email', NEW.email
            )
        );
    ELSIF TG_OP = 'UPDATE' THEN
        UPDATE graph_node
        SET label = NEW.display_name,
            properties = jsonb_build_object(
                'party_type', NEW.party_type,
                'status', NEW.status,
                'email', NEW.email
            ),
            updated_at = now()
        WHERE node_type = 'party' AND entity_id = NEW.id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE graph_node SET is_active = false, updated_at = now()
        WHERE node_type = 'party' AND entity_id = OLD.id;
        UPDATE graph_edge SET is_active = false, valid_to = now(), updated_at = now()
        WHERE source_node_id IN (SELECT id FROM graph_node WHERE entity_id = OLD.id)
           OR target_node_id IN (SELECT id FROM graph_node WHERE entity_id = OLD.id);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_party_graph_sync
    AFTER INSERT OR UPDATE OR DELETE ON party
    FOR EACH ROW EXECUTE FUNCTION sync_party_to_graph();

-- Similar triggers for loan_application, collateral, address, staff, organisation
```

---

## HMDA & Compliance Tables

```sql
-- HMDA data (relational for regulatory reporting)
CREATE TABLE hmda_data (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id) UNIQUE,
    lei VARCHAR(20) NOT NULL,
    uli VARCHAR(45) NOT NULL,
    action_taken INT NOT NULL,
    action_taken_date DATE,
    loan_type INT NOT NULL,
    loan_purpose INT NOT NULL,
    loan_amount DECIMAL(14,2) NOT NULL,
    state_code VARCHAR(2),
    county_code VARCHAR(3),
    census_tract VARCHAR(11),
    applicant_ethnicity_1 INT,
    applicant_race_1 INT,
    applicant_sex INT,
    interest_rate DECIMAL(6,4),
    dti_ratio VARCHAR(10),
    denial_reason_1 INT,
    reporting_year INT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_hmda_year ON hmda_data(reporting_year);

-- Adverse action reasons
CREATE TABLE adverse_action_reason (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    decision_id UUID NOT NULL REFERENCES underwriting_decision(id),
    reason_code VARCHAR(10) NOT NULL,
    reason_text VARCHAR(255) NOT NULL,
    sort_order INT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- TRID disclosure tracking
CREATE TABLE trid_disclosure (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES loan_application(id),
    disclosure_type VARCHAR(30) NOT NULL,
    version_number INT NOT NULL DEFAULT 1,
    issued_date DATE NOT NULL,
    received_date DATE,
    waiting_period_end DATE,
    is_compliant BOOLEAN,
    document_id UUID REFERENCES document(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_trid_app ON trid_disclosure(application_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Party Management | 2 | party, address (addresses are graph nodes for dedup) |
| Loan Application | 1 | loan_application |
| Collateral | 1 | collateral |
| Credit & Underwriting | 1 | underwriting_decision |
| Documents | 1 | document |
| Organisation & Staff | 3 | organisation, branch, staff |
| Compliance | 3 | hmda_data, adverse_action_reason, trid_disclosure |
| Audit | 1 | audit_log |
| Product | 1 | loan_product |
| **Graph Layer** | **2** | **graph_node, graph_edge** |
| **Total** | **16** | Plus triggers for graph sync |

---

## Key Design Decisions

1. **Graph overlay, not graph replacement**: The relational tables handle CRUD operations and transactional integrity. The graph layer adds relationship intelligence. This avoids the operational risks of a graph-only data store (immature backup, monitoring, and recovery tooling) while gaining graph query performance for relationship-heavy analytics.

2. **Generic node/edge model in PostgreSQL**: Using `graph_node` and `graph_edge` tables keeps everything in a single PostgreSQL database, avoiding the operational overhead of a separate graph database (Neo4j, Neptune). For teams that outgrow this approach, the data can be exported to a dedicated graph database later.

3. **Temporal edges with valid_from/valid_to**: Relationships change over time (a party moves to a new address, an officer leaves an organisation). Temporal validity on edges enables point-in-time graph queries ("what was the ownership chain on the date of underwriting?").

4. **Address as a first-class graph node**: Addresses are normalised and deduplicated (via normalised_hash). This enables shared-address detection across parties — a key fraud signal. When two unrelated applicants share the same address node, the graph makes this immediately visible.

5. **Trigger-based graph synchronisation**: PostgreSQL triggers keep graph nodes in sync with relational tables automatically. This ensures consistency without requiring application code to maintain both layers, though it adds write overhead.

6. **Edge properties for relationship metadata**: Ownership percentages, guarantee amounts, lien positions, and fraud detection confidence scores are stored as JSONB properties on edges. This avoids creating separate junction tables for each relationship type while preserving the ability to query on these attributes.

7. **Fraud detection edges as first-class relationships**: Shared identifiers (SSN, phone, email, address, device) between parties are modelled as explicit graph edges with confidence scores. This turns fraud ring detection from a complex multi-table join into a simple graph traversal.

8. **Denormalised properties on graph nodes**: Graph nodes carry denormalised copies of key fields (display_name, risk_grade, total_exposure) to enable graph traversal queries that do not need to join back to relational tables. This trades some storage for significant query performance.

9. **FinCEN beneficial ownership alignment**: Ownership edges carry `ownership_pct` and `ownership_type` properties, directly supporting FinCEN beneficial ownership reporting requirements (25%+ ownership threshold triggers) and Corporate Transparency Act compliance.

10. **Separation of concerns for compliance tables**: HMDA, TRID, and adverse action data remain in conventional relational tables because regulatory reporting requires exact field-level compliance with structured formats. The graph layer adds analytical capability on top without interfering with compliance data integrity.
