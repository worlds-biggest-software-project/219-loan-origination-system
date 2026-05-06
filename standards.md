# Standards & API Reference

> Project: Loan Origination System · Generated: 2026-05-03

## Industry Standards & Specifications

### Mortgage Industry Standards (MISMO)

**MISMO Reference Model (v3.4 / v3.6)**
- URL: https://www.mismo.org/standards-resources/residential-specifications/reference-model
- The Mortgage Industry Standards Maintenance Organization (MISMO) Reference Model is the canonical data interchange standard for the US mortgage industry. Required by Fannie Mae, Freddie Mac, Ginnie Mae, FHA, and VA. Defines the logical data model (XML and JSON) for all loan origination, delivery, and servicing transactions. Version 3.4 underpins the URLA and AUS submission formats; version 3.6 introduces the Loan Boarding Data Segment (LBDS).

**MISMO iLAD (Industry Loan Application Dataset) v2.2.0**
- URL: https://www.mismo.org/standards-resources/mismo-product/industry-loan-application-dataset-ilad-2-2-0
- A compendium of data points enabling seamless exchange of loan application data between industry partners. Maps Form 1003 fields to the MISMO v3.4 Reference Model. Used by origination platforms to standardise application intake and data handoff.

**MISMO API Toolkit**
- URL: https://www.mismo.org/standards-resources/apis
- Provides guidance on adopting microservices architecture for mortgage APIs. Allows organisations to design MISMO-aligned data structures for specific business use cases and accelerates REST API development within the mortgage ecosystem.

**ULAD (Uniform Loan Application Dataset)**
- URL: https://singlefamily.fanniemae.com/delivering/uniform-mortgage-data-program/uniform-residential-loan-application
- Published jointly by Fannie Mae and Freddie Mac. Maps redesigned Form 1003 (URLA) fields to MISMO v3.4 data points. Required mapping for any LOS that submits loans to the GSEs via Desktop Underwriter (DU) or Loan Product Advisor (LPA).

**ULDD (Uniform Loan Delivery Dataset)**
- URL: https://singlefamily.fanniemae.com/delivering/uniform-mortgage-data-program/uniform-loan-delivery-dataset
- Defines the mandatory data set for loan delivery to Fannie Mae and Freddie Mac after closing. Any LOS must produce ULDD-compliant XML files; used to validate loan completeness before securitisation.

### Regulatory & Compliance Standards

**TRID — TILA-RESPA Integrated Disclosures (Regulation Z / Regulation X)**
- URL: https://www.consumerfinance.gov/compliance/compliance-resources/mortgage-resources/tila-respa-integrated-disclosures/
- Implemented by the CFPB under the Dodd-Frank Act (effective October 2015). Requires all residential mortgage lenders to deliver a Loan Estimate (LE) within three business days of application, and a Closing Disclosure (CD) at least three business days before closing. All US residential LOS must generate TRID-compliant documents; forms are tightly governed (tolerance levels, APR calculations, fee categorisation).

