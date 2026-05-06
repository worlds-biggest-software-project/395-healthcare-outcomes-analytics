# Healthcare Outcomes Analytics — Feature & Functionality Survey

> Candidate #395 · Researched: 2026-05-06

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Innovaccer Data Activation Platform (DAP) | Commercial SaaS | Proprietary / subscription | https://innovaccer.com |
| Arcadia Data Platform | Commercial SaaS | Proprietary / subscription | https://arcadia.io |
| Health Catalyst DOS | Commercial SaaS | Proprietary / subscription | https://healthcatalyst.com |
| Persivia CareSpace | Commercial SaaS | Proprietary / subscription | https://persivia.com |
| Oracle Health Data Intelligence (Cerner) | Commercial SaaS | Proprietary / subscription | https://docs.healtheintent.com |
| AWS HealthLake | Managed Cloud Service | Commercial / pay-per-use | https://aws.amazon.com/healthlake |
| Google Cloud Healthcare API | Managed Cloud Service | Commercial / pay-per-use | https://cloud.google.com/healthcare-api |
| Azure Health Data Services | Managed Cloud Service | Commercial / pay-per-use | https://azure.microsoft.com/products/health-data-services |
| AHRQ Quality Indicators (QI) Software | Open Source / Government | Free (public domain) | https://www.ahrq.gov/data/qualityindicators |
| HAPI FHIR Server | Open Source | Apache 2.0 | https://hapifhir.io |

---

## Feature Analysis by Solution

### Innovaccer Data Activation Platform (DAP)

**Core features**
- Multi-source data ingestion with 65+ pre-built EHR connectors and 200+ IT vendor connectors
- Unified patient longitudinal record using a FHIR+ data model at Level 4 interoperability
- Master Patient Index (MPI) for probabilistic patient identity matching across sources
- HEDIS score calculation and 400+ quality measure tracking across federal and state programmes
- AI-powered risk stratification to flag high-risk patients and care gaps in near real time
- Care coordination workflow tools including task assignment and care manager outreach queues
- Provider and executive dashboards with customisable analytics views
- Function-as-a-Service (FaaS) for custom AI/ML algorithm deployment

**Differentiating features**
- FHIR+ graph data model optimised for high-performance population-level queries
- End-to-end analytics from data ingestion to point-of-care workflow integration
- Network-level benchmarking across payer-provider relationships

**UX patterns**
- Role-based dashboards for clinical, financial, and operational personas
- Embedded SMART on FHIR app that surfaces insights directly inside EHR workflows
- Care gap prioritisation lists auto-assigned to care coordinators

**Integration points**
- SMART on FHIR app launch for EHR embedding
- OAuth 2.0 and SMART on FHIR authorization
- REST APIs for third-party application access
- Azure Marketplace and AWS Marketplace listings
- Direct Epic and Oracle Health EHR integrations

**Known gaps**
- Detailed public developer/API documentation is limited; requires vendor engagement
- Custom algorithm deployment (FaaS) requires significant technical expertise
- Pricing is enterprise-only and not publicly disclosed

**Licence / IP notes**
- Fully proprietary; no open-source components disclosed
- HIPAA-compliant BAA available; SOC 2 Type II certified

---

### Arcadia Data Platform

**Core features**
- Clinical, claims, SDOH, pharmacy, and lab data aggregation into a unified data lakehouse
- Risk stratification and population segmentation for value-based care contracts
- Quality measure performance tracking (HEDIS, CMS Stars, MIPS)
- Care gap identification and automated outreach workflows
- Contract performance modelling for ACO, MSSP, and commercial risk programmes
- Benchmark reporting for provider network comparisons
- AI-driven coding gap identification for risk adjustment (HCC coding)
- BCDA (Beneficiary Claims Data API) integration via FHIR adapters with AWS HealthLake

**Differentiating features**
- Deep specialisation in value-based care contract performance and payer-provider relationships
- Combined clinical and financial data view in a single report
- Point-of-care insights delivered as a SMART on FHIR application inside EHRs
- Strong interoperability with AWS HealthLake for CMS claims data

**UX patterns**
- Signal real-time data dashboards for operations teams
- Role-stratified views: ACO leadership, care managers, network administrators
- Configurable benchmarking and ranking of providers within a network

