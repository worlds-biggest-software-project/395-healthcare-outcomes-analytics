# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Healthcare Outcomes Analytics · Created: 2026-05-29

## Philosophy

This model gives every healthcare analytics concept its own table with explicit foreign keys and referential integrity. The data flows through a clear pipeline: FHIR R4 bulk data ingestion populates patients, encounters, conditions, observations, medications, and procedures; a Master Patient Index links identities across source systems; quality measure calculations (HEDIS, CMS Stars, MIPS) produce measure results per patient per measure per period; risk stratification generates HCC codes and risk scores; and care gaps are identified as actionable records assigned to care coordinators.

The schema aligns with HL7 QI-Core FHIR profiles for quality measure calculation and OMOP CDM v5.4 concepts for observational research queries. Clinical data is stored in normalised tables that mirror FHIR resource types, enabling CQL-based measure logic to query standard relational columns rather than parsing raw FHIR JSON. HIPAA audit logging and role-based access control are built into the schema.

**Best for:** Health systems requiring auditable quality measure calculations with full data lineage from FHIR source to measure result, HIPAA-compliant access control at the patient and data-element level, and explicit referential integrity between clinical observations, diagnoses, and quality outcomes.

**Trade-offs:**
- (+) Full referential integrity from patient → encounter → condition → observation → measure result
- (+) Quality measure calculation queries standard SQL columns mapped to QI-Core profiles
- (+) Clear data lineage from FHIR ingestion to care gap for regulatory audit
- (+) HIPAA audit log with explicit resource-level access tracking
- (-) Higher table count increases migration complexity as FHIR profiles evolve
- (-) FHIR-to-relational mapping requires ongoing maintenance as US Core profiles update
- (-) Clinical NLP extractions need their own table to avoid widening existing clinical tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | Clinical tables map to FHIR resource types (Patient, Encounter, Condition, Observation, etc.) |
| HL7 QI-Core | Quality measure queries reference QI-Core-profiled columns |
| CQL v1.5.3 | Measure logic executes against normalised clinical tables |
| HL7 Da Vinci DEQM | Measure report submission uses DEQM exchange patterns |
| US Core IG | Patient, condition, and observation fields follow US Core required elements |
| OMOP CDM v5.4 | Concept IDs for conditions, procedures, and medications align with OMOP vocabulary |
| CMS HCC Risk Adjustment | HCC category mapping and risk score calculation |
| NCQA HEDIS MY 2026 | Measure definitions implemented as CQL against QI-Core tables |
| AHRQ QI | IQI, PSI, PQI, PDI measure logic ported from SAS reference implementations |
| HIPAA Security Rule | Audit logging, access control, encryption at rest |
| SMART on FHIR v2.2 | App launch context stored for in-EHR integration |

---

## Entity Management

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    organisation_type TEXT NOT NULL CHECK (organisation_type IN (
        'health_system', 'aco', 'payer', 'community_health_center', 'physician_group'
    )),
    npi             TEXT,  -- National Provider Identifier
    tin             TEXT,  -- Tax Identification Number
    plan            TEXT NOT NULL CHECK (plan IN ('community', 'professional', 'enterprise')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           TEXT NOT NULL,
    name            TEXT NOT NULL,
    role            TEXT NOT NULL CHECK (role IN (
        'admin', 'quality_director', 'care_manager', 'physician',
        'analyst', 'viewer', 'api_service'
    )),
    npi             TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);

