# Healthcare Outcomes Analytics — Phased Development Plan

> Project: 395-healthcare-outcomes-analytics · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three data-model suggestions into a concrete, phased implementation. The database schema follows **Data Model Suggestion 1 (Entity-Centric Normalized Relational)** because it delivers the auditable data lineage (FHIR source → clinical fact → measure result → care gap) that an open-source, ONC-aligned quality engine requires, and because CQL evaluation maps naturally onto normalised, QI-Core-shaped tables. Event-sourcing concepts from Suggestion 3 (retroactive measure recalculation, immutable AI-suggestion trail) are folded in selectively as an append-only `audit_log` plus a `measure_run` history, deferring a full event store to the backlog.

---

## Core Requirements (synthesis)

- **What it does**: Unifies FHIR R4 clinical data, claims, pharmacy, lab, and SDOH into a continuously updated population health model; calculates HEDIS/eCQM/AHRQ quality measures via CQL; stratifies population risk; and produces prioritised care-gap worklists — replacing manual chart abstraction for value-based-care quality reporting.
- **Who uses it**: Quality directors, ACO leadership, care managers/coordinators, population health analysts, and physicians (via embedded SMART on FHIR views) at health systems, ACOs, payers, and community health centres.
- **Key differentiators**: Open-source and self-hostable (no 65+ proprietary connectors, no trade-secret AI engine); transparent, auditable CQL measure logic and interrogable AI; lightweight enough for smaller providers; SDOH and patient-reported outcomes as first-class signals.
- **Deployment model**: Self-hostable containerised stack (API + workers + Postgres + object storage + reverse proxy), with managed FHIR stores (HealthLake, GCP Healthcare, HAPI FHIR) as ingestion targets. SaaS multi-tenancy is built in from the schema up.
- **Integration surface**: FHIR R4 servers (Epic, Oracle Health, HAPI FHIR, cloud FHIR) via Bulk Data `$export`; CMS BCDA claims; HL7 v2 ADT/ORU feeds; LLM providers for NLP/generative features; SMART on FHIR EHR launch.
- **Standards compliance**: FHIR R4 + US Core + QI-Core; FHIR Bulk Data (`$export`, NDJSON); CQL v1.5.x + CQF-Measures + Da Vinci DEQM `MeasureReport`; SMART on FHIR v2.2 + OAuth 2.0/OIDC; HIPAA Security Rule (45 CFR 164) with NIST SP 800-66r2 control mapping; OpenAPI 3.1; CMS HCC risk adjustment.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **Python 3.12** | Dominant language for clinical NLP (spaCy, scispaCy, Med7, BioMedBERT via Hugging Face) and for LLM orchestration; the AI-native features are the differentiator and they live in Python. Keeps one language across API, workers, and ML. |
| API framework | **FastAPI** | Async-native (needed for long-running Bulk Data polling and LLM calls), generates **OpenAPI 3.1** automatically (a stated standard), first-class Pydantic validation for FHIR-shaped payloads. |
| Data validation / models | **Pydantic v2** | Strict typed models for config, API I/O, and FHIR resource subsets; serialises to/from the JSONB `evidence`/`config` columns. |
| FHIR resource handling | **fhir.resources** (Pydantic FHIR R4 models) | Validated FHIR R4 + US Core parsing of NDJSON exports without hand-rolling resource schemas. |
| Database | **PostgreSQL 16** | Suggestion 1 relies on declarative range **partitioning** (encounters, observations, audit_log), `JSONB` evidence columns, GIN indexes, and `INET`/array types. HIPAA-grade ACID + row-level security for tenant isolation. SQLite is used only in unit tests via a thin repository abstraction. |
| Migrations | **Alembic** | Versioned, reviewable DDL; partition creation handled in migration helpers. |
| ORM / query layer | **SQLAlchemy 2.0 (core + typed ORM)** | Async engine for FastAPI; raw SQL escape hatch for partition-aware population queries. |
| Task queue | **Celery + Redis** | Bulk `$export` polling, NDJSON ingestion, CQL batch runs, NLP jobs, and risk scoring are long-running and must survive restarts; Celery gives retries, scheduling (Celery Beat for HEDIS refresh cycles), and result tracking. |
| CQL execution | **cqframework CQL engine via a sidecar JVM service** (`cql-engine-svc`) | The mature, validated open-source CQL engine (`cqframework/clinical_quality_language`, Apache 2.0) is JVM-based and is the reference for FHIR-native eCQM/HEDIS dQM. We wrap it in a small HTTP service rather than reimplement CQL in Python — accuracy of measure logic directly affects reimbursement. |
| FHIR data store (optional self-host) | **HAPI FHIR (Apache 2.0)** as a supported ingestion source | Lets self-hosters run a fully open FHIR store; cloud FHIR (HealthLake/GCP/Azure) are alternative sources via the same Bulk Data interface. |
| LLM / NLP | **scispaCy + Med7** (structured extraction) and a pluggable **LLM provider** (OpenAI/Anthropic/local via a `LLMClient` interface) | scispaCy/Med7 for deterministic entity extraction and coding; LLM for care-plan narrative synthesis and HCC coding-gap suggestions, always with confidence scores and human-in-the-loop review. |
| Object storage | **S3-compatible (MinIO self-hosted / AWS S3)** | Stores raw NDJSON export bundles and source clinical documents for NLP, kept out of Postgres. |
| Frontend | **React + TypeScript (Vite) + TanStack Query** dashboard; **SMART on FHIR JS app** for in-EHR embedding | Role-based dashboards (quality, care manager, executive) and an embeddable point-of-care view. Server-rendered is unnecessary; the app is data-dense and interactive. |
| Charts | **Recharts / visx** | Population trend lines, measure performance bars, provider benchmarking. |
| AuthN/AuthZ | **OAuth 2.0 / OIDC** (Authlib) for users + **SMART on FHIR v2.2** launch; **JWT (RFC 7519)** bearer tokens; Postgres **row-level security** for tenant isolation | Mandated by the standards; SMART launch is required for EHR embedding. |
| Secrets | **HashiCorp Vault / cloud KMS** referenced by `credentials_ref` | FHIR/claims source credentials never stored in plaintext; schema already references `credentials_ref`. |
| Containerisation | **Docker + docker-compose** (api, worker, beat, postgres, redis, minio, cql-engine-svc, nginx) | Self-hostable single-command deployment is a core value proposition. |
| Testing | **pytest** (+ pytest-asyncio), **testcontainers** (real Postgres/Redis), **respx/httpx_mock** (mocked FHIR), **Synthea** synthetic FHIR bundles as fixtures | Synthea gives realistic, PHI-free FHIR R4 + US Core data to validate ingestion and measures end-to-end. |
| Code quality | **ruff** (lint+format), **mypy --strict**, **bandit** (security), **pip-audit** | Healthcare code demands typed, security-scanned code; bandit/pip-audit support HIPAA technical safeguard evidence. |
| Package manager | **uv** (with `pyproject.toml`) | Fast, reproducible locking. |
| CI | **GitHub Actions** | Lint, type-check, test matrix, Docker build, security scan, OpenAPI diff. |

