# Standards & API Reference

> Project: Healthcare Outcomes Analytics · Generated: 2026-05-06

---

## Industry Standards & Specifications

### HL7 Standards

**HL7 FHIR R4 (Fast Healthcare Interoperability Resources, Release 4)**
- URL: https://hl7.org/fhir/R4/
- The primary normative standard for representing and exchanging electronic health records via RESTful APIs. FHIR R4 is the mandatory base for all ONC and CMS interoperability rule compliance. A healthcare outcomes analytics platform must ingest, store, and serve data in FHIR R4 format to interoperate with Epic, Oracle Health, payer systems, and cloud FHIR stores.

**HL7 FHIR Bulk Data Access (Flat FHIR / $export)**
- URL: https://hl7.org/fhir/uv/bulkdata/
- Defines the asynchronous population-level data export pattern ($export) using NDJSON files. Required for extracting large patient cohorts from EHRs for analytics pipelines without per-patient API calls. Supported by Epic, Oracle Health, and major cloud FHIR services.

**HL7 v2 Messaging Standard**
- URL: https://www.hl7.org/implement/standards/product_brief.cfm?product_id=185
- The legacy standard for real-time clinical event messaging (ADT, ORU, ORM messages) still dominant in hospital operational systems. Analytics platforms must ingest HL7 v2 messages for real-time admission, discharge, and lab result signals alongside bulk FHIR.

**HL7 Quality Improvement Core (QI-Core) Implementation Guide**
- URL: http://hl7.org/fhir/us/qicore/
- Defines a set of FHIR profiles and extensions for quality improvement and clinical decision support applications. QI-Core profiles are the required data model for FHIR-based eCQM calculation; all quality measure logic must reference QI-Core-profiled resources.

**HL7 CQF-Measures (Quality Measure Implementation Guide)**
- URL: https://build.fhir.org/ig/HL7/cqf-measures/
- Describes how to represent clinical quality measures using FHIR and the Clinical Quality Language (CQL). The definitive specification for building FHIR-native HEDIS and eCQM measure logic. Updated to v5.0.0 with alignment to CMS and NCQA digital quality measure roadmaps.

**HL7 Da Vinci Data Exchange for Quality Measures (DEQM)**
- URL: http://hl7.org/fhir/us/davinci-deqm/
- Defines FHIR-based data exchange patterns for quality measure reporting between providers, payers, and quality reporting organisations. Specifies how to submit measure reports and individual patient-level data for HEDIS and CMS Stars programmes.

**Clinical Quality Language (CQL) Specification v1.5.3**
- URL: https://cql.hl7.org/
- A high-level domain-specific language for expressing clinical quality measure logic and clinical decision support rules in a computable, shareable format. All FHIR-based eCQMs and digital HEDIS measures are expressed in CQL; the analytics platform must embed a CQL execution engine.

**SMART on FHIR App Launch Framework v2.2.0**
- URL: https://build.fhir.org/ig/HL7/smart-app-launch/
- Defines the standards for launching FHIR-based applications securely within EHR contexts (clinician-facing) and standalone (patient-facing). Required for embedding population health dashboards and care gap alerts directly into Epic and Oracle Health EHR workflows.

**US Core Implementation Guide**
- URL: https://www.hl7.org/fhir/us/core/
- The US-realm FHIR profile set that ONC mandates for certified health IT. Analytics platforms operating in the US must be able to ingest and query US Core-profiled resources as the baseline patient data representation.

---

### IHE Profiles

**IHE QEDm — Query for Existing Data for Mobile**
- URL: https://build.fhir.org/ig/IHE/PCC.QEDm/
- An IHE profile for querying structured clinical data elements (observations, conditions, medications, procedures) over FHIR R4 RESTful APIs. Provides a standardised query interface that analytics platforms can implement to pull clinical data from compliant sources without bespoke integration.

---

### W3C & IETF Standards

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Governs the HTTP methods, status codes, and content negotiation used by all FHIR REST APIs. All FHIR server and client implementations must conform to this RFC.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Defines the Link header used in FHIR pagination responses for bulk data export and search result sets. Required for correct implementation of FHIR bundle navigation.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Specifies the JWT format used in SMART on FHIR and OAuth 2.0 bearer token flows for authenticating API requests to FHIR servers and analytics services.

---

### Data Model & API Specifications