CREATE TABLE data_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    source_type     TEXT NOT NULL CHECK (source_type IN (
        'epic_fhir', 'oracle_health_fhir', 'claims_837', 'claims_cms',
        'lab_hl7v2', 'pharmacy', 'sdoh', 'pro', 'adt_hl7v2',
        'healthlake', 'gcp_healthcare', 'azure_health', 'hapi_fhir',
        'csv_import', 'custom_fhir'
    )),
    fhir_base_url   TEXT,
    auth_method     TEXT CHECK (auth_method IN ('smart_on_fhir', 'oauth2_client_credentials', 'bearer_token', 'api_key')),
    credentials_ref TEXT,
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'connected', 'syncing', 'active', 'error', 'disconnected'
    )),
    last_sync_at    TIMESTAMPTZ,
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_data_sources_tenant ON data_sources(tenant_id);
```

## Master Patient Index

```sql
CREATE TABLE patients (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    mpi_id          UUID NOT NULL,  -- Master Patient Index identifier
    mrn             TEXT,  -- Medical Record Number (per source)
    data_source_id  UUID NOT NULL REFERENCES data_sources(id),
    fhir_id         TEXT,  -- FHIR Patient resource ID
    name_family     TEXT NOT NULL,
    name_given      TEXT,
    date_of_birth   DATE NOT NULL,
    gender          TEXT CHECK (gender IN ('male', 'female', 'other', 'unknown')),
    race            TEXT,
    ethnicity       TEXT,
    language        TEXT,
    address_line    TEXT,
    address_city    TEXT,
    address_state   TEXT,
    address_zip     TEXT,
    address_country CHAR(2) DEFAULT 'US',
    phone           TEXT,
    email           TEXT,
    deceased        BOOLEAN NOT NULL DEFAULT false,
    deceased_date   DATE,
    payer_id        TEXT,
    payer_name      TEXT,
    attribution_provider_npi TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, data_source_id, fhir_id)
);

CREATE INDEX idx_patients_tenant ON patients(tenant_id);
CREATE INDEX idx_patients_mpi ON patients(mpi_id);
CREATE INDEX idx_patients_dob ON patients(tenant_id, date_of_birth);
CREATE INDEX idx_patients_zip ON patients(tenant_id, address_zip);

CREATE TABLE patient_identities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    mpi_id          UUID NOT NULL,
    data_source_id  UUID NOT NULL REFERENCES data_sources(id),
    patient_id      UUID NOT NULL REFERENCES patients(id),
    identifier_system TEXT NOT NULL,
    identifier_value TEXT NOT NULL,
    match_confidence NUMERIC(5,4),
    match_method    TEXT CHECK (match_method IN ('deterministic', 'probabilistic', 'manual', 'llm_embedding')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, data_source_id, identifier_system, identifier_value)
);

CREATE INDEX idx_patient_identities_mpi ON patient_identities(mpi_id);
```

## Clinical Data

```sql
CREATE TABLE encounters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    patient_id      UUID NOT NULL REFERENCES patients(id),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id),
    fhir_id         TEXT,
    encounter_class TEXT NOT NULL CHECK (encounter_class IN (
        'ambulatory', 'emergency', 'inpatient', 'observation', 'virtual', 'home_health'
    )),
    status          TEXT NOT NULL CHECK (status IN (
        'planned', 'arrived', 'in_progress', 'finished', 'cancelled'
    )),
    period_start    TIMESTAMPTZ NOT NULL,
    period_end      TIMESTAMPTZ,
    facility_npi    TEXT,
    facility_name   TEXT,
    provider_npi    TEXT,
    provider_name   TEXT,
    discharge_disposition TEXT,
    primary_diagnosis_code TEXT,
    drg_code        TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (period_start);

CREATE INDEX idx_encounters_patient ON encounters(patient_id);
CREATE INDEX idx_encounters_tenant ON encounters(tenant_id, period_start);
CREATE INDEX idx_encounters_class ON encounters(tenant_id, encounter_class);

CREATE TABLE conditions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    patient_id      UUID NOT NULL REFERENCES patients(id),
    encounter_id    UUID REFERENCES encounters(id),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id),
    fhir_id         TEXT,
    code_system     TEXT NOT NULL,  -- ICD-10-CM, SNOMED CT
    code            TEXT NOT NULL,
    display         TEXT,
    category        TEXT CHECK (category IN ('encounter_diagnosis', 'problem_list', 'health_concern')),
    clinical_status TEXT CHECK (clinical_status IN ('active', 'recurrence', 'relapse', 'inactive', 'remission', 'resolved')),
    onset_date      DATE,
    abatement_date  DATE,
    recorded_date   DATE,
    hcc_category    TEXT,  -- CMS HCC mapping
    hcc_model_version TEXT,
    is_nlp_extracted BOOLEAN NOT NULL DEFAULT false,
    nlp_confidence  NUMERIC(5,4),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_conditions_patient ON conditions(patient_id);
