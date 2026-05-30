# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Healthcare Outcomes Analytics · Created: 2026-05-29

## Philosophy

This model treats every clinical observation, diagnosis, encounter, quality measure calculation, risk score, and care gap action as an immutable event in a single append-only store. The patient's clinical state is not a mutable record — it is a projection computed by replaying clinical events from all source systems. Quality measure results are events produced by CQL evaluation runs. Care gap lifecycle (opened, assigned, outreach attempted, closed) is an event sequence. Risk scores and HCC coding assessments are events with full model input traceability.

Healthcare is one of the strongest natural fits for event sourcing: clinical data is inherently temporal (observations happen at specific times, conditions have onset and resolution dates, medications have start and stop), regulatory requirements demand complete audit trails (HIPAA, ONC interoperability rules), and quality measure calculations must be reproducible at any historical point. An event-sourced model natively supports "what was this patient's clinical state on date X?" queries, retroactive measure recalculation when new data arrives, and permanent recording of every AI-generated clinical suggestion for malpractice defence and regulatory audit.

**Best for:** Organisations requiring full temporal clinical audit trails, retroactive quality measure recalculation when delayed claims or lab results arrive, HIPAA-compliant event-level access logging, and permanent traceability of AI-generated clinical recommendations.

**Trade-offs:**
- (+) Complete temporal clinical history for every patient from every source
- (+) Retroactive measure recalculation when delayed data arrives — replay from the new event
- (+) Every AI clinical suggestion permanently recorded for malpractice and regulatory audit
- (+) HIPAA audit at event level — who created, accessed, or modified every clinical fact
- (-) Quality measure population queries require materialised read models
- (-) Patient lookup ("current medications") requires read model, not direct query
- (-) High event volume for large patient populations requires careful partition management
- (-) CQL engine must query read models rather than normalised FHIR tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | Clinical events carry FHIR resource structure in event data |
| CloudEvents v1.0 | Event store uses CloudEvents envelope; ce_source identifies FHIR server origin |
| HL7 QI-Core | Measure calculations query rm_measure_population projection |
| CQL v1.5.3 | Measure logic executes against read models aligned with QI-Core |
| CMS HCC | HCC coding assessments stored as events with full category mapping |
| HIPAA Security Rule | Event-level audit trail with PHI access tracking |
| SMART on FHIR | App launch context recorded as events for session audit |

---

## Event Infrastructure

```sql
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL CHECK (stream_type IN (
        'tenant', 'user', 'data_source', 'provider',
        'patient', 'encounter', 'condition', 'observation',
        'procedure', 'medication', 'immunization',
        'measure', 'care_gap', 'risk', 'nlp',
        'ai', 'config'
    )),
    stream_id       UUID NOT NULL,
    version         BIGINT NOT NULL,
    event_type      TEXT NOT NULL,
    actor_type      TEXT NOT NULL CHECK (actor_type IN (
        'user', 'system', 'api_key', 'smart_app', 'ai',
        'fhir_server', 'claims_feed', 'lab_interface', 'pharmacy',
        'hl7v2_adt', 'nlp_pipeline', 'cql_engine',
        'scheduler', 'projection_engine'
    )),
    actor_id        TEXT,
    tenant_id       UUID,
    patient_id      UUID,  -- denormalised for HIPAA audit queries

    -- CloudEvents envelope
    ce_source       TEXT NOT NULL,  -- e.g., 'epic/fhir/R4/acme-health'
    ce_type         TEXT NOT NULL,
    ce_time         TIMESTAMPTZ NOT NULL,
    ce_specversion  TEXT NOT NULL DEFAULT '1.0',

    data            JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    phi_level       TEXT NOT NULL DEFAULT 'phi' CHECK (phi_level IN ('phi', 'limited', 'de_identified', 'none')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, version)
) PARTITION BY RANGE (ce_time);

CREATE INDEX idx_events_stream ON event_store(stream_type, stream_id, version);
CREATE INDEX idx_events_tenant ON event_store(tenant_id, ce_time);
CREATE INDEX idx_events_patient ON event_store(patient_id, ce_time);
CREATE INDEX idx_events_type ON event_store(event_type, ce_time);

CREATE TABLE stream_snapshots (
    stream_type     TEXT NOT NULL,
    stream_id       UUID NOT NULL,
    version         BIGINT NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_type, stream_id, version)
);

CREATE TABLE projection_checkpoints (
    projection_name TEXT NOT NULL,
    partition_key   TEXT NOT NULL DEFAULT '_global',
    last_event_id   UUID NOT NULL,
    last_event_time TIMESTAMPTZ NOT NULL,
    state           JSONB NOT NULL DEFAULT '{}',
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (projection_name, partition_key)
);
```

## Event Taxonomy