**Integration points**
- SMART on FHIR Point of Care Insights application
- REST API (Plug API) with reference documentation at docs.arcadia.com
- Signal API for real-time data streams
- AWS HealthLake integration for FHIR bulk data
- EHR integrations with Epic and Oracle Health

**Known gaps**
- KLAS score (78.7) slightly below Innovaccer, with user feedback citing implementation complexity
- Public developer documentation (docs.arcadia.com) exists but depth is limited compared to cloud-native providers
- Smaller partner ecosystem than Health Catalyst

**Licence / IP notes**
- Fully proprietary
- HIPAA BAA available; HITRUST certified

---

### Health Catalyst Data Operating System (DOS)

**Core features**
- Enterprise data lake aggregating 40 trillion facts from 300+ source systems across 125 million patients
- Pre-built analytical applications covering clinical quality, finance, and operations
- ONC-certified electronic clinical quality measure (eCQM) calculation and submission
- AI predictive models: readmission risk, sepsis prediction, surgical quality benchmarking
- Cohort definition workspaces with configurable logic using OMOP and custom value sets
- Cost accounting and contract performance management
- Care variation analysis across providers and service lines
- Patient safety identification and reporting (e.g., harm event detection)

**Differentiating features**
- Breadth of coverage across an entire integrated delivery network, not just population health
- ONC certification for eCQM regulatory submission — validated for accuracy
- Cohort definition time reduced by 50%+ compared to manual approaches
- Library of pre-built analytics applications deployable across organisations

**UX patterns**
- Role-based dashboards for clinical, financial, and operations leaders
- Real-time tracking by population, provider, or contract
- Integrated intervention planning workflows linked to analytics insights

**Integration points**
- FHIR R4 APIs for data ingestion and exchange
- EHR integrations: Epic, Oracle Health, and 300+ other source systems
- Pre-built connectors for claims, lab, pharmacy, and SDOH data
- Custom application development via DOS SDK for analytics teams

**Known gaps**
- KLAS score (79.3) indicates moderate satisfaction relative to cost
- Platform complexity makes it best suited to large integrated delivery networks; smaller organisations report difficulty realising value
- Custom application development requires significant internal data engineering resources

**Licence / IP notes**
- Fully proprietary
- HIPAA BAA and HITRUST certification available

---

### Persivia CareSpace

**Core features**
- AI-driven population health analytics using the proprietary Soliton AI engine
- Data lakehouse architecture with semantic normalisation across clinical, claims, SDOH, device, and patient-reported data
- Risk stratification and expenditure prediction for all Attributed Population and Episodic Models
- Quality measure performance management and reporting (HEDIS, Stars, MIPS)
- Care management workflow support with automated task routing
- HCC coding gap identification for risk adjustment
- Real-time care gap identification and intervention opportunity flagging

**Differentiating features**
- Soliton AI Engine delivering predictive, prescriptive, and generative insights simultaneously
- Semantic normalisation layer that makes data engineering-ready without custom ETL
- Support for a broad range of risk models including both attributed and episodic payment models

**UX patterns**
- Configurable dashboards per role: clinician, care manager, finance, executive
- Automated care manager task assignment from AI risk scores
- Real-time alerting for intervention opportunities

**Integration points**
- Standards-based FHIR APIs for EHR integration
- Claims data ingestion from payer feeds
- SDOH data sources including community resource databases

**Known gaps**
- Less mature partner ecosystem and fewer pre-built connectors than Innovaccer or Health Catalyst
- Public API documentation is not disclosed; vendor engagement required
- Smaller market presence limits third-party validation data (fewer KLAS reviews)

**Licence / IP notes**
- Fully proprietary; Soliton AI Engine is a trademarked proprietary system
- HIPAA BAA available

---

### Oracle Health Data Intelligence (formerly Cerner HealtheAnalytics)

**Core features**
- Longitudinal patient record management with cohort and registry definition
- Cohort API for building patient subsets by clinical or demographic characteristics
- HealtheRegistries for programme enrolment, care gap tracking, and outcome measurement
- Observation, Condition, and Patient APIs for structured clinical data access
- FHIR R4 APIs for Oracle Health Millennium Platform (inpatient/outpatient EHR)
- Population health analytics integrated with Cerner EHR for in-workflow insights
- Risk stratification and HCC coding gap identification