CREATE INDEX idx_conditions_code ON conditions(tenant_id, code_system, code);
CREATE INDEX idx_conditions_hcc ON conditions(tenant_id, hcc_category) WHERE hcc_category IS NOT NULL;

CREATE TABLE observations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    patient_id      UUID NOT NULL REFERENCES patients(id),
    encounter_id    UUID REFERENCES encounters(id),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id),
    fhir_id         TEXT,
    code_system     TEXT NOT NULL,  -- LOINC, SNOMED CT
    code            TEXT NOT NULL,
    display         TEXT,
    category        TEXT CHECK (category IN (
        'vital_signs', 'laboratory', 'social_history', 'survey', 'imaging', 'sdoh'
    )),
    value_quantity  NUMERIC(12,4),
    value_unit      TEXT,
    value_string    TEXT,
    value_code      TEXT,
    effective_date  TIMESTAMPTZ NOT NULL,
    status          TEXT NOT NULL DEFAULT 'final',
    is_nlp_extracted BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (effective_date);

CREATE INDEX idx_observations_patient ON observations(patient_id);
CREATE INDEX idx_observations_code ON observations(tenant_id, code_system, code);
CREATE INDEX idx_observations_category ON observations(tenant_id, category);

CREATE TABLE procedures (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    patient_id      UUID NOT NULL REFERENCES patients(id),
    encounter_id    UUID REFERENCES encounters(id),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id),
    fhir_id         TEXT,
    code_system     TEXT NOT NULL,  -- CPT, HCPCS, ICD-10-PCS, SNOMED CT
    code            TEXT NOT NULL,
    display         TEXT,
    performed_date  TIMESTAMPTZ NOT NULL,
    status          TEXT NOT NULL DEFAULT 'completed',
    performer_npi   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_procedures_patient ON procedures(patient_id);
CREATE INDEX idx_procedures_code ON procedures(tenant_id, code_system, code);

CREATE TABLE medications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    patient_id      UUID NOT NULL REFERENCES patients(id),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id),
    fhir_id         TEXT,
    code_system     TEXT NOT NULL,  -- RxNorm, NDC
    code            TEXT NOT NULL,
    display         TEXT,
    status          TEXT NOT NULL CHECK (status IN ('active', 'completed', 'stopped', 'entered_in_error')),
    medication_type TEXT CHECK (medication_type IN ('request', 'statement', 'administration', 'dispense')),
    start_date      DATE,
    end_date        DATE,
    dosage          TEXT,
    prescriber_npi  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_medications_patient ON medications(patient_id);
CREATE INDEX idx_medications_code ON medications(tenant_id, code_system, code);
```

## Quality Measures & Care Gaps

```sql
CREATE TABLE measure_definitions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    measure_id      TEXT NOT NULL,  -- e.g., CMS122v12, NQF0059
    measure_set     TEXT NOT NULL CHECK (measure_set IN (
        'hedis', 'cms_ecqm', 'mips', 'ahrq_iqi', 'ahrq_psi', 'ahrq_pqi', 'ahrq_pdi', 'custom'
    )),
    title           TEXT NOT NULL,
    description     TEXT,
    measurement_year INT NOT NULL,
    version         TEXT NOT NULL,
    cql_library     TEXT,  -- reference to CQL library name
    numerator_description TEXT,
    denominator_description TEXT,
    exclusion_description TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, measure_id, measurement_year)
);

CREATE INDEX idx_measures_tenant ON measure_definitions(tenant_id);
CREATE INDEX idx_measures_set ON measure_definitions(tenant_id, measure_set);