### Project Structure

```
healthcare-outcomes-analytics/
├── pyproject.toml
├── uv.lock
├── README.md
├── docker-compose.yml
├── Dockerfile                       # api + worker image
├── .env.example
├── alembic.ini
├── migrations/                      # Alembic versions (incl. partition helpers)
│   └── versions/
├── cql-engine-svc/                  # JVM sidecar wrapping cqframework CQL engine
│   ├── Dockerfile
│   └── src/                         # thin HTTP wrapper + measure bundle loader
├── measures/                        # CQL libraries + value sets (committed)
│   ├── hedis/                       # e.g. CMS122v12 (Diabetes HbA1c), CMS165v12 (BP)
│   ├── ecqm/
│   ├── ahrq/                        # PSI/IQI/PQI logic ported to CQL
│   └── valuesets/                   # VSAC-derived value set JSON
├── src/
│   └── hoa/
│       ├── main.py                  # FastAPI app factory
│       ├── config.py                # Pydantic Settings
│       ├── db/
│       │   ├── engine.py            # async SQLAlchemy engine, RLS session
│       │   ├── models/              # ORM models (one module per domain)
│       │   └── repositories/        # data-access layer (swap Postgres/SQLite)
│       ├── fhir/
│       │   ├── bulk_client.py       # FHIR Bulk Data $export client
│       │   ├── ndjson.py            # streaming NDJSON parser
│       │   └── mappers/             # FHIR resource → relational row mappers
│       ├── ingestion/
│       │   ├── tasks.py             # Celery ingestion tasks
│       │   └── claims.py            # BCDA / 837 claims ingestion
│       ├── mpi/
│       │   ├── matcher.py           # probabilistic + LLM-embedding matching
│       │   └── blocking.py
│       ├── measures/
│       │   ├── cql_client.py        # HTTP client to cql-engine-svc
│       │   ├── runner.py            # batch measure evaluation orchestration
│       │   └── deqm.py              # MeasureReport (DEQM) builder
│       ├── risk/
│       │   ├── hcc.py               # HCC category mapping + RAF
│       │   └── models.py            # readmission / chronic-condition models
│       ├── caregaps/
│       │   ├── generator.py         # measure_result → care_gap
│       │   └── assignment.py        # worklist + round-robin assignment
│       ├── nlp/
│       │   ├── pipeline.py          # scispaCy/Med7 extraction
│       │   └── llm.py               # LLMClient interface + providers
│       ├── ai/
│       │   ├── care_plan.py         # generative narrative synthesis
│       │   ├── coding_gap.py        # HCC coding-gap suggestion
│       │   └── anomaly.py           # measure/data anomaly detection
│       ├── api/
│       │   ├── deps.py              # auth, tenant, pagination deps
│       │   ├── routers/             # patients, measures, caregaps, risk, ...
│       │   └── schemas/             # Pydantic request/response models
│       ├── auth/
│       │   ├── oidc.py              # user OAuth2/OIDC
│       │   ├── smart.py             # SMART on FHIR v2.2 launch
│       │   └── rbac.py              # role → permission mapping
│       ├── audit/
│       │   └── logger.py            # HIPAA audit_log writer
│       └── benchmarking/
│           └── scorecard.py         # provider scorecard aggregation
├── frontend/
│   ├── package.json
│   ├── src/
│   │   ├── routes/                  # dashboards by role
│   │   ├── components/
│   │   └── api/                     # generated OpenAPI client
│   └── smart-app/                   # SMART on FHIR embedded view
└── tests/
    ├── unit/
    ├── integration/
    ├── e2e/
    └── fixtures/
        └── synthea/                 # synthetic FHIR R4 NDJSON bundles
```

