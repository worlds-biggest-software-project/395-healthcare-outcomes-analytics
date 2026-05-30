# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Healthcare Outcomes Analytics · Created: 2026-05-29

## Philosophy

This model keeps patients, encounters, and measure results as relational tables (for quality measure calculation and population health queries) while embedding clinical observations, conditions, procedures, medications, and risk scores into JSONB columns on the patient record. The patient becomes a longitudinal health document containing all clinical facts, care gaps, risk scores, and quality measure outcomes. Quality measure calculation queries the relational encounter and measure tables, while the care coordinator drill-down view fetches a single patient row.

Healthcare analytics has two access patterns: population-level queries ("how many diabetic patients had an A1c in the measurement period?") that benefit from relational indexing on encounter dates, diagnosis codes, and measure membership; and patient-level queries ("show me everything about this patient") that benefit from a single-document model. The hybrid approach serves both: measure calculations run against relational encounter and measure tables, while care gap assignment and care plan review operate on the patient document.

**Best for:** Rapid MVP deployment where the team wants to iterate on clinical data elements (new SDOH fields, new FHIR extensions) without schema migrations, while maintaining relational performance for population-level quality measure queries.

**Trade-offs:**
- (+) Patient is a complete longitudinal health document — single fetch for care coordinator view
- (+) New clinical data elements (SDOH, PROs) added as JSONB fields without ALTER TABLE
- (+) Quality measure calculations on relational encounter and measure tables
- (+) FHIR resource extensions stored naturally in JSONB without schema widening
- (-) Population queries on embedded conditions/observations require JSONB operators or GIN indexes
- (-) Referential integrity between embedded clinical data and encounter IDs validated in application code
- (-) Patient rows can grow large for patients with extensive clinical histories

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | Patient JSONB embeds FHIR resource representations |
| HL7 QI-Core | Measure calculations query relational encounters and measure results |
| CQL v1.5.3 | Measure logic adapted to hybrid query pattern |
| US Core IG | Patient relational fields follow US Core required elements |
| CMS HCC | HCC categories and risk scores embedded in patient clinical data |
| NCQA HEDIS | Measure definitions and results tracked relationally |
| HIPAA | Audit logging with PHI access tracking |

---