CREATE TABLE measure_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    measure_id      UUID NOT NULL REFERENCES measure_definitions(id),
    patient_id      UUID NOT NULL REFERENCES patients(id),
    measurement_period_start DATE NOT NULL,
    measurement_period_end   DATE NOT NULL,
    in_initial_population BOOLEAN NOT NULL DEFAULT false,
    in_denominator  BOOLEAN NOT NULL DEFAULT false,
    in_denominator_exclusion BOOLEAN NOT NULL DEFAULT false,
    in_numerator    BOOLEAN NOT NULL DEFAULT false,
    in_numerator_exclusion BOOLEAN NOT NULL DEFAULT false,
    in_exception    BOOLEAN NOT NULL DEFAULT false,
    performance_met BOOLEAN,
    evidence        JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "qualifying_encounters": ["uuid_1", "uuid_2"],
    --   "qualifying_conditions": ["uuid_3"],
    --   "qualifying_observations": ["uuid_4"],
    --   "cql_evaluation_log": "..."
    -- }
    calculated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    cql_engine_version TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, measure_id, patient_id, measurement_period_start)
);

CREATE INDEX idx_measure_results_patient ON measure_results(patient_id);
CREATE INDEX idx_measure_results_measure ON measure_results(measure_id);
CREATE INDEX idx_measure_results_performance ON measure_results(tenant_id, measure_id) WHERE NOT performance_met AND in_denominator AND NOT in_denominator_exclusion;

CREATE TABLE care_gaps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    patient_id      UUID NOT NULL REFERENCES patients(id),
    measure_id      UUID NOT NULL REFERENCES measure_definitions(id),
    measure_result_id UUID REFERENCES measure_results(id),
    gap_type        TEXT NOT NULL CHECK (gap_type IN ('open', 'predicted', 'coding')),
    priority        TEXT NOT NULL CHECK (priority IN ('low', 'medium', 'high', 'critical')),
    status          TEXT NOT NULL DEFAULT 'open' CHECK (status IN (
        'open', 'assigned', 'outreach_attempted', 'scheduled', 'closed_met', 'closed_excluded', 'closed_expired'
    )),
    assigned_to     UUID REFERENCES users(id),
    due_date        DATE,
    description     TEXT NOT NULL,
    recommended_action TEXT,
    closed_at       TIMESTAMPTZ,
    closure_evidence JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_care_gaps_patient ON care_gaps(patient_id);
CREATE INDEX idx_care_gaps_measure ON care_gaps(measure_id);
CREATE INDEX idx_care_gaps_status ON care_gaps(tenant_id, status);
CREATE INDEX idx_care_gaps_assigned ON care_gaps(assigned_to) WHERE status IN ('assigned', 'outreach_attempted');
CREATE INDEX idx_care_gaps_priority ON care_gaps(tenant_id, priority) WHERE status = 'open';
```

## Risk Stratification & AI

```sql
CREATE TABLE risk_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    patient_id      UUID NOT NULL REFERENCES patients(id),
    score_type      TEXT NOT NULL CHECK (score_type IN (
        'hcc_raf', 'readmission_30d', 'ed_utilisation', 'chronic_condition',
        'total_cost_of_care', 'custom'
    )),
    score_date      DATE NOT NULL,
    score_value     NUMERIC(10,4) NOT NULL,
    risk_tier       TEXT CHECK (risk_tier IN ('low', 'rising', 'moderate', 'high', 'very_high')),
    model_version   TEXT NOT NULL,
    hcc_categories  TEXT[],  -- for HCC RAF scores
    contributing_conditions JSONB,
    feature_contributions JSONB,
    confidence      NUMERIC(5,4),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, patient_id, score_type, score_date)
);

CREATE INDEX idx_risk_scores_patient ON risk_scores(patient_id);
CREATE INDEX idx_risk_scores_tenant ON risk_scores(tenant_id, score_type, score_date);
CREATE INDEX idx_risk_scores_tier ON risk_scores(tenant_id, risk_tier);