---

## Phase 1: Foundation, Schema, and Tenancy

### Purpose
Establish the project skeleton, configuration, the PostgreSQL schema from Data Model Suggestion 1, multi-tenant row-level security, and the HIPAA audit-log spine. After this phase the platform can run, accept authenticated requests, isolate tenants, and record every PHI access — the non-negotiable security baseline for everything that follows.

### Tasks

#### 1.1 — Project scaffold, config, and tooling

**What**: Create the `uv`/`pyproject.toml` project, FastAPI app factory, Pydantic `Settings`, Docker Compose stack, and CI.

**Design**:
```python
# src/hoa/config.py
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="HOA_", env_file=".env")
    database_url: PostgresDsn
    redis_url: RedisDsn
    s3_endpoint: str
    s3_bucket: str = "hoa-clinical"
    cql_engine_url: AnyHttpUrl                      # http://cql-engine-svc:8080
    oidc_issuer: AnyHttpUrl
    oidc_audience: str
    llm_provider: Literal["openai", "anthropic", "local", "disabled"] = "disabled"
    llm_api_key: SecretStr | None = None
    phi_log_retention_days: int = 2190              # 6 years (HIPAA)
    environment: Literal["dev", "staging", "prod"] = "dev"
```
`docker-compose.yml` services: `api`, `worker`, `beat`, `postgres:16`, `redis:7`, `minio`, `cql-engine-svc`, `nginx`. CI runs ruff, mypy --strict, bandit, pip-audit, pytest, docker build, and an OpenAPI schema diff.

**Testing**:
- `Unit: Settings loads from env with defaults → phi_log_retention_days == 2190`
- `Unit: missing HOA_DATABASE_URL → ValidationError naming the field`
- `Integration: GET /healthz → 200 {"status":"ok","db":"ok","redis":"ok"}`
- `Integration: docker compose up → all services healthy within 60s`

#### 1.2 — Database schema & migrations (Suggestion 1)

**What**: Implement all 16 tables from Data Model Suggestion 1 as Alembic migrations with declarative range partitioning on `encounters`, `observations`, and `audit_log`.

**Design**: Tables exactly as specified in `data-model-suggestion-1.md`: `tenants`, `users`, `data_sources`, `patients`, `patient_identities`, `encounters` (partitioned by `period_start`), `conditions`, `observations` (partitioned by `effective_date`), `procedures`, `medications`, `measure_definitions`, `measure_results`, `care_gaps`, `risk_scores`, `ai_suggestions`, `audit_log` (partitioned by `created_at`). Add a partition-management helper:
```python
def ensure_monthly_partitions(conn, table: str, months_ahead: int = 3) -> None:
    """CREATE TABLE IF NOT EXISTS <table>_YYYY_MM PARTITION OF <table>
       FOR VALUES FROM (...) TO (...)."""
```
A Celery Beat job calls it monthly. Enable `ROW LEVEL SECURITY` on every tenant-scoped table:
```sql
ALTER TABLE patients ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON patients
  USING (tenant_id = current_setting('hoa.tenant_id')::uuid);
```
The async session sets `SET LOCAL hoa.tenant_id = :tenant` per request.

**Testing**:
- `Integration (testcontainers Postgres): run all migrations → 16 tables + expected indexes exist`
- `Integration: insert encounter dated next month → lands in correct partition`
- `Integration: session A (tenant 1) cannot SELECT tenant 2 rows under RLS`
- `Unit: ensure_monthly_partitions is idempotent (second call no-ops)`
- `Integration: alembic downgrade base → upgrade head round-trips cleanly`

#### 1.3 — HIPAA audit logging spine

**What**: A middleware + helper that writes an `audit_log` row for every API request touching PHI, with `phi_accessed` correctly set.

**Design**:
```python
@dataclass
class AuditEvent:
    actor_type: Literal["user","system","api_key","smart_app","ai","scheduler","fhir_client"]
    actor_id: str | None
    action: str                      # "read","create","update","delete","export"
    resource_type: str
    resource_id: UUID | None
    patient_id: UUID | None
    phi_accessed: bool
    changes: dict | None = None
    ip_address: str | None = None
    user_agent: str | None = None

async def record(ev: AuditEvent, tenant_id: UUID) -> None: ...
```
Routes declare PHI exposure via a dependency (`phi_resource=True`); middleware records after the response. Audit writes are append-only (no UPDATE/DELETE grant on `audit_log`).

**Testing**:
- `Integration: GET /patients/{id} → audit_log row with phi_accessed=true, action="read"`
- `Integration: GET /measures (aggregate, no PHI) → phi_accessed=false`
- `Unit: record() denormalises patient_id and tenant_id correctly`
- `Integration: attempt UPDATE on audit_log via app role → permission denied`