## Core Tables

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    organisation_type TEXT NOT NULL CHECK (organisation_type IN (
        'health_system', 'aco', 'payer', 'community_health_center', 'physician_group'
    )),

    -- Embedded users
    users_json      JSONB NOT NULL DEFAULT '[]',

    -- Data source connections
    data_sources_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Epic Production",
    --   "source_type": "epic_fhir|oracle_health_fhir|claims_837|lab_hl7v2|...",
    --   "fhir_base_url": "https://fhir.epic.example.com/R4",
    --   "auth_method": "smart_on_fhir", "credentials_ref": "vault://...",
    --   "status": "active", "last_sync_at": "2026-05-29T..."
    -- }]

    -- Measure programme configuration
    programmes_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "MSSP ACO 2026",
    --   "programme_type": "mssp|stars|mips|hedis|custom",
    --   "measurement_year": 2026,
    --   "measure_ids": ["CMS122v12", "CMS165v12", ...],
    --   "reporting_deadline": "2027-03-31"
    -- }]

    -- Provider network
    providers_json  JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "npi": "1234567890", "name": "Dr. Jane Smith",
    --   "specialty": "Internal Medicine", "facility": "Main Clinic",
    --   "attributed_patients": 450
    -- }]

    -- Platform configuration
    config_json     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "plan": "professional",
    --   "risk_models": ["hcc_raf", "readmission_30d", "chronic_condition"],
    --   "nlp": {"enabled": true, "models": ["scispacy", "biomedbert"]},
    --   "smart_on_fhir": {"client_id": "...", "redirect_uri": "..."},
    --   "care_gap_assignment": {"auto_assign": true, "round_robin": true}
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_slug ON tenants(slug);
```

## Patients

```sql
CREATE TABLE patients (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    mpi_id          UUID NOT NULL,
    name_family     TEXT NOT NULL,
    name_given      TEXT,
    date_of_birth   DATE NOT NULL,
    gender          TEXT CHECK (gender IN ('male', 'female', 'other', 'unknown')),
    race            TEXT,
    ethnicity       TEXT,
    address_zip     TEXT,
    address_state   TEXT,
    deceased        BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    attribution_provider_npi TEXT,
    payer_name      TEXT,

    -- Identity links across sources
    identities_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "data_source_id": "uuid", "mrn": "MRN12345", "fhir_id": "pat-001",
    --   "identifier_system": "urn:oid:1.2.3.4", "match_confidence": 0.98,
    --   "match_method": "probabilistic"
    -- }]

    -- Embedded conditions (active problem list + historical)
    conditions_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "code_system": "ICD-10-CM", "code": "E11.65",
    --   "display": "Type 2 diabetes with hyperglycemia",
    --   "category": "problem_list", "clinical_status": "active",
    --   "onset_date": "2020-03-15", "recorded_date": "2020-03-15",
    --   "hcc_category": "HCC19", "hcc_model_version": "V28",
    --   "is_nlp_extracted": false, "data_source_id": "uuid"
    -- }]

    -- Embedded observations (labs, vitals, SDOH, PROs)
    observations_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "code_system": "LOINC", "code": "4548-4",
    --   "display": "Hemoglobin A1c", "category": "laboratory",
    --   "value_quantity": 7.2, "value_unit": "%",
    --   "effective_date": "2026-05-15", "status": "final",
    --   "data_source_id": "uuid"
    -- }]

    -- Embedded medications
    medications_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "code_system": "RxNorm", "code": "860975",
    --   "display": "Metformin 500mg", "status": "active",
    --   "start_date": "2020-04-01", "dosage": "500mg BID",
    --   "prescriber_npi": "1234567890"
    -- }]

    -- Embedded procedures
    procedures_json JSONB NOT NULL DEFAULT '[]',

    -- Risk scores
    risk_json       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "hcc_raf": {"score": 1.45, "tier": "high", "date": "2026-05-01",
    --     "categories": ["HCC19", "HCC85"], "model_version": "V28"},
    --   "readmission_30d": {"score": 0.22, "tier": "moderate", "date": "2026-05-25"},
    --   "chronic_condition": {"score": 0.68, "tier": "high", "conditions": ["diabetes", "ckd"]}
    -- }

    -- Care gaps
    care_gaps_json  JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "measure_id": "CMS122v12", "measure_title": "Diabetes: HbA1c Control",
    --   "gap_type": "open", "priority": "high",
    --   "status": "assigned", "assigned_to": "uuid",
    --   "due_date": "2026-12-31",
    --   "description": "No HbA1c result in measurement period",
    --   "recommended_action": "Order HbA1c lab test"
    -- }]

    -- SDOH data
    sdoh_json       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "food_insecurity": {"screen_date": "2026-03-15", "result": "at_risk"},
    --   "housing_instability": {"screen_date": "2026-03-15", "result": "stable"},
    --   "transportation": {"screen_date": "2026-03-15", "result": "adequate"},
    --   "z_codes": ["Z59.41", "Z56.0"]
    -- }

    -- NLP extractions summary
    nlp_json        JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "last_processed_at": "2026-05-28T...",
    --   "documents_processed": 12,
    --   "extractions": [
    --     {"type": "condition", "code": "E11.65", "confidence": 0.92, "source_doc": "discharge_summary_2026-05-20"}
    --   ]
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_patients_tenant ON patients(tenant_id);
CREATE INDEX idx_patients_mpi ON patients(mpi_id);
CREATE INDEX idx_patients_dob ON patients(tenant_id, date_of_birth);
CREATE INDEX idx_patients_zip ON patients(tenant_id, address_zip);
CREATE INDEX idx_patients_provider ON patients(tenant_id, attribution_provider_npi);
CREATE INDEX idx_patients_conditions ON patients USING GIN (conditions_json);
CREATE INDEX idx_patients_observations ON patients USING GIN (observations_json);
CREATE INDEX idx_patients_risk ON patients USING GIN (risk_json);
CREATE INDEX idx_patients_care_gaps ON patients USING GIN (care_gaps_json);
```

## Encounters & Measures

```sql
CREATE TABLE encounters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    patient_id      UUID NOT NULL REFERENCES patients(id),
    data_source_id  UUID NOT NULL,
    fhir_id         TEXT,
    encounter_class TEXT NOT NULL CHECK (encounter_class IN (
        'ambulatory', 'emergency', 'inpatient', 'observation', 'virtual', 'home_health'
    )),
    status          TEXT NOT NULL CHECK (status IN ('planned', 'arrived', 'in_progress', 'finished', 'cancelled')),
    period_start    TIMESTAMPTZ NOT NULL,
    period_end      TIMESTAMPTZ,
    facility_npi    TEXT,
    provider_npi    TEXT,
    discharge_disposition TEXT,

    -- Encounter-level clinical data
    diagnoses_json  JSONB NOT NULL DEFAULT '[]',
    -- [{"code_system": "ICD-10-CM", "code": "E11.65", "rank": "primary", "poa": true}]

    procedures_json JSONB NOT NULL DEFAULT '[]',
    -- [{"code_system": "CPT", "code": "83036", "display": "HbA1c test"}]

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (period_start);