**HMDA (Home Mortgage Disclosure Act) — Regulation C (12 CFR Part 1003)**
- URL: https://ffiec.cfpb.gov/
- Requires financial institutions to collect and report 110 data fields on covered loan applications and originations. Reporting is annual via the FFIEC HMDA Platform (open-source backend: https://github.com/cfpb/hmda-platform). The platform exposes public-facing REST APIs including a Ratespread Calculator API. Any mortgage LOS handling regulated institutions must capture and export HMDA-required fields.

**ECOA — Equal Credit Opportunity Act (Regulation B, 12 CFR Part 1002)**
- URL: https://www.consumerfinance.gov/compliance/compliance-resources/other-applicable-requirements/equal-credit-opportunity-act/
- Prohibits credit discrimination based on protected characteristics. Requires adverse action notices with specific reasons when credit is denied. Directly affects AI-driven underwriting: the CFPB has affirmed that "complex algorithms" must still provide specific, human-understandable reasons for adverse decisions. A LOS must capture and surface these reasons for every declined application.

**FCRA — Fair Credit Reporting Act (Regulation V)**
- URL: https://www.ftc.gov/legal-library/browse/statutes/fair-credit-reporting-act
- Governs the use of consumer credit reports in lending decisions. Requires disclosure of credit score and related information when used in adverse action or risk-based pricing (amended by Dodd-Frank). Any LOS that pulls credit data must implement FCRA-compliant adverse action workflows and credit score disclosures.

**ESIGN Act & UETA — Electronic Signatures in Global and National Commerce**
- URL: https://www.docusign.com/products/electronic-signature/learn/esign-act-ueta
- The federal ESIGN Act (2000) and state-level UETA (adopted by 47 states) establish the legal equivalence of electronic signatures to wet-ink signatures on loan documents. A LOS must integrate with a compliant e-signature provider, maintain tamper-evident audit trails, and capture borrower consent to conduct business electronically.

### Financial Messaging & Data Standards

**ISO 20022 — Financial Services Universal Financial Industry Message Scheme**
- URL: https://www.iso20022.org/about-iso-20022
- The global standard for electronic data interchange between financial institutions, covering payments, securities, credit, and trade finance. ISO 20022 messages are format-agnostic (XML and JSON supported). Relevant for a LOS that needs to integrate with core banking systems or payment rails (e.g., loan disbursement via Fedwire, which adopted ISO 20022 in July 2025). Defines structured credit and trade finance message types applicable to commercial loan origination.

**Open Banking UK — SME Loan API Specification (v2.4.0)**
- URL: https://openbankinguk.github.io/opendata-api-docs-pub/v2.4.0/smeloan/sme-loan.html
- Published by Open Banking Limited (UK). Defines a standardised REST API for SME unsecured loan product data, enabling comparison-site and third-party access to lender product information. A useful reference data model for consumer and SMB loan origination APIs outside the US mortgage context.

**Open Banking UK — API Standards (v4.0.1)**
- URL: https://standards.openbanking.org.uk/api-specifications/latest/
- Comprehensive UK open banking standards including OAuth 2.0 / OpenID Connect authentication, third-party provider onboarding, and read/write API conventions. Defines the security and API interaction patterns that a modern LOS API should follow for open-banking-compatible integrations.

### API & Data Format Specifications

**OpenAPI Specification v3.1.0**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- The industry standard for describing RESTful APIs in a machine-readable format. OAS 3.1 aligns fully with JSON Schema (Draft 2020-12). All major LOS vendors (ICE Encompass, LoanPro, Freddie Mac, Fannie Mae) publish OpenAPI-compliant API documentation. A new LOS should expose its own API via an OAS 3.1 specification.

**JSON Schema (IETF Internet Draft / Draft 2020-12)**
- URL: https://datatracker.ietf.org/doc/charter-ietf-jsonschema/00-03/
- Defines the vocabulary for describing and validating JSON data structures. Used within OpenAPI 3.1 and by MISMO for JSON-format message definitions. Essential for validating loan application payloads at API boundaries.

**OAuth 2.0 (RFC 6749) & OpenID Connect 1.0**
- URL: https://datatracker.ietf.org/doc/rfc6749/ and https://openid.net/connect/
- De-facto authentication and authorisation standards for modern financial APIs. Required by Fannie Mae, Freddie Mac, ICE Encompass, and Open Banking frameworks. A LOS API must implement OAuth 2.0 for machine-to-machine integrations and OpenID Connect for user-facing authentication flows.

### Model Risk & AI Standards

**FFIEC / OCC Model Risk Management Guidance (SR 11-7 / OCC 2011-12)**
- URL: https://www.federalreserve.gov/supervisionreg/srletters/sr1107.htm
- Federal supervisory guidance on managing risks from models used in credit decisions (including AI/ML underwriting models). Requires model validation, documentation, and ongoing monitoring. Any AI-powered decisioning component in a LOS must be built with this framework in mind for regulated-institution customers.

**NIST AI Risk Management Framework (AI RMF 1.0)**
- URL: https://www.nist.gov/system/files/documents/2023/01/26/NIST.AI.100-1.pdf
- NIST's voluntary framework for managing risks associated with AI systems across their lifecycle. Increasingly referenced by bank regulators as a benchmark for responsible AI in credit decisioning. Relevant for documenting fairness, explainability, and robustness of ML-based underwriting modules.

---

## Similar Products — Developer Documentation & APIs

### ICE Encompass (ICE Mortgage Technology)
- **Description:** Market-leading US residential mortgage LOS. Encompasses the full loan lifecycle from point-of-sale through post-closing and secondary market delivery.
- **API Documentation:** https://developer.icemortgagetechnology.com/developer-connect/docs/welcome
- **SDKs/Libraries:** REST API with OAuth 2.0; legacy .NET SDK (being deprecated in favour of REST APIs); SDK-to-API migration guide available.
- **Developer Guide:** https://developer.icemortgagetechnology.com/developer-connect/docs/sdk-to-api-migration-getting-started-guide
- **Standards:** REST/JSON; OAuth 2.0; MISMO v3.x data model; ULAD/URLA compliance.
- **Authentication:** OAuth 2.0 (client credentials and authorisation code flows).

### Fannie Mae Developer Portal
- **Description:** Fannie Mae's APIs cover loan origination, automated underwriting (Desktop Underwriter / DU), pricing, appraisal, and delivery to the GSE.
- **API Documentation:** https://developer.fanniemae.com/
- **SDKs/Libraries:** REST APIs; JSON payloads; no official SDK published; MISMO v3.4 XML schemas for AUS submissions.
- **Developer Guide:** https://singlefamily.fanniemae.com/applications-technology/application-programming-interfaces-apis-developer-portal
- **Standards:** REST/JSON and MISMO XML; ULAD/ULDD data models; OpenAPI-documented endpoints.
- **Authentication:** OAuth 2.0 (business partner credentials required for non-public APIs).

### Freddie Mac API Solutions
- **Description:** Freddie Mac's origination APIs provide access to loan data, Loan Product Advisor (LPA) automated underwriting, appraisal, and loan delivery.
- **API Documentation:** https://sf.freddiemac.com/tools-learning/apis/our-api-solutions
- **SDKs/Libraries:** REST APIs; JSON and MISMO XML supported; developer portal with sandbox.
- **Developer Guide:** https://sf.freddiemac.com/tools-learning/apis/getting-started-with-apis
- **Standards:** REST/JSON; MISMO v3.x; ULDD; ULAD.
- **Authentication:** OAuth 2.0 via developer portal credentials.

### LoanPro LMS API
- **Description:** API-first loan management and origination platform for consumer and commercial lenders. Covers the full lifecycle from application through servicing.
- **API Documentation:** https://developers.loanpro.io/docs/origination-with-loanpro
- **SDKs/Libraries:** REST API; JSON payloads; webhooks for event-driven workflows.
- **Developer Guide:** https://developers.loanpro.io/docs/lms-api-introduction
- **Standards:** REST/JSON; OpenAPI-documented; webhook-based event notifications.
- **Authentication:** API token-based authentication (tokens can be scoped and rotated).

### Apache Fineract
- **Description:** Open-source (Apache 2.0) core banking platform with loan origination, disbursement, repayment scheduling, and portfolio management. Used by microfinance institutions and fintechs globally.
- **API Documentation:** https://fineract.apache.org/docs/current/
- **SDKs/Libraries:** REST API (Spring Boot); Swagger/OpenAPI live documentation at https://demo.mifos.io/api-docs/apiLive.htm; community SDKs in Java and Python.
- **Developer Guide:** https://github.com/apache/fineract
- **Standards:** REST/JSON; OpenAPI 2.0 (Swagger); HTTPS-only; stateless communication.
- **Authentication:** Basic Auth and OAuth 2.0 (configurable per deployment).

### DigiFi Open-Source LOS
- **Description:** Self-described as the world's first open-source loan origination system. Node.js-based platform covering application intake, credit decisioning, document management, and portfolio analytics.
- **API Documentation:** https://docs.digifi.io/docs/the-digifi-loan-origination-system
- **SDKs/Libraries:** REST APIs; webhooks; JavaScript/Node.js SDK; integration builder.
- **Developer Guide:** https://docs.digifi.io/
- **Standards:** REST/JSON; webhook event model.
- **Authentication:** API key / token-based.
- **Licence:** Open-source (MIT / proprietary dual licence for enterprise features); GitHub: https://github.com/digifi-io

### MuleSoft Accelerator for Financial Services — FINS Loan Origination System API
- **Description:** Pre-built API specification and implementation template for loan origination, covering customer onboarding, credit scoring, document e-signature, and LOS integration within the MuleSoft/Anypoint ecosystem.
- **API Documentation:** https://anypoint.mulesoft.com/exchange/org.mule.examples/fins-loan-origination-sys-api-spec/
- **SDKs/Libraries:** RAML/OAS specification; MuleSoft Anypoint connectors; canonical FINS Banking Library data types.
- **Developer Guide:** https://docs.mulesoft.com/financial-services/latest/
- **Standards:** REST/JSON; RAML 1.0 / OpenAPI; canonical data model aligned to FSC objects.
- **Authentication:** OAuth 2.0.

### Salesforce Financial Services Cloud (FSC)
- **Description:** CRM and workflow platform with native loan origination objects (LoanApplicant, LoanApplicationFinancial, LoanApplicationProperty, etc.). Used by nCino and other ISVs as a platform layer for commercial and consumer LOS.
- **API Documentation:** https://developer.salesforce.com/docs/atlas.en-us.financial_services_cloud_object_reference.meta/financial_services_cloud_object_reference/fsc_api_access_and_usage.htm
- **SDKs/Libraries:** Salesforce REST and SOAP APIs; Apex; Lightning Web Components; Connect REST API.
- **Developer Guide:** https://developer.salesforce.com/docs/industries/fsc/overview
- **Standards:** REST/JSON; SOAP/XML; OpenAPI-like Swagger docs via Salesforce API Explorer.
- **Authentication:** OAuth 2.0 (Salesforce identity platform).

### CFPB HMDA Platform
- **Description:** Open-source regulatory technology application used by financial institutions to submit HMDA data. Exposes public REST APIs including a Ratespread Calculator.
- **API Documentation:** https://ffiec.cfpb.gov/
- **SDKs/Libraries:** Open-source Scala/Play backend; GitHub: https://github.com/cfpb/hmda-platform.
- **Developer Guide:** Available via the FFIEC HMDA Filing Instructions Guide (FIG).
- **Standards:** REST/JSON; streaming CSV upload support; public API endpoints.
- **Authentication:** Public APIs are unauthenticated; institution filing portal uses OAuth.

### Floify Mortgage POS API
- **Description:** Mortgage point-of-sale and automation platform. Provides open REST APIs for LOS integrations, borrower portals, and document collection workflows.
- **API Documentation:** https://floify.com/blog/mortgage-real-estate-open-api
- **SDKs/Libraries:** REST API; webhooks; free developer accounts available.
- **Developer Guide:** Developer portal (requires account registration).
- **Standards:** REST/JSON; webhook event notifications.
- **Authentication:** API key / OAuth token.

---

## Notes

### Gaps and Evolving Areas

- **Commercial Loan Standards:** MISMO focuses on residential mortgage. Commercial and SMB loan origination lacks a comparable universal standard; most vendors use proprietary data models or ISO 20022 financial messaging for commercial credit events. The LSTA (Loan Syndications and Trading Association) publishes data standards for syndicated loans but these are not publicly available without membership.

- **AI/ML Decisioning Transparency:** The intersection of ECOA adverse action requirements and black-box ML models remains an active regulatory debate. The CFPB's 2023 guidance on "complex algorithms" has not yet been codified as a formal standard. Vendors are adopting SHAP/LIME-based explainability tooling as a de-facto approach.

- **Open Banking Beyond UK/EU:** In the US, open banking rulemaking (CFPB Section 1033 / Personal Financial Data Rights Rule, finalized October 2024) will eventually mandate consumer data portability from lenders, which will create new API obligations for LOS platforms.

- **eClosing and eNote Standards:** MISMO eMortgage standards (eNote, eVault) are gaining traction but adoption is still limited relative to traditional paper-based closing. MERS (Mortgage Electronic Registration System) provides the eNote registry infrastructure.