### Tenant & Configuration Events
- `tenant_created` — {name, slug, organisation_type, plan}
- `data_source_connected` — {source_type, fhir_base_url, auth_method}
- `data_source_sync_completed` — {resources_ingested, duration_ms}
- `measure_programme_configured` — {programme_type, measurement_year, measure_ids}
- `provider_registered` — {npi, name, specialty, facility}

### Patient Identity Events
- `patient_ingested` — {data_source_id, fhir_id, mrn, name, dob, gender, race, ethnicity, address, payer}
- `patient_identity_matched` — {mpi_id, data_source_id, match_confidence, match_method}
- `patient_identity_merged` — {mpi_id, merged_patient_ids}
- `patient_updated` — {fields_changed}
- `patient_deceased` — {deceased_date}
- `patient_attributed` — {provider_npi, effective_date}

### Clinical Events
- `encounter_recorded` — {fhir_id, encounter_class, status, period_start, period_end, facility_npi, provider_npi, discharge_disposition, diagnoses: [{code_system, code, rank, poa}], procedures: [{code_system, code}]}
- `condition_recorded` — {fhir_id, code_system, code, display, category, clinical_status, onset_date, data_source_id}
- `condition_resolved` — {condition_id, abatement_date}
- `condition_nlp_extracted` — {code_system, code, display, confidence, source_document, extraction_model}
- `observation_recorded` — {fhir_id, code_system, code, display, category, value_quantity, value_unit, effective_date, data_source_id}
- `observation_nlp_extracted` — {code_system, code, value, confidence, source_document}
- `procedure_recorded` — {fhir_id, code_system, code, display, performed_date, performer_npi}
- `medication_started` — {fhir_id, code_system, code, display, start_date, dosage, prescriber_npi}
- `medication_stopped` — {medication_id, end_date, reason}
- `medication_dispensed` — {code_system, code, quantity, supply_days, pharmacy}
- `immunization_administered` — {code_system, code, display, administered_date}
- `sdoh_screening_completed` — {screening_type, result, z_codes, data_source_id}

### Quality Measure Events
- `measure_evaluated` — {measure_id, measure_set, measurement_year, patient_id, in_initial_population, in_denominator, in_denominator_exclusion, in_numerator, performance_met, evidence, cql_engine_version}
- `measure_recalculated` — {measure_id, patient_id, reason, previous_result, new_result}
- `measure_batch_completed` — {measure_id, measurement_year, total_patients, denominator_count, numerator_count, performance_rate}

### Care Gap Events
- `care_gap_opened` — {patient_id, measure_id, gap_type, priority, description, recommended_action, due_date}
- `care_gap_predicted` — {patient_id, measure_id, predicted_gap_date, confidence, risk_factors}
- `care_gap_assigned` — {care_gap_id, assigned_to}
- `care_gap_outreach_attempted` — {care_gap_id, outreach_method, outcome}
- `care_gap_scheduled` — {care_gap_id, appointment_date}
- `care_gap_closed` — {care_gap_id, closure_reason, closure_evidence}

### Risk Stratification Events
- `hcc_assessment_completed` — {patient_id, categories, raf_score, model_version, contributing_conditions}
- `hcc_coding_gap_identified` — {patient_id, suspected_hcc, supporting_evidence, confidence}
- `risk_score_calculated` — {patient_id, score_type, score_value, risk_tier, feature_contributions, model_version}
- `risk_tier_changed` — {patient_id, score_type, old_tier, new_tier}

### NLP Events
- `nlp_document_processed` — {patient_id, document_type, document_date, extractions_count, model_version}
- `nlp_extraction_validated` — {extraction_id, validator_npi, is_confirmed}
- `nlp_extraction_rejected` — {extraction_id, validator_npi, reason}

### AI Events
- `ai_suggestion_generated` — {suggestion_type, patient_id, title, description, evidence, confidence, model_version}
- `ai_suggestion_accepted` — {suggestion_id}
- `ai_suggestion_dismissed` — {suggestion_id, reason}
- `ai_care_plan_generated` — {patient_id, narrative, risk_factors, recommended_interventions}
- `ai_anomaly_detected` — {anomaly_type, entity_type, entity_id, expected_value, actual_value}

---

## Read Models