**OMOP Common Data Model (CDM) v5.4**
- URL: https://ohdsi.github.io/CommonDataModel/
- The OHDSI-maintained observational research data model used for cross-institutional analytics and clinical research. OMOP CDM is the preferred target format for analytics pipelines that need to run validated epidemiological studies; it scores alongside FHIR as the highest-performing healthcare data standard in systematic reviews.

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The standard for documenting REST APIs. All analytics platform APIs should be documented using OpenAPI 3.1 to enable code generation, testing, and developer adoption.

**NDJSON (Newline-Delimited JSON)**
- URL: https://ndjson.org/
- The mandatory output format for FHIR Bulk Data $export operations. Analytics ingestion pipelines must parse NDJSON streams to process population-scale EHR exports.

**Apache Parquet / Apache Iceberg**
- URL: https://parquet.apache.org/ / https://iceberg.apache.org/
- The de-facto columnar storage formats for healthcare data lakes. AWS HealthLake automatically converts FHIR to Apache Iceberg; analytics platforms should store normalised FHIR data in Parquet/Iceberg format for SQL-on-FHIR queries at scale.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) and OpenID Connect (OIDC)**
- URL: https://oauth.net/2/ / https://openid.net/connect/
- The mandatory authentication and authorisation framework for FHIR APIs, SMART on FHIR, and all inter-system data exchange. Analytics platforms must implement OAuth 2.0 client credentials flow for back-end bulk data access and SMART on FHIR launch for in-EHR app integration.

**HIPAA Security Rule (45 CFR Part 164)**
- URL: https://www.hhs.gov/hipaa/for-professionals/security/index.html
- US federal regulation mandating administrative, physical, and technical safeguards for electronic protected health information (ePHI). All data storage, processing, and transmission in a healthcare analytics platform must comply with the HIPAA Security Rule.

**NIST SP 800-66 Revision 2 — Implementing the HIPAA Security Rule**
- URL: https://csrc.nist.gov/pubs/sp/800/66/r2/final
- NIST's practical cybersecurity guidance for HIPAA compliance, published February 2024. Provides a crosswalk between HIPAA Security Rule requirements, NIST CSF, and NIST SP 800-53 controls. The reference framework for security architecture reviews of healthcare analytics systems.

**NIST Cybersecurity Framework (CSF) 2.0**
- URL: https://www.nist.gov/cyberframework
- The voluntary framework for managing cybersecurity risk. Healthcare analytics platforms should map their security controls to CSF 2.0 functions (Govern, Identify, Protect, Detect, Respond, Recover) as the baseline for security programme governance.

**ONC 21st Century Cures Act Interoperability Rule (45 CFR Part 170)**
- URL: https://www.healthit.gov/curesrule/
- Mandates that certified EHRs support FHIR R4 patient access and provider access APIs, and prohibits information blocking. Analytics platforms receiving data from certified EHRs must align with these access patterns and cannot engage in practices that would constitute information blocking.

---

### Quality Measure Programme Specifications

**NCQA HEDIS Measurement Year 2026 Specifications**
- URL: https://www.ncqa.org/hedis/
- The annual Healthcare Effectiveness Data and Information Set published by NCQA, defining the measure set used by health plans for quality reporting and CMS Stars ratings. MY 2026 adds seven new measures, retires two, and transitions four measures to Electronic Clinical Data Systems (ECDS) format. All HEDIS-supporting analytics platforms must update measure logic annually.

**CMS eCQM Library 2026 — Electronic Clinical Quality Measure Specifications**
- URL: https://ecqi.healthit.gov/updated-ecqm-specifications-and-implementation-resources-2026-reporting/performance-period
- CMS's official eCQM specifications for the 2026 reporting period, covering hospital inpatient, hospital outpatient, and eligible clinician programmes. Expressed in CQL/FHIR format; analytics platforms must implement these specifications to support regulatory submission.

**AHRQ Quality Indicators (IQI, PSI, PQI, PDI)**
- URL: https://www.ahrq.gov/data/qualityindicators/index.html
- Standardised, government-validated quality measure sets for inpatient quality (IQI), patient safety (PSI), prevention quality (PQI), and paediatric quality (PDI) indicators. Free reference implementations in SAS provide validated calculation logic that analytics platforms can port to FHIR-native CQL.