**Differentiating features**
- Native integration with Oracle Health EHR (Cerner Millennium), the second-largest EHR in the US
- Developer sandbox (cernerdemo tenant) with realistic synthetic data for testing
- Population-level FHIR APIs enabling bulk data extraction for analytics pipelines

**UX patterns**
- EHR-embedded population health views for clinicians during patient encounters
- Registry-based care management workflows for care coordinators
- API-first design enables custom front-end development by health systems

**Integration points**
- FHIR R4 Millennium Platform APIs: https://docs.oracle.com/en/industries/health/millennium-platform-apis/
- Oracle Health Data Intelligence REST APIs: https://docs.healtheintent.com/
- Bearer token and OAuth 1.0a authentication options
- cernerdemo sandbox for developer testing

**Known gaps**
- Platform is tightly coupled to the Oracle Health (Cerner) EHR; interoperability with Epic-heavy networks requires additional integration work
- Analytics capabilities lag behind dedicated platforms like Health Catalyst and Innovaccer for cross-system population health
- Developer documentation exists but is less polished than Epic's FHIR developer programme

**Licence / IP notes**
- Fully proprietary
- HIPAA BAA available as part of Oracle Health agreements

---

### AWS HealthLake

**Core features**
- Fully managed HIPAA-eligible FHIR R4 data store at petabyte scale
- Automatic transformation of FHIR data into Apache Iceberg analytics-ready format
- SQL-on-FHIR for population health and quality measure analytics without custom pipelines
- Built-in de-identification and NLP via Amazon Comprehend Medical
- SMART on FHIR 2.0 authorization and OAuth 2.0/OpenID Connect support
- HealthLake Analytics for dashboard building, ML model training, and deployment
- ONC and CMS interoperability rule compliance

**Differentiating features**
- Eliminates custom ETL by automatically converting FHIR to analytics-ready Apache Iceberg tables
- Native integration with AWS ML ecosystem (SageMaker, Comprehend Medical, Athena)
- SMART 2.0 support for granular, patient-level consent and access control

**UX patterns**
- Developer-first: API-centric with no built-in clinical UI
- Integration with AWS QuickSight for dashboard creation
- Designed to be consumed by analytics or application layers built on top

**Integration points**
- FHIR R4 REST API with bulk $export
- SMART on FHIR 2.0 launch framework
- AWS Lake Formation and Athena for SQL analytics
- Amazon SageMaker for ML model training on clinical data
- Official documentation: https://aws.amazon.com/healthlake/

**Known gaps**
- No built-in clinical workflow or care management UI; requires additional application development
- Cost scales with data volume; large deployments can be expensive
- US-only HIPAA coverage; international deployments require separate compliance review

**Licence / IP notes**
- Commercial managed service; no source code disclosed
- HIPAA BAA available under standard AWS Business Associate Agreement

---

### AHRQ Quality Indicators (QI) Software

**Core features**
- Standardised measure sets for inpatient quality (IQI), patient safety (PSI), prevention quality (PQI), and paediatric quality (PDI)
- Reference SAS and WINQL software implementations for measure calculation
- Risk adjustment methodologies for fair provider comparisons
- Publicly validated against CMS claims data for regulatory use

**Differentiating features**
- Government-maintained and freely available; accepted as regulatory benchmarks
- Provides reference implementations that commercial platforms use to validate their own calculations
- No licensing cost for use in analytics platforms

**UX patterns**
- Command-line/batch processing tools designed for analysts, not clinicians
- Outputs structured tabular results for downstream visualisation in other tools

**Integration points**
- Input accepts standardised administrative claims format
- Output compatible with common analytical tools (SAS, R, Tableau)
- Official reference: https://www.ahrq.gov/data/qualityindicators/index.html

**Known gaps**
- No real-time or FHIR-native capability; designed for retrospective batch analysis
- Requires significant technical expertise to deploy and maintain
- No care management workflow integration