```sql
CREATE TABLE rm_patient_summary (
    tenant_id       UUID NOT NULL,
    patient_id      UUID NOT NULL,
    mpi_id          UUID NOT NULL,

    -- Demographics
    name_family     TEXT NOT NULL,
    name_given      TEXT,
    date_of_birth   DATE NOT NULL,
    gender          TEXT,
    race            TEXT,
    ethnicity       TEXT,
    address_zip     TEXT,
    address_state   TEXT,
    deceased        BOOLEAN NOT NULL DEFAULT false,
    payer_name      TEXT,
    attribution_provider_npi TEXT,

    -- Active conditions
    active_conditions JSONB NOT NULL DEFAULT '[]',

    -- Recent observations (last 12 months)
    recent_observations JSONB NOT NULL DEFAULT '[]',

    -- Active medications
    active_medications JSONB NOT NULL DEFAULT '[]',

    -- Risk scores
    risk_scores     JSONB NOT NULL DEFAULT '{}',

    -- HCC status
    hcc_json        JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "raf_score": 1.45, "categories": ["HCC19", "HCC85"],
    --   "coding_gaps": [{"suspected_hcc": "HCC18", "confidence": 0.85}],
    --   "model_version": "V28", "assessment_date": "2026-05-01"
    -- }

    -- Care gaps
    open_care_gaps  JSONB NOT NULL DEFAULT '[]',

    -- SDOH
    sdoh_json       JSONB NOT NULL DEFAULT '{}',

    -- NLP summary
    nlp_json        JSONB NOT NULL DEFAULT '{}',

    -- Data sources contributing
    data_sources    TEXT[] NOT NULL DEFAULT '{}',

    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, patient_id)
);

CREATE INDEX idx_rm_patient_mpi ON rm_patient_summary(mpi_id);
CREATE INDEX idx_rm_patient_provider ON rm_patient_summary(tenant_id, attribution_provider_npi);
CREATE INDEX idx_rm_patient_zip ON rm_patient_summary(tenant_id, address_zip);
CREATE INDEX idx_rm_patient_risk ON rm_patient_summary USING GIN (risk_scores);
CREATE INDEX idx_rm_patient_gaps ON rm_patient_summary USING GIN (open_care_gaps);

CREATE TABLE rm_measure_population (
    tenant_id       UUID NOT NULL,
    measure_id      TEXT NOT NULL,
    measurement_year INT NOT NULL,
    patient_id      UUID NOT NULL,

    -- Population membership
    in_initial_population BOOLEAN NOT NULL DEFAULT false,
    in_denominator  BOOLEAN NOT NULL DEFAULT false,
    in_denominator_exclusion BOOLEAN NOT NULL DEFAULT false,
    in_numerator    BOOLEAN NOT NULL DEFAULT false,
    performance_met BOOLEAN,

    -- Evidence
    evidence        JSONB NOT NULL DEFAULT '{}',
    calculated_at   TIMESTAMPTZ NOT NULL,

    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, measure_id, measurement_year, patient_id)
);

CREATE INDEX idx_rm_measure_gap ON rm_measure_population(tenant_id, measure_id, measurement_year)
    WHERE NOT performance_met AND in_denominator AND NOT in_denominator_exclusion;

CREATE TABLE rm_population_dashboard (
    tenant_id       UUID NOT NULL,
    dashboard_date  DATE NOT NULL,

    -- Population overview
    total_patients      INT NOT NULL DEFAULT 0,
    total_attributed    INT NOT NULL DEFAULT 0,

    -- Risk stratification
    risk_distribution   JSONB NOT NULL DEFAULT '{}',
    -- {"low": 5000, "rising": 1200, "moderate": 800, "high": 300, "very_high": 50}

    -- Quality measure performance
    measure_performance JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "measure_id": "CMS122v12", "title": "Diabetes: HbA1c Control",
    --   "measure_set": "hedis", "denominator": 450, "numerator": 380,
    --   "performance_rate": 0.844, "target_rate": 0.900,
    --   "open_gaps": 70, "predicted_gaps": 15
    -- }]

    -- Care gap summary
    total_open_gaps     INT NOT NULL DEFAULT 0,
    gaps_by_priority    JSONB NOT NULL DEFAULT '{}',
    gaps_by_measure     JSONB NOT NULL DEFAULT '[]',

    -- Provider breakdown
    provider_performance JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "npi": "1234567890", "name": "Dr. Smith",
    --   "attributed_patients": 450, "avg_raf_score": 1.15,
    --   "measure_rates": {"CMS122v12": 0.88, "CMS165v12": 0.92}
    -- }]

    -- HCC coding
    hcc_summary         JSONB NOT NULL DEFAULT '{}',

    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, dashboard_date)
) PARTITION BY RANGE (dashboard_date);

CREATE TABLE rm_care_gap_worklist (
    tenant_id       UUID NOT NULL,
    care_gap_id     UUID NOT NULL,
    patient_id      UUID NOT NULL,

    -- Gap details
    measure_id      TEXT NOT NULL,
    measure_title   TEXT NOT NULL,
    gap_type        TEXT NOT NULL,
    priority        TEXT NOT NULL,
    status          TEXT NOT NULL,
    assigned_to     UUID,
    due_date        DATE,
    description     TEXT NOT NULL,
    recommended_action TEXT,

    -- Patient context
    patient_name    TEXT NOT NULL,
    patient_dob     DATE NOT NULL,
    patient_phone   TEXT,
    attribution_provider TEXT,
    risk_tier       TEXT,

    -- Outreach history
    outreach_history JSONB NOT NULL DEFAULT '[]',

    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, care_gap_id)
);

CREATE INDEX idx_rm_worklist_status ON rm_care_gap_worklist(tenant_id, status);
CREATE INDEX idx_rm_worklist_assigned ON rm_care_gap_worklist(assigned_to) WHERE status IN ('assigned', 'outreach_attempted');
CREATE INDEX idx_rm_worklist_priority ON rm_care_gap_worklist(tenant_id, priority) WHERE status = 'open';
CREATE INDEX idx_rm_worklist_measure ON rm_care_gap_worklist(tenant_id, measure_id);

CREATE TABLE rm_provider_scorecard (
    tenant_id       UUID NOT NULL,
    provider_npi    TEXT NOT NULL,

    -- Provider identity
    name            TEXT NOT NULL,
    specialty       TEXT,
    facility        TEXT,

    -- Panel metrics
    attributed_patients INT NOT NULL DEFAULT 0,
    avg_raf_score   NUMERIC(6,4),
    total_open_gaps INT NOT NULL DEFAULT 0,

    -- Quality measure rates
    measure_rates   JSONB NOT NULL DEFAULT '[]',
    -- [{"measure_id": "CMS122v12", "denominator": 120, "numerator": 105, "rate": 0.875}]

    -- Utilisation
    utilisation_json JSONB NOT NULL DEFAULT '{}',
    -- {"ed_visits_per_1k": 45, "admissions_per_1k": 12, "readmission_rate_30d": 0.08}

    -- Benchmarking
    percentile_rank JSONB NOT NULL DEFAULT '{}',
    -- {"overall": 72, "quality": 80, "utilisation": 65}

    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, provider_npi)
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Read Models | 5 | rm_patient_summary, rm_measure_population, rm_population_dashboard (partitioned), rm_care_gap_worklist, rm_provider_scorecard |
| **Total** | **8** | 2 partitioned tables |

---

## Key Design Decisions

1. **Clinical events from multiple sources** — condition_recorded, observation_recorded, and medication_started events each carry data_source_id and fhir_id, enabling the rm_patient_summary projection to merge clinical facts from Epic, Oracle Health, claims, labs, and pharmacy into one longitudinal view. Conflicting data from different sources is resolved by the projection using configurable source-priority rules.

2. **Quality measures as events** — measure_evaluated events permanently record every CQL evaluation run with its inputs and results. When delayed data arrives (a claim from 3 months ago), a measure_recalculated event replays the evaluation and records both old and new results. The rm_measure_population projection always reflects the latest calculation.

3. **Care gap lifecycle as event sequence** — care_gap_opened → care_gap_assigned → care_gap_outreach_attempted → care_gap_closed forms a complete audit trail of every care coordination action. The rm_care_gap_worklist projection provides the care coordinator's operational view.

4. **HCC coding assessments as events** — hcc_assessment_completed events record every risk adjustment analysis with full category mapping and contributing conditions. hcc_coding_gap_identified events flag suspected missing diagnoses for clinician review. Both are permanently auditable for CMS risk adjustment validation.

5. **NLP extractions as events with validation** — condition_nlp_extracted and observation_nlp_extracted events record AI-extracted clinical data with confidence scores. nlp_extraction_validated and nlp_extraction_rejected events complete the human-in-the-loop workflow, creating a permanent audit trail of AI-assisted clinical documentation.

6. **PHI level on every event** — event_store.phi_level classifies each event's data sensitivity (phi, limited, de_identified, none), enabling fine-grained HIPAA access control. Research queries can filter to de_identified events without accessing PHI.

7. **Patient ID denormalised on events** — event_store.patient_id enables efficient HIPAA audit queries ("show all events accessing patient X's data") without joining to stream metadata.

8. **Provider scorecard as a projection** — rm_provider_scorecard aggregates quality measure rates, utilisation metrics, and benchmarking percentiles per provider from measure, encounter, and risk events. Network-level provider comparison is a single-table query.

9. **Retroactive measure recalculation** — when a late-arriving claim or lab result triggers a new clinical event, the measure projection detects affected measures and emits measure_recalculated events. This replaces the batch-only retrospective approach used by most incumbents.

10. **Predictive care gaps as events** — care_gap_predicted events flag patients at risk of developing care gaps before they open, with confidence scores and risk factors. The rm_care_gap_worklist projection surfaces both open and predicted gaps, enabling proactive outreach.