CREATE TABLE ai_suggestions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    suggestion_type TEXT NOT NULL CHECK (suggestion_type IN (
        'care_gap_predicted', 'hcc_coding_gap', 'readmission_risk',
        'measure_anomaly', 'data_completeness', 'nlp_extraction',
        'care_plan_narrative', 'provider_benchmark',
        'sdoh_risk_factor', 'medication_adherence'
    )),
    entity_type     TEXT,
    entity_id       UUID,
    patient_id      UUID REFERENCES patients(id),
    severity        TEXT NOT NULL CHECK (severity IN ('info', 'warning', 'critical')),
    title           TEXT NOT NULL,
    description     TEXT NOT NULL,
    evidence        JSONB NOT NULL DEFAULT '{}',
    recommended_action TEXT,
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'accepted', 'dismissed', 'expired', 'auto_applied'
    )),
    confidence      NUMERIC(5,4),
    model_version   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_ai_suggestions_tenant ON ai_suggestions(tenant_id);
CREATE INDEX idx_ai_suggestions_type ON ai_suggestions(tenant_id, suggestion_type);
CREATE INDEX idx_ai_suggestions_patient ON ai_suggestions(patient_id);
CREATE INDEX idx_ai_suggestions_pending ON ai_suggestions(tenant_id) WHERE status = 'pending';

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    actor_type      TEXT NOT NULL CHECK (actor_type IN ('user', 'system', 'api_key', 'smart_app', 'ai', 'scheduler', 'fhir_client')),
    actor_id        TEXT,
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID,
    patient_id      UUID,
    phi_accessed    BOOLEAN NOT NULL DEFAULT false,
    changes         JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at);
CREATE INDEX idx_audit_log_patient ON audit_log(patient_id);
CREATE INDEX idx_audit_log_phi ON audit_log(tenant_id) WHERE phi_accessed;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Entity Management | 3 | tenants, users, data_sources |
| Master Patient Index | 2 | patients, patient_identities |
| Clinical Data | 4 | encounters (partitioned), conditions, observations (partitioned), procedures, medications |
| Quality Measures & Care Gaps | 3 | measure_definitions, measure_results, care_gaps |
| Risk & AI & Audit | 3 | risk_scores, ai_suggestions, audit_log (partitioned) |
| **Total** | **16** | 3 partitioned tables (plus medications = 16 total) |

---

## Key Design Decisions

1. **Clinical tables map to FHIR resource types** — encounters, conditions, observations, procedures, and medications mirror FHIR Patient, Encounter, Condition, Observation, Procedure, and MedicationRequest/MedicationStatement. CQL measure logic queries these tables directly rather than parsing raw FHIR JSON.

2. **Master Patient Index with probabilistic matching** — patient_identities stores cross-source identity links with match_confidence and match_method (including llm_embedding for AI-enhanced matching). mpi_id groups patients from different sources.

3. **HCC coding on conditions** — conditions.hcc_category and hcc_model_version store the CMS HCC mapping directly on diagnosis records, enabling coding gap identification without a separate lookup table.

4. **NLP extraction flagging** — conditions and observations carry is_nlp_extracted and nlp_confidence flags, distinguishing structured EHR data from clinical NLP pipeline outputs. This supports audit and clinician trust in AI-extracted data.

5. **Measure results with full HEDIS/eCQM population logic** — measure_results stores the complete quality measure population membership (initial population, denominator, numerator, exclusions, exceptions) per patient per measure per period, following the QI-Core/DEQM pattern.

6. **Care gaps as actionable workflow records** — care_gaps bridges quality measurement to clinical action. Gap types include "open" (measure not met), "predicted" (AI predicts gap will open), and "coding" (HCC diagnosis not captured). Assigned care coordinators track outreach status.

7. **HIPAA audit log with PHI flag** — audit_log.phi_accessed explicitly marks when protected health information was accessed, supporting HIPAA minimum necessary analysis and breach investigation.

8. **SDOH observations** — observations.category includes 'sdoh' for social determinants of health data, enabling SDOH-informed risk stratification alongside clinical observations.

9. **Multi-code-system support** — conditions, observations, procedures, and medications each carry code_system and code columns supporting ICD-10, SNOMED CT, LOINC, CPT, HCPCS, RxNorm, and NDC without code-system-specific tables.

10. **Evidence linking on measure results** — measure_results.evidence JSONB captures the specific encounters, conditions, and observations that contributed to the measure evaluation, enabling click-through audit from measure result to clinical data.