---

## Phase 2: Authentication, RBAC & SMART on FHIR

### Purpose
Add user authentication (OAuth 2.0/OIDC), role-based access control aligned to the schema's role enum, service-to-service auth for back-end bulk jobs, and the SMART on FHIR v2.2 launch flow required to embed views in Epic/Oracle Health. After this phase, every endpoint is access-controlled and the EHR-embedding path is established.

### Tasks

#### 2.1 — User OIDC login & JWT sessions

**What**: OIDC authorization-code login, JWT issuance, and a `get_current_user` dependency.

**Design**: Authlib client against `oidc_issuer`. JWT (RFC 7519) claims: `sub`, `tenant_id`, `role`, `npi`, `exp`. `get_current_user` validates signature/audience/expiry and loads the `users` row (must be `is_active`). Token refresh endpoint.

**Testing**:
- `Integration (mocked OIDC): valid code → JWT with tenant_id + role claims`
- `Unit: expired JWT → 401`
- `Unit: token for is_active=false user → 403`

#### 2.2 — RBAC permission matrix

**What**: Map the seven roles (`admin`, `quality_director`, `care_manager`, `physician`, `analyst`, `viewer`, `api_service`) to permissions enforced by a `require(permission)` dependency.

**Design**:
```python
PERMISSIONS: dict[str, set[str]] = {
  "admin": {"*"},
  "quality_director": {"measures:read","measures:run","reports:submit","patients:read","caregaps:read"},
  "care_manager": {"caregaps:read","caregaps:write","patients:read","ai:read"},
  "physician": {"patients:read","caregaps:read","ai:read","nlp:validate"},
  "analyst": {"measures:read","patients:read","benchmark:read"},
  "viewer": {"measures:read","benchmark:read"},
  "api_service": {"fhir:ingest","measures:run"},
}
```

**Testing**:
- `Unit: care_manager has caregaps:write, lacks measures:run`
- `Integration: viewer POST /measures/run → 403; quality_director → 202`
- `Unit: admin "*" satisfies any permission check`

#### 2.3 — SMART on FHIR v2.2 launch + back-end services auth

**What**: Implement SMART EHR launch (clinician-facing) and OAuth 2.0 client-credentials (back-end bulk data), per SMART App Launch v2.2.

**Design**: Endpoints `/smart/launch`, `/smart/callback`; store launch context (patient, encounter, fhirUser) for audit. Back-end services flow uses signed JWT client assertion to obtain a bulk-data access token from the source FHIR server; tokens cached in Redis until expiry. Scopes follow SMART 2.0 granular syntax (`patient/*.rs`, `system/*.rs`).

**Testing**:
- `Integration (mocked auth server): client-credentials flow → bearer token cached`
- `Integration: SMART launch with patient context → context persisted + audit row`
- `Unit: requested scope exceeds granted → launch rejected`

---

## Phase 3: FHIR Bulk Ingestion & Normalisation

### Purpose
Build the data backbone: connect a data source, run a FHIR Bulk Data `$export`, stream the NDJSON, validate against US Core, and map resources into the normalised clinical tables. This is the heart of replacing manual abstraction — once clinical data lands in queryable tables, measures and risk become possible.

### Tasks

#### 3.1 — FHIR Bulk Data `$export` client

**What**: An async client implementing the FHIR Bulk Data Access (Flat FHIR) kick-off → poll → download flow.

**Design**:
```python
class BulkExportClient:
    async def kickoff(self, source: DataSource, since: datetime | None,
                      types: list[str]) -> str: ...          # returns polling URL
    async def poll(self, polling_url: str) -> BulkStatus: ...  # 202 in-progress | 200 complete
    async def download(self, file_url: str) -> AsyncIterator[bytes]: ...
```
Honours `Retry-After` (RFC 7231) and `Link` headers (RFC 8288) for pagination. Uses the bearer token from 2.3. Default `types`: `Patient,Encounter,Condition,Observation,Procedure,MedicationRequest,MedicationStatement,Immunization`. Raw NDJSON files saved to S3/MinIO keyed by `data_source_id` + export timestamp.

**Testing**:
- `Integration (respx-mocked FHIR): kickoff → 202 with Content-Location → poll → 200 → file URLs`
- `Unit: 429 with Retry-After → client waits and retries`
- `Integration: download streams large NDJSON without loading fully into memory`

#### 3.2 — Streaming NDJSON parser & FHIR validation

**What**: Parse NDJSON line-by-line, validate each resource with `fhir.resources` against R4 + US Core profiles.

**Design**:
```python
async def parse_ndjson(stream: AsyncIterator[bytes],
                       resource_type: str) -> AsyncIterator[FHIRResource]: ...
```
Invalid resources are quarantined (logged + counted) rather than aborting the batch. US Core required-element checks emit `data_completeness` warnings.

**Testing**:
- `Fixture: Synthea Patient NDJSON → validated Patient models`
- `Unit: malformed JSON line → quarantined, batch continues`
- `Unit: Observation missing US Core status → completeness warning recorded`

#### 3.3 — FHIR → relational mappers

**What**: Map each FHIR resource type to its `data-model-suggestion-1` table row.