**CMS Hierarchical Condition Categories (HCC) Risk Adjustment Model**
- URL: https://www.cms.gov/medicare/payment/medicare-advantage/risk-adjustment
- The CMS model for calculating risk scores from diagnosis codes for Medicare Advantage and value-based care payment. Analytics platforms supporting risk adjustment must implement HCC category mapping and coding gap identification logic.

---

## Similar Products — Developer Documentation & APIs

### Epic on FHIR / open.epic

- **Description:** The FHIR developer programme for Epic, the largest EHR vendor in the US. Provides 800+ free FHIR R4 APIs covering clinical, administrative, and financial data. The primary source of patient and population data for analytics platforms deployed in Epic-heavy health systems.
- **API Documentation:** https://fhir.epic.com/Documentation
- **Developer Portal:** https://open.epic.com/
- **Bulk Data (Analytics) Guide:** https://fhir.epic.com/Documentation (FHIR Bulk Data Access tutorial)
- **Sandbox:** Available via Epic on FHIR developer registration
- **Standards:** FHIR R4, SMART on FHIR 2.0, US Core, Bulk FHIR ($export)
- **Authentication:** SMART on FHIR (EHR launch and standalone), OAuth 2.0 client credentials for back-end bulk data

---

### Oracle Health Data Intelligence (Cerner) APIs

- **Description:** Oracle's population health and analytics platform built on Cerner Millennium. Provides REST APIs for patient records, cohort management, care registries, and observations. Second-largest EHR data source in the US after Epic.
- **API Documentation:** https://docs.healtheintent.com/
- **FHIR R4 APIs:** https://docs.oracle.com/en/industries/health/millennium-platform-apis/mfrap/r4_overview.html
- **Developer Sandbox:** cernerdemo tenant — full production-level tenant with synthetic data
- **Key APIs:** Cohort API v1, HealtheRegistries API, Patient API v1, Observation API v1, Condition API v1
- **Standards:** FHIR R4, HL7 v2
- **Authentication:** Bearer token and OAuth 1.0a

---

### Innovaccer Data Activation Platform

- **Description:** Commercial healthcare data platform unifying EHR, claims, and SDOH data with FHIR+ APIs and quality measure analytics. 65+ pre-built EHR connectors; supports HEDIS, Stars, and MIPS measure reporting.
- **API Documentation:** Requires vendor engagement; limited public documentation
- **AWS Marketplace:** https://aws.amazon.com/marketplace/pp/prodview-5mdtsqcptte7m
- **Azure Marketplace:** https://azuremarketplace.microsoft.com/en-us/marketplace/apps/innovaccerinc.innovaccer-dap
- **Standards:** FHIR R4, FHIR+ (Level 4 graph model), SMART on FHIR
- **Authentication:** OAuth 2.0, SMART on FHIR

---

### Arcadia Data Platform APIs

- **Description:** Value-based care analytics platform with a data lakehouse, quality measure tracking, and SMART on FHIR point-of-care integration. Strong in payer-provider network analytics and CMS BCDA claims data ingestion.
- **API Documentation:** https://docs.arcadia.com/
- **Developer Resources:** https://dev.arcadia.io/resources
- **Key APIs:** Plug API (integration), Signal API (real-time data streams)
- **Standards:** FHIR R4, SMART on FHIR, REST/JSON
- **Authentication:** OAuth 2.0 (details via vendor engagement)

---

### AWS HealthLake

- **Description:** Fully managed HIPAA-eligible FHIR R4 data store with automatic conversion to Apache Iceberg for SQL-on-FHIR analytics, NLP via Comprehend Medical, and native AWS ML integration. Best for analytics platforms building on AWS infrastructure.
- **API Documentation:** https://docs.aws.amazon.com/healthlake/
- **Features Page:** https://aws.amazon.com/healthlake/features/
- **SDKs:** AWS SDK for Python (Boto3), Java, JavaScript, Go, .NET
- **Standards:** FHIR R4 v4.0.1, SMART on FHIR 2.0, Bulk FHIR $export, Apache Iceberg, ONC compliance
- **Authentication:** AWS IAM, SMART on FHIR 2.0, OAuth 2.0 / OpenID Connect

---

### Google Cloud Healthcare API