CREATE INDEX idx_encounters_patient ON encounters(patient_id);
CREATE INDEX idx_encounters_tenant ON encounters(tenant_id, period_start);
CREATE INDEX idx_encounters_class ON encounters(tenant_id, encounter_class);
CREATE INDEX idx_encounters_provider ON encounters(provider_npi);

CREATE TABLE measure_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    patient_id      UUID NOT NULL REFERENCES patients(id),
    measure_id      TEXT NOT NULL,
    measure_set     TEXT NOT NULL CHECK (measure_set IN ('hedis', 'cms_ecqm', 'mips', 'ahrq', 'custom')),
    measurement_year INT NOT NULL,
    measurement_period_start DATE NOT NULL,
    measurement_period_end   DATE NOT NULL,
    in_initial_population BOOLEAN NOT NULL DEFAULT false,
    in_denominator  BOOLEAN NOT NULL DEFAULT false,
    in_denominator_exclusion BOOLEAN NOT NULL DEFAULT false,
    in_numerator    BOOLEAN NOT NULL DEFAULT false,
    performance_met BOOLEAN,
    evidence        JSONB NOT NULL DEFAULT '{}',
    calculated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    cql_engine_version TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, measure_id, patient_id, measurement_period_start)
);

CREATE INDEX idx_measure_results_patient ON measure_results(patient_id);
CREATE INDEX idx_measure_results_measure ON measure_results(tenant_id, measure_id, measurement_year);
CREATE INDEX idx_measure_results_gap ON measure_results(tenant_id, measure_id)
    WHERE NOT performance_met AND in_denominator AND NOT in_denominator_exclusion;
```

## AI & Audit

```sql
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
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Configuration | 1 | tenants (embeds users, data sources, programmes, providers, config) |
| Patients | 1 | patients (embeds identities, conditions, observations, medications, procedures, risk, care gaps, SDOH, NLP) |
| Encounters & Measures | 2 | encounters (partitioned, embeds diagnoses/procedures), measure_results |
| AI & Audit | 2 | ai_suggestions, audit_log (partitioned) |
| **Total** | **6** | 2 partitioned tables |

---

## Key Design Decisions

1. **Patient as a longitudinal health document** — all clinical facts (conditions, observations, medications, procedures), risk scores, care gaps, SDOH data, and NLP extractions embed on the patient. The care coordinator view is a single-row fetch.

2. **Encounters remain relational** — encounters are the primary join target for CQL-based quality measure calculations (e.g., "qualifying encounters in measurement period"). Keeping them relational with partitioning enables efficient population-level queries.

3. **Measure results relational for quality reporting** — measure_results stores HEDIS/eCQM population membership per patient per measure per period. Quality performance rates are simple COUNT queries with WHERE clauses on denominator/numerator flags.

4. **Conditions and observations embedded with GIN indexes** — patients.conditions_json and patients.observations_json carry GIN indexes for containment queries like `conditions_json @> '[{"code": "E11.65"}]'`, enabling population queries on embedded clinical data.

5. **HCC mapping embedded on conditions** — each condition in conditions_json carries its HCC category and model version, enabling coding gap identification by scanning the patient's condition array.

6. **Care gaps embedded on patient** — care_gaps_json co-locates gaps with the clinical data that explains them. The care coordinator sees gaps alongside the patient's conditions and observations in one view.

7. **SDOH as a first-class embedded section** — sdoh_json stores screening results, Z-codes, and risk factors as a named section on the patient, supporting SDOH-informed risk stratification without a separate table.

8. **NLP extractions tracked on patient** — nlp_json summarises clinical NLP outputs with confidence scores and source document references, enabling clinician review of AI-extracted data directly on the patient record.