**Design**: One mapper per resource implementing `map(resource, tenant_id, data_source_id) -> Row`. Mappings: `Patient→patients`, `Encounter→encounters` (`class` → `encounter_class`), `Condition→conditions` (system/code/category/clinical_status; populate `hcc_category` in Phase 6), `Observation→observations` (LOINC/category; `sdoh` category from US Core SDOH profiles), `Procedure→procedures`, `MedicationRequest/Statement→medications`. Upsert keyed by `(tenant_id, data_source_id, fhir_id)`.

**Testing**:
- `Unit: Encounter.class="IMP" → encounter_class="inpatient"`
- `Unit: re-ingesting same fhir_id updates, not duplicates`
- `Fixture: full Synthea bundle → expected row counts per table`
- `Unit: SDOH Observation → category="sdoh"`

#### 3.4 — Ingestion orchestration (Celery)

**What**: A Celery task chain: kick off export → poll → download → parse → map → upsert → update `data_sources.status`/`last_sync_at`, with progress and error capture.

**Design**: `ingest_source(data_source_id)` task; status transitions `pending→syncing→active|error`. Per-resource-type subtasks run in parallel. Emits a `data_source_sync_completed`-style audit row with `resources_ingested`.

**Testing**:
- `Integration (mocked FHIR + real Postgres/Redis): ingest_source → patients/encounters populated, status="active"`
- `Integration: download failure mid-run → status="error", partial data retained, retryable`
- `E2E: register source → trigger ingest → GET /patients shows ingested cohort`

---

## Phase 4: Master Patient Index

### Purpose
The same patient appears across Epic, Oracle Health, claims, and labs under different identifiers. The MPI links these into one `mpi_id` so measures and risk operate on a complete longitudinal record. Without it, denominators are wrong and care gaps duplicate.

### Tasks

#### 4.1 — Deterministic + probabilistic matching

**What**: Match incoming patients to existing `mpi_id`s using blocking + scored comparison, writing `patient_identities` with confidence and method.

**Design**: Blocking keys (e.g. `soundex(name_family)+dob`, `zip+dob`). Field comparators (Jaro-Winkler on names, exact DOB/gender, normalised address) produce a Fellegi–Sunter score. Thresholds: `>=0.95` auto-link (`probabilistic`), `0.80–0.95` queue for manual review, `<0.80` new `mpi_id`. Exact SSN/MRN match → `deterministic`, confidence `1.0`.
```python
@dataclass
class MatchCandidate:
    existing_mpi_id: UUID
    score: float
    method: Literal["deterministic","probabilistic","manual","llm_embedding"]
```

**Testing**:
- `Unit: identical name+dob+zip → score >= 0.95 auto-link`
- `Unit: same SSN, differing name spelling → deterministic link, 1.0`
- `Unit: twins (same dob/zip, different given name) → below auto threshold → review queue`
- `Integration: two source patients linked → share mpi_id, two patient_identities rows`

#### 4.2 — LLM-embedding-assisted matching (AI-native)

**What**: For borderline (0.80–0.95) candidates, use embedding similarity over normalised demographic strings to nudge the decision, recorded as `llm_embedding` with confidence.

**Design**: `LLMClient.embed(text)` → vector; cosine similarity blended with the rule-based score. Every embedding-assisted decision writes an `ai_suggestion` (`suggestion_type` not applicable here → use audit) and is reversible. Falls back to manual-review queue when `llm_provider="disabled"`.

**Testing**:
- `Unit (mocked embeddings): borderline pair with high embedding similarity → linked, method="llm_embedding"`
- `Unit: llm disabled → borderline pair stays in review queue`
- `Integration: manual merge endpoint reassigns mpi_id and logs audit`

---

## Phase 5: CQL Measure Engine & Quality Calculation

### Purpose
Deliver the core product value: automated, auditable HEDIS/eCQM/AHRQ measure calculation via CQL against the normalised clinical data, producing per-patient population membership and DEQM-conformant reports. This replaces manual chart abstraction.

### Tasks

#### 5.1 — CQL engine sidecar (`cql-engine-svc`)

**What**: A JVM HTTP service wrapping the cqframework CQL engine that evaluates a measure against a patient bundle and returns population flags.

**Design**: `POST /evaluate {measureId, periodStart, periodEnd, patientBundle}` → `{initialPopulation, denominator, denominatorExclusion, numerator, numeratorExclusion, exception, evidence}`. Loads CQL libraries + value sets from the `measures/` directory at startup. Data passed as a FHIR Bundle assembled from the patient's relational rows (reverse of the 3.3 mappers).

**Testing**:
- `Integration: evaluate CMS122v12 for a Synthea diabetic with HbA1c 6.5% → in_numerator=true`
- `Integration: diabetic with no HbA1c in period → in_denominator=true, in_numerator=false`
- `Unit: unknown measureId → 404`

#### 5.2 — Measure definitions & value sets

**What**: Load `measure_definitions` rows and commit at least three reference measures end-to-end: CMS122v12 (Diabetes HbA1c Poor Control), CMS165v12 (Controlling High Blood Pressure), and one AHRQ PQI (e.g. PQI01 Diabetes Short-Term Complications) ported to CQL.