- **Description:** Managed FHIR (R4, STU3, DSTU2), HL7v2, and DICOM data service on Google Cloud. Provides FHIR REST APIs, de-identification, and integration with BigQuery for population-scale analytics. HIPAA-compliant with BAA.
- **API Documentation:** https://cloud.google.com/healthcare-api/docs
- **FHIR Concepts:** https://cloud.google.com/healthcare-api/docs/concepts/fhir
- **SDKs:** Google Cloud client libraries for Python, Java, Go, Node.js, Ruby
- **Standards:** FHIR R4/STU3/DSTU2, HL7v2, DICOM, REST/JSON
- **Authentication:** Google IAM, OAuth 2.0, SMART on FHIR

---

### Azure Health Data Services

- **Description:** Microsoft's unified healthcare data platform combining FHIR, DICOM, and MedTech (IoT device) services. Deep integration with Microsoft Fabric (lakehouse analytics) and Dynamics 365 for patient engagement. HIPAA BAA available.
- **API Documentation:** https://learn.microsoft.com/en-us/azure/healthcare-apis/
- **FHIR Service Docs:** https://learn.microsoft.com/en-us/azure/healthcare-apis/fhir/
- **SDKs:** Azure SDK for Python, .NET, Java, JavaScript
- **Standards:** FHIR R4, HL7v2, DICOM, REST/JSON, Microsoft Fabric integration
- **Authentication:** Azure Active Directory (Entra ID), OAuth 2.0, SMART on FHIR

---

### HAPI FHIR Server (Open Source)

- **Description:** The leading open-source FHIR server implementation (Apache 2.0 licence), used as the reference implementation for HL7 FHIR. Can be self-hosted to build an open-source FHIR data store for a healthcare analytics platform without vendor lock-in.
- **API Documentation:** https://hapifhir.io/hapi-fhir/docs/
- **GitHub:** https://github.com/hapifhir/hapi-fhir
- **SDKs:** Java client library; REST-compatible with any HTTP client
- **Standards:** FHIR R4, STU3, DSTU2; US Core; Bulk FHIR; SMART on FHIR
- **Authentication:** Pluggable; supports OAuth 2.0, SMART on FHIR via integration

---

### NCQA HEDIS Digital Quality Measures (dQM)

- **Description:** NCQA's FHIR-CQL based digital HEDIS measure specifications, replacing the traditional hybrid measure methodology. Includes 22 published FHIR-CQL HEDIS measures with a roadmap to full ECDS transition by MY 2029.
- **Digital Measures Overview:** https://www.ncqa.org/hedis/the-future-of-hedis/digital-measures/
- **CQL Resources:** https://www.ncqa.org/resources/clinical-quality-language-and-cql-engines-the-basics/
- **Bulk FHIR Quality Coalition:** NCQA pilot programme for §170.315(g)(10) Bulk FHIR with HEDIS dQM use cases
- **Standards:** FHIR R4, CQL v1.5, QI-Core, ECDS

---

### AHRQ Quality Indicators Software

- **Description:** Government-maintained, free reference software for calculating standardised inpatient and outpatient quality indicators using administrative claims data. Provides validated calculation logic in SAS; an open-source analytics platform can port this logic to CQL for FHIR-native execution.
- **Download and Documentation:** https://www.ahrq.gov/data/qualityindicators/software.html
- **Overview:** https://www.ahrq.gov/data/qualityindicators/index.html
- **Licence:** Public domain (US government work)
- **Input Format:** Standardised administrative claims (UB-04 / CMS-1500 format)

---

## Notes

- **HEDIS annual update cycle:** NCQA publishes new HEDIS specifications each year (typically November–December for the following measurement year). Analytics platforms must build a structured measure update pipeline to maintain certification and compliance.
- **CQL engine options:** Open-source CQL execution engines include the cqframework reference engine (Java, Apache 2.0) at https://github.com/cqframework/clinical_quality_language, which can be embedded in analytics platforms for FHIR-native measure calculation without licensing a commercial measure engine.
- **SMART 2.0 vs SMART 1.0:** SMART on FHIR 2.0 (released 2023) introduces granular consent scopes and patient-level access control; platforms should target SMART 2.0 for new development, though Epic and other EHRs may still require SMART 1.0 compatibility.
- **Emerging standard — USCDI+:** ONC is developing USCDI+ (US Core Data for Interoperability Plus) programme-specific datasets (e.g., USCDI+ Quality); analytics platforms should monitor its adoption as it may expand mandatory data elements for quality reporting beyond the base US Core set.