**Licence / IP notes**
- Public domain (US government work); free to use, modify, and distribute

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Multi-source data ingestion: EHR, claims, pharmacy, lab, and SDOH data into a unified patient record
- Master Patient Index / identity matching for deduplication across source systems
- HEDIS, CMS Stars, and MIPS quality measure calculation and automated reporting
- Risk stratification with configurable risk models for chronic condition and utilisation prediction
- Care gap identification and prioritised outreach lists for care coordinators
- Role-based dashboards for clinical, financial, and operational users
- HIPAA-compliant data storage with BAA, audit logging, and access controls
- FHIR R4 APIs for data ingestion, interoperability, and downstream application access

### Differentiating Features
- AI-generated prescriptive recommendations (not just descriptive reporting) for care managers
- Point-of-care insights embedded directly in EHR workflows via SMART on FHIR
- Network-level benchmarking of providers across a payer-provider network
- Support for episodic and attributed payment model analytics (ACO, MSSP, BPCI)
- ONC-certified eCQM regulatory submission capability
- Real-time (not batch-only) population risk updates
- HCC coding gap identification for risk adjustment revenue optimisation

### Underserved Areas / Opportunities
- Transparent, auditable AI/ML models that clinicians can interrogate and trust
- Patient-reported outcomes (PRO) integration as a first-class data source alongside EHR and claims
- Natural language generation to surface actionable summaries from complex population data — not just charts
- Open-source, self-hostable quality measure engines that organisations can run without vendor lock-in
- Interoperability across heterogeneous EHR environments without needing 65+ proprietary connectors
- Social determinants of health (SDOH) data as a primary risk signal, not an afterthought
- Lightweight deployment for smaller provider organisations and community health centres that cannot afford enterprise SaaS pricing

### AI-Augmentation Candidates
- Clinical NLP to extract quality-measure-relevant data from unstructured physician notes, discharge summaries, and operative reports
- Predictive care gap identification: flagging patients likely to miss upcoming screenings before the gap opens
- Generative AI for care plan narratives: converting risk scores into plain-language guidance for care managers
- Automated HCC coding suggestion from clinical notes using LLMs trained on medical coding ontologies
- Intelligent patient matching: using LLM embeddings to improve probabilistic identity resolution across data sources
- Anomaly detection for quality measure calculation errors and data completeness issues

---

## Legal & IP Summary

All major commercial platforms (Innovaccer, Arcadia, Health Catalyst, Persivia, Oracle Health) are fully proprietary with no open-source components disclosed. Their APIs are publicly documented to varying degrees, but the core data models, connectors, and AI engines are trade secrets. AHRQ QI software is US government public domain and can be used freely. HAPI FHIR (Apache 2.0) and the HL7 QI-Core Implementation Guide (Creative Commons) are open standards with permissive licensing. Clinical Quality Language (CQL) is an HL7 specification published under the HL7 IP policy, which permits implementation. No specific patent concerns were identified in the research, though individual AI/ML model architectures at commercial vendors may be patented. An AI-native open-source platform should build on FHIR R4, CQL, QI-Core, and AHRQ QI reference implementations to avoid IP risk.

---

## Recommended Feature Scope

**Must-have (MVP)**
- FHIR R4 bulk data ingestion from Epic, Oracle Health, and claims sources
- Patient identity matching (probabilistic MPI) across ingested sources
- HEDIS and CMS eCQM quality measure calculation using CQL
- Population risk stratification with configurable chronic-condition risk models
- Care gap identification dashboard with prioritised outreach lists
- Role-based access control with HIPAA audit logging

**Should-have (v1.1)**
- Clinical NLP pipeline (spaCy/scispaCy or BioMedBERT) for unstructured note extraction
- SMART on FHIR application for in-EHR point-of-care insights
- HCC coding gap identification and risk adjustment analytics
- SDOH data integration as a risk stratification signal
- Provider network benchmarking across quality and utilisation metrics
- eCQM regulatory submission (ONC-certified calculation path)

**Nice-to-have (backlog)**
- Generative AI care plan narrative synthesis for care managers
- Episodic payment model analytics (BPCI, bundled payments)
- Patient-reported outcomes (PRO) as a first-class data source
- Real-time alerting for acute risk changes (e.g., post-discharge readmission risk)
- Open plugin marketplace for custom quality measure logic contributed by the community