**Design**: Each measure: CQL library in `measures/`, VSAC-derived value sets in `measures/valuesets/`, and a `measure_definitions` row (`measure_id`, `measure_set`, `version`, `cql_library`, `measurement_year`). A `measure-import` CLI registers them per tenant.

**Testing**:
- `Unit: import CMS122v12 → measure_definitions row with measure_set="cms_ecqm"`
- `Fixture: value set load → expected code count for HbA1c LOINC set`
- `Integration: PQI01 CQL evaluates against an inpatient Synthea encounter`

#### 5.3 — Batch measure runner & evidence capture

**What**: Evaluate a measure across a population, persist `measure_results` per patient with `evidence` (qualifying encounter/condition/observation IDs), and record a measure-run history entry.

**Design**:
```python
async def run_measure(tenant_id: UUID, measure_def_id: UUID,
                      period: DateRange) -> MeasureRunSummary: ...
```
Celery task fans out per patient batch → calls `cql-engine-svc` → upserts `measure_results` (unique on tenant/measure/patient/period) with `evidence` JSONB capturing contributing row IDs and a CQL evaluation log. Returns denominator/numerator counts and performance rate.

**Testing**:
- `Integration: run CMS122v12 over 100 Synthea patients → measure_results rows + correct rate`
- `Unit: evidence captures qualifying observation IDs (click-through audit)`
- `Integration: rerun with new lab result → patient flips to numerator, prior result superseded, audit recorded`

#### 5.4 — DEQM MeasureReport export

**What**: Produce Da Vinci DEQM `MeasureReport` resources (individual + summary) for regulatory/payer submission.

**Design**: `build_measure_report(measure_def, results, kind="summary"|"individual") -> MeasureReport`. Summary report aggregates population counts; individual reports include `evaluatedResource` references. Exposed at `GET /measures/{id}/report?period=&kind=`.

**Testing**:
- `Unit: summary report population counts equal measure_results aggregates`
- `Unit: report validates against DEQM MeasureReport profile`
- `Integration: GET report endpoint → valid FHIR MeasureReport JSON`

---

## Phase 6: Risk Stratification & HCC Coding

### Purpose
Stratify the population by risk and identify HCC coding gaps that affect risk-adjustment revenue. Risk tiers drive care-gap prioritisation; HCC gaps surface unbilled diagnoses. Both are core value-based-care capabilities.

### Tasks

#### 6.1 — HCC category mapping & RAF

**What**: Map `conditions` to CMS HCC categories (model V28), compute a patient RAF score, and store `risk_scores` (`score_type="hcc_raf"`).

**Design**: ICD-10-CM → HCC crosswalk loaded from a committed CMS mapping file; back-fills `conditions.hcc_category`/`hcc_model_version`. RAF = demographic factor + sum of HCC coefficients (hierarchy-trimmed). Stores `hcc_categories[]` and `contributing_conditions` JSONB on `risk_scores`.

**Testing**:
- `Unit: E11.65 → HCC18/19 per V28 crosswalk`
- `Unit: hierarchy trimming keeps only highest HCC in a family`
- `Integration: patient with three chronic conditions → expected RAF within tolerance`

#### 6.2 — Predictive risk models

**What**: Readmission-30d, ED-utilisation, and chronic-condition risk scores with explainable feature contributions.

**Design**: Gradient-boosted models (or transparent logistic baseline) trained on engineered features (prior admissions, condition counts, SDOH flags, med adherence proxies). Output `score_value`, `risk_tier`, and `feature_contributions` JSONB (SHAP-style) — transparency is the differentiator. Models versioned via `model_version`.

**Testing**:
- `Unit: feature vector builder produces stable schema from patient rows`
- `Unit: score maps to correct tier per thresholds`
- `Integration: scoring run populates risk_scores with feature_contributions`

#### 6.3 — HCC coding-gap suggestions (AI-native)

**What**: Flag suspected HCCs supported by clinical evidence but not coded, as `ai_suggestions` (`suggestion_type="hcc_coding_gap"`) for clinician review.

**Design**: Combine rule logic (e.g. active diabetes meds + labs but no diabetes HCC condition) with optional LLM analysis of NLP-extracted notes. Each suggestion carries `evidence`, `confidence`, `recommended_action`, and stays `pending` until accepted/dismissed (human-in-the-loop).

**Testing**:
- `Unit: metformin + A1c 8% + no diabetes HCC → coding-gap suggestion`
- `Unit: confidence + evidence populated; status="pending"`
- `Integration: accept suggestion → audit row; dismiss → resolved_at set`

---

## Phase 7: Care Gap Generation & Coordinator Workflow

### Purpose
Turn measure non-compliance and predicted gaps into an actionable, prioritised worklist for care coordinators — the operational payoff that drives clinician adoption.

### Tasks

#### 7.1 — Care-gap generation

**What**: Generate `care_gaps` from `measure_results` where the patient is in the denominator, not excluded, and not in the numerator.

**Design**: `gap_type="open"`; `priority` derived from risk tier + measure weight; `recommended_action` from measure metadata (e.g. "Order HbA1c lab test"). Links `measure_result_id` for click-through. Idempotent per `(patient, measure, period)`.

**Testing**:
- `Unit: denominator & not numerator & not excluded → open gap created`
- `Unit: numerator-met patient → no gap`
- `Unit: high-risk-tier patient → priority escalated`

#### 7.2 — Predictive care gaps (AI-native)

**What**: Flag patients likely to miss an upcoming screening before the gap opens (`gap_type="predicted"`).

**Design**: Uses Phase 6 risk signals + historical adherence to predict gap-open probability; creates `care_gaps` with `gap_type="predicted"` and a paired `ai_suggestion` (`care_gap_predicted`) carrying confidence and risk factors.

**Testing**:
- `Unit: patient with lapsing screening pattern → predicted gap with confidence`
- `Integration: predicted gap appears in worklist alongside open gaps`

#### 7.3 — Worklist & assignment

**What**: Care-coordinator worklist endpoints with filtering, prioritisation, assignment (incl. round-robin), and lifecycle transitions.

**Design**: `GET /caregaps?status=&priority=&assigned_to=&measure_id=` (paginated, RBAC-gated). Lifecycle: `open→assigned→outreach_attempted→scheduled→closed_{met,excluded,expired}`. Auto-assignment honours tenant `config`. Closing with `met` records `closure_evidence`.

**Testing**:
- `Integration: care_manager lists own assigned gaps, sorted by priority`
- `Unit: invalid transition (closed→assigned) → 409`
- `Integration: auto-assign round-robin distributes evenly across coordinators`
- `Integration: viewer cannot write gaps → 403`

---

## Phase 8: Clinical NLP Pipeline

### Purpose
Extract quality-measure-relevant data from unstructured notes (the manual-abstraction replacement). NLP-extracted conditions/observations feed measures and HCC, flagged for clinician validation.

### Tasks

#### 8.1 — scispaCy/Med7 extraction

**What**: Process clinical documents (discharge summaries, notes) and extract conditions/observations with codes and confidence.

**Design**:
```python
async def extract(document_text: str, doc_type: str) -> list[Extraction]
@dataclass
class Extraction:
    kind: Literal["condition","observation"]
    code_system: str; code: str; display: str
    value: float | str | None
    confidence: float
    source_span: tuple[int,int]
```
Writes `conditions`/`observations` with `is_nlp_extracted=true`, `nlp_confidence`. Documents pulled from S3.

**Testing**:
- `Fixture: discharge summary mentioning "Type 2 diabetes" → condition E11.x extraction`
- `Unit: confidence + source_span populated; is_nlp_extracted=true`
- `Unit: negated mention ("no evidence of CHF") → not extracted`

#### 8.2 — Human-in-the-loop validation

**What**: Clinician review queue to confirm/reject NLP extractions before they count toward measures.

**Design**: `GET /nlp/extractions?status=pending`; validate/reject endpoints (physician role). Rejected extractions excluded from measure evaluation. Every decision audited.

**Testing**:
- `Integration: physician validates extraction → counts toward measure rerun`
- `Integration: rejected extraction excluded from CQL bundle`
- `Unit: non-physician validate → 403`

---

## Phase 9: Dashboards, Benchmarking & Generative AI

### Purpose
Deliver role-based dashboards, provider network benchmarking, and generative AI care-plan narratives — the user-facing surfaces and the highest-value AI feature.

### Tasks

#### 9.1 — Role-based dashboards (frontend)

**What**: React dashboards for quality director, care manager, and executive personas consuming the OpenAPI client.

**Design**: Routes per role; quality view shows measure performance vs target; care-manager view is the worklist; executive view shows population risk distribution + trends. TanStack Query against the API; charts via Recharts. Auth via OIDC.

**Testing**:
- `E2E (Playwright): quality_director sees measure rates; care_manager sees worklist`
- `E2E: viewer cannot access write actions`
- `Unit: API client generated from OpenAPI 3.1 matches server schema`

#### 9.2 — Provider network benchmarking

**What**: Aggregate measure rates, RAF, and utilisation per provider with percentile ranking across the network.

**Design**: `scorecard.py` aggregates `measure_results`, `risk_scores`, and `encounters` per `provider_npi`; `GET /benchmark/providers` returns rates + percentile ranks. Cached per refresh cycle.

**Testing**:
- `Integration: provider scorecard rates match underlying measure_results`
- `Unit: percentile ranking correct for a known distribution`

#### 9.3 — Generative care-plan narratives (AI-native)

**What**: Convert a patient's risk scores, gaps, and conditions into a plain-language care-plan narrative for care managers.

**Design**: `LLMClient.generate(prompt)` with a structured template injecting active conditions, open gaps, risk tier, and SDOH factors. Output stored as `ai_suggestions` (`care_plan_narrative`), `status="pending"`, never auto-applied. Prompt template:
```
System: You are a care-coordination assistant. Summarise this patient's
care priorities in plain language for a care manager. Cite the specific
gaps and risk factors provided. Do not invent clinical facts.
User: Conditions: {conditions}. Open gaps: {gaps}. Risk: {risk}. SDOH: {sdoh}.
```

**Testing**:
- `Unit (mocked LLM): narrative references provided gaps only (no fabrication beyond inputs)`
- `Unit: llm disabled → feature returns 503 cleanly`
- `Integration: generated narrative stored as pending ai_suggestion`

#### 9.4 — Measure & data anomaly detection (AI-native)

**What**: Detect measure-calculation anomalies and data-completeness issues, surfaced as `ai_suggestions`.

**Design**: Statistical checks (sudden rate shifts, denominator drops, source completeness deltas) plus optional LLM explanation; emits `measure_anomaly`/`data_completeness` suggestions with severity.

**Testing**:
- `Unit: 30% denominator drop vs prior run → anomaly suggestion (severity=warning)`
- `Unit: source missing required US Core elements → data_completeness suggestion`

---

## Phase 10: SMART on FHIR App, Hardening & Self-Host Packaging

### Purpose
Ship the embeddable point-of-care view, complete HIPAA/NIST hardening, and produce the one-command self-host deployment that lets smaller providers adopt without enterprise SaaS pricing.

### Tasks

#### 10.1 — SMART on FHIR embedded app

**What**: A standalone JS app launched from the EHR (using Phase 2.3 launch) showing the patient's open gaps, risk tier, and HCC suggestions in-workflow.

**Design**: `frontend/smart-app/` launched via SMART context; reads patient via launch `patient` context; calls `/patients/{id}/caregaps` and `/patients/{id}/risk`. Minimal, fast, read-only.

**Testing**:
- `E2E (mocked SMART launch): app renders patient gaps for launch context`
- `Integration: app requests scoped to launch patient only`

#### 10.2 — Security hardening & NIST 800-66r2 mapping

**What**: Encryption at rest/in transit, secrets via Vault/KMS, dependency/security scanning gates, and a control map to NIST SP 800-66r2 / CSF 2.0.

**Design**: TLS everywhere; Postgres TDE/disk encryption; `credentials_ref` resolves to Vault; bandit + pip-audit fail CI on findings; `SECURITY.md` mapping controls to HIPAA Security Rule safeguards (Govern/Identify/Protect/Detect/Respond/Recover).

**Testing**:
- `Integration: source credentials never persisted in plaintext (only credentials_ref)`
- `CI: bandit/pip-audit gates fail build on known-vuln dependency`
- `Integration: audit_log retains >= phi_log_retention_days, purge job respects floor`

#### 10.3 — Self-host packaging & docs

**What**: One-command `docker compose up` bringing the full stack online with seeded Synthea demo data, plus operator docs.

**Design**: Compose profiles (`demo` seeds Synthea + reference measures; `prod` expects external Postgres/S3). Make targets: `make demo`, `make migrate`, `make import-measures`. Quickstart in README.

**Testing**:
- `E2E: make demo → dashboards reachable with seeded cohort + computed measures within 5 min`
- `Integration: prod profile starts with external DSNs, no demo seed`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Schema, Tenancy        ─── required by everything
    │
Phase 2: Auth, RBAC, SMART                   ─── requires P1
    │
Phase 3: FHIR Bulk Ingestion                 ─── requires P1, P2 (back-end auth)
    │
Phase 4: Master Patient Index                ─── requires P3
    │
Phase 5: CQL Measure Engine                  ─── requires P3, P4
    ├── Phase 6: Risk Stratification & HCC    ─── requires P5; parallel with P7
    └── Phase 7: Care Gaps & Workflow         ─── requires P5; parallel with P6
         │
Phase 8: Clinical NLP                        ─── requires P3 (data) + P5 (feeds measures); can start after P3, integrates at P5
    │
Phase 9: Dashboards, Benchmarking, Gen-AI    ─── requires P5, P6, P7 (P9.3/9.4 need P6/P8)
    │
Phase 10: SMART App, Hardening, Packaging    ─── requires P2 (launch), P6, P7, P9
```

**Parallelism opportunities**:
- Phases 6 and 7 can be developed concurrently once Phase 5 lands.
- Phase 8 (NLP) extraction (8.1) can begin after Phase 3 and only needs Phase 5 for the measure-integration tests.
- Frontend (9.1) can be scaffolded against the OpenAPI schema as soon as Phase 5 endpoints exist, in parallel with 6/7.
- The `cql-engine-svc` sidecar (5.1) and `measures/` content (5.2) can be built in parallel with Phase 4.

---

## Definition of Done (per phase)

A phase is complete only when:

1. All tasks implemented.
2. All unit and integration tests pass (`pytest`), including testcontainers Postgres/Redis tests.
3. `ruff` lint + format clean; `mypy --strict` passes.
4. `bandit` and `pip-audit` security gates pass with no unresolved findings.
5. Docker images build; `docker compose up` brings the affected services up healthy.
6. The phase's feature works end-to-end against Synthea fixture data.
7. New config options documented in `.env.example` and README.
8. New API endpoints appear in the auto-generated **OpenAPI 3.1** spec; the CI OpenAPI diff is reviewed.
9. Alembic migrations created, and `downgrade`→`upgrade` round-trips cleanly.
10. Every PHI-touching path writes a correct `audit_log` entry (`phi_accessed` set appropriately).
11. RLS tenant isolation verified for any new tenant-scoped table or query.
12. Any new AI/ML output is explainable (confidence + evidence) and, where clinical, defaults to human-in-the-loop (`status="pending"`, never auto-applied).
```
