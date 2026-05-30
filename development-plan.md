# Digital Twin Platform — Phased Development Plan

> Project: 483-digital-twin-platform · Created: 2026-05-31
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesizes `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` documents into a concrete, buildable specification for an open-source digital twin platform: a continuously synchronised 3D virtual replica of physical assets that ingests live IoT sensor streams, renders them against 3D geometry, and supports historical playback, anomaly detection, simulation, and maintenance workflows.

**Core value proposition.** A single open-source product spanning high-fidelity web 3D visualization, real-time IoT ingest (MQTT/OPC-UA/REST), historical time-series playback, explainable anomaly detection, pluggable simulation, and maintenance workflows — accessible to SMEs locked out by Siemens/PTC/Ansys/IBM enterprise pricing, and free of the Azure/AWS vendor lock-in that fragments the open ecosystem.

**Primary personas.** Operations & maintenance engineers (manufacturing, energy, infrastructure), facility managers (smart buildings), and integration developers who connect sensors and build dashboards.

**Key differentiators (AI-native).** LLM-assisted sensor-tag → ontology mapping (the most common deployment failure point); natural-language spatial query ("show me all pumps above 80% capacity near the cooling tower"); explainable causal attribution for anomalies (not a black box); an MCP server exposing twin state to AI agents.

**Deployment model.** Self-hosted first (Docker Compose for SME single-node; Kubernetes/Helm for scale), with an optional edge agent for local aggregation and offline operation. Cloud/hybrid is an additive deployment topology, not a separate codebase.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary backend language | **Python 3.12** | The MVP-and-beyond scope is ML-heavy (anomaly detection, LLM ontology mapping, FMU simulation via `fmpy`, `scikit-learn`). Python has the richest ecosystem for OPC-UA (`asyncua`), MQTT (`aiomqtt`/`paho`), FMI (`fmpy`), and LLM SDKs (`anthropic`). A single language for ingest, analytics, and API minimises the deployment surface for SMEs. |
| API framework | **FastAPI** | Async-native (essential for fan-in of many concurrent sensor connectors), generates OpenAPI 3.1 automatically (standards.md mandates OAS 3.1 for SDK generation), and Pydantic v2 gives free request/response validation that also defines the data contracts in this plan. |
| Metadata / config store | **PostgreSQL 16** (relational + JSONB) | Implements **data-model-suggestion-3** (hybrid relational+JSONB). Stable relational backbone for tenants/users/assets/sensors with foreign keys, plus JSONB for heterogeneous per-asset-type properties, connector configs, and anomaly explanations — avoiding a sparse column explosion across pumps/HVAC/motors. |
| Time-series store | **TimescaleDB** (PostgreSQL 16 extension) | Implements the telemetry layer of **data-model-suggestion-4**. Hypertables + columnar compression + continuous aggregates give industrial-scale ingest and fast playback/dashboard rollups, while staying PostgreSQL-compatible so a single-node SME deployment runs *one* database engine (Postgres with both extensions). |
| Hierarchy traversal | **Materialised `ancestor_path UUID[]` + GIN** | From suggestion 3. Subtree queries via `'X' = ANY(ancestor_path)` without a separate closure table or a graph engine. Neo4j (suggestion 4) is deferred — its operational cost is unjustified for the MVP; graph queries are added later as an optional projection (Phase 11). |
| Stream decoupling / async work | **Redis 7** (Streams + RQ-style queue) | Decouples high-frequency ingest from DB writes and powers async jobs (model file processing, anomaly scoring, simulation runs, alert dispatch). Redis is far lighter to operate than Kafka for SME single-node; Kafka becomes an optional connector at scale (Phase 4). |
| Real-time push to UI | **WebSocket (FastAPI) + Redis Pub/Sub** | Live state sync to the 3D viewer. Redis Pub/Sub fans out sensor-update and anomaly events to all connected WebSocket sessions across API replicas. |
| Frontend | **React 18 + TypeScript + Vite** | Industry-standard SPA tooling; pairs with the auto-generated TypeScript SDK. |
| 3D rendering | **Three.js** (`react-three-fiber`) + **3D Tiles** (`3d-tiles-renderer`) | README/research mandate WebGL with instanced meshes, frustum culling, and OGC 3D Tiles 1.1 tile streaming for industrial-scale scenes. Three.js is the practical baseline with the largest ecosystem. |
| 3D ingest pipeline | **IfcOpenShell** (IFC), **`pygltflib`/`trimesh`** (glTF/OBJ), **py3dtiles** (tiling) | Convert IFC/CAD/point-cloud inputs to glTF + 3D Tiles tilesets for the web viewer. OpenUSD ingest is deferred (Phase 11, optional Omniverse bridge). |
| LLM provider | **Anthropic Claude (via `anthropic` SDK), provider-abstracted** | Powers ontology mapping, NL query, and anomaly explanation. Abstracted behind an `LLMClient` interface so self-hosters can point at a local model. Prompt caching enabled on system prompts (ontology schemas are large and reused). |
| Auth | **OAuth 2.0 + OIDC, JWT (RFC 7519)** via `authlib` + `python-jose` | standards.md requires OAuth2/OIDC for enterprise SSO and JWT as the token format. Local password auth (argon2) for SME single-node. |
| Edge agent | **Python, packaged as a standalone wheel + Docker image** | Aggregates/filters/buffers locally, syncs upstream over authenticated REST/MQTT with store-and-forward for offline operation. |
| Simulation | **`fmpy`** (FMI 2.0/3.0 FMU) + Python-callable plugin interface | standards.md/features.md require simulation-engine-agnostic deployment via the FMI/FMU standard; a Python entry-point plugin covers data-driven surrogate models. |
| Containerisation | **Docker + Docker Compose**; **Helm chart** | Self-hosted first. Compose for SME single-node; Helm for K8s scale-out. |
| Testing | **pytest** + **pytest-asyncio** + **testcontainers** (backend); **Vitest** + **Playwright** (frontend) | testcontainers spins real Postgres/TimescaleDB/Redis for integration tests; Playwright drives the 3D viewer e2e. |
| Code quality | **ruff** (lint+format), **mypy --strict**, **pre-commit**; **ESLint + Prettier** (frontend) | Enforced in CI. |
| Package mgmt | **uv** (Python), **pnpm** (frontend) | Fast, lockfile-based, reproducible. |
| Migrations | **Alembic** | Versioned schema migrations for Postgres/TimescaleDB. |
| SDKs | **Python SDK** (hand-thin over OpenAPI), **TypeScript SDK** (generated via `openapi-typescript-codegen`) | README/features mandate Python + JS SDKs. |

### Project Structure

```
digital-twin-platform/
├── pyproject.toml                  # uv-managed; workspace root
├── docker-compose.yml              # postgres+timescale, redis, api, worker, web
├── docker-compose.edge.yml         # edge agent standalone
├── Dockerfile                      # multi-stage: api / worker
├── alembic.ini
├── helm/                           # Kubernetes chart
│   └── digital-twin-platform/
├── migrations/                     # Alembic versions (sql + timescale hypertables)
├── packages/
│   ├── dtp-core/                   # domain models, no I/O
│   │   └── src/dtp_core/
│   │       ├── models/             # Pydantic + SQLAlchemy ORM
│   │       ├── ontology/           # DTDL/AAS/RealEstateCore parsing
│   │       └── schemas/            # API request/response Pydantic schemas
│   ├── dtp-api/                    # FastAPI app
│   │   └── src/dtp_api/
│   │       ├── main.py
│   │       ├── deps.py             # auth, db session, tenant context
│   │       ├── routers/            # assets, sensors, telemetry, anomalies,
│   │       │                       #   simulations, maintenance, dashboards,
│   │       │                       #   connectors, query, auth, mcp
│   │       ├── ws/                 # WebSocket hub
│   │       └── security/           # OAuth2/OIDC/JWT, RBAC enforcement
│   ├── dtp-ingest/                 # connector framework + protocol adapters
│   │   └── src/dtp_ingest/
│   │       ├── connectors/         # mqtt, opcua, rest, kafka, modbus
│   │       ├── pipeline.py         # normalise → quality-check → persist → publish
│   │       └── mapping/            # sensor-tag → ontology mapping (rule + LLM)
│   ├── dtp-worker/                 # async jobs (RQ): model proc, anomaly, sim, alerts
│   │   └── src/dtp_worker/
│   ├── dtp-3d/                     # 3D ingest/tiling (IFC→glTF→3D Tiles)
│   │   └── src/dtp_3d/
│   ├── dtp-anomaly/                # detectors + explainers
│   ├── dtp-sim/                    # FMU + python plugin runners
│   ├── dtp-ai/                     # LLMClient, prompts, NL query, MCP server
│   └── dtp-edge/                   # edge agent
├── sdks/
│   ├── python/                     # dtp_client
│   └── typescript/                 # generated client
├── web/                            # React + Vite SPA
│   └── src/
│       ├── viewer/                 # react-three-fiber 3D viewport + 3D Tiles
│       ├── dashboard/              # no-code dashboard builder + widgets
│       ├── api/                    # generated TS SDK wrapper
│       └── pages/
├── tests/
│   ├── unit/
│   ├── integration/                # testcontainers
│   ├── e2e/                        # Playwright
│   └── fixtures/                   # sample IFC, MQTT payloads, FMUs, DTDL models
└── docs/
```

---

## Phase 1: Foundation — Project Scaffold, Data Layer, Auth

### Purpose
Establish the repository, the dual-extension PostgreSQL schema (relational+JSONB metadata and TimescaleDB telemetry), multi-tenant identity with OAuth2/OIDC and JWT, and a runnable FastAPI app with health checks and auto-generated OpenAPI 3.1. After this phase the platform boots, authenticates users, and persists tenants — everything else builds on it.

### Tasks

#### 1.1 — Repository scaffold, tooling, CI

**What**: Create the uv workspace, Docker Compose stack, linting/type-checking, and CI pipeline.

**Design**:
- `pyproject.toml` declares workspace members (`packages/*`) and dev tools: `ruff`, `mypy`, `pytest`, `pytest-asyncio`, `testcontainers`, `alembic`, `pre-commit`.
- `docker-compose.yml` services:
  - `db`: `timescale/timescaledb:2.x-pg16` (Postgres + TimescaleDB in one image), volume `pgdata`, env `POSTGRES_DB=dtp`.
  - `redis`: `redis:7-alpine`.
  - `api`: built from `Dockerfile` target `api`, `command: uvicorn dtp_api.main:app`, depends_on db+redis.
  - `worker`: target `worker`, `command: rq worker -u redis://redis:6379 default ingest anomaly sim alerts`.
  - `web`: Vite dev/preview server.
- Config via `pydantic-settings` `Settings` (env-prefixed `DTP_`): `database_url`, `redis_url`, `jwt_secret`, `oidc_issuer`, `oidc_client_id`, `llm_provider`, `anthropic_api_key`, `storage_backend` (`local|s3`), `storage_url`.
- CI (GitHub Actions): `ruff check`, `ruff format --check`, `mypy --strict packages`, `pytest` (with testcontainers), `pnpm -C web build`, `docker build`.

**Testing**:
- `Unit: Settings loads from env with DTP_ prefix → fields populated, defaults applied`
- `Unit: Settings missing DTP_JWT_SECRET → ValidationError naming the field`
- `Integration: docker compose up → GET /healthz returns 200 {"status":"ok","db":"up","redis":"up"}`
- CI smoke: `ruff`, `mypy`, `pytest -q` all green on empty scaffold.

#### 1.2 — Database schema & migrations (metadata + telemetry)

**What**: Alembic migrations creating the relational+JSONB metadata schema and the TimescaleDB telemetry hypertables.

**Design**:
Adopt **data-model-suggestion-3** verbatim for metadata tables (`tenants`, `users`, `roles`, `user_roles`, `asset_types`, `assets` with `properties`/`ontology_refs`/`geo_location` JSONB and `ancestor_path UUID[]`, `model_files`, `sensors`, `data_connectors`, `sensor_connector_map`, `simulation_definitions`, `simulation_runs`, `anomaly_models`, `anomaly_events` with JSONB `explanation`, `maintenance_tickets`, `digital_thread_links`, `alert_channels`, `alert_rules`, `dashboards`). Adopt the **suggestion-4 TimescaleDB** block for telemetry:

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;
CREATE TABLE sensor_readings (
    time TIMESTAMPTZ NOT NULL, sensor_id UUID NOT NULL,
    value_numeric DOUBLE PRECISION, value_string VARCHAR(255),
    quality SMALLINT NOT NULL DEFAULT 0,        -- 0 good,1 uncertain,2 bad,3 stale
    PRIMARY KEY (sensor_id, time));
SELECT create_hypertable('sensor_readings','time',chunk_time_interval=>INTERVAL '1 day');
ALTER TABLE sensor_readings SET (timescaledb.compress,
    timescaledb.compress_segmentby='sensor_id', timescaledb.compress_orderby='time DESC');
SELECT add_compression_policy('sensor_readings', INTERVAL '7 days');
```
Plus the `sensor_readings_1min` / `_1hr` / `_1day` continuous aggregates from suggestion 4. Enable Postgres **Row-Level Security** on every tenant-scoped table:
```sql
ALTER TABLE assets ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON assets
  USING (tenant_id = current_setting('dtp.tenant_id')::uuid);
```
The API sets `SET LOCAL dtp.tenant_id = '<uuid>'` per request transaction (see 1.4). An `assets` BEFORE INSERT/UPDATE trigger maintains `ancestor_path` from `parent_asset_id`.

**Testing**:
- `Integration (testcontainers): run all migrations up then down → no errors, schema empty after down`
- `Integration: insert asset with parent → trigger sets ancestor_path = parent.ancestor_path || parent.asset_id`
- `Integration: query 'SELECT * FROM assets WHERE :root = ANY(ancestor_path)' → returns full subtree`
- `Integration: RLS — set dtp.tenant_id=A, insert; set dtp.tenant_id=B, SELECT → 0 rows`
- `Integration: insert 10k sensor_readings, run compression policy → chunk compressed, count preserved`

#### 1.3 — ORM models & repositories

**What**: SQLAlchemy 2.0 (async) ORM mappings and repository classes mirroring the schema.

**Design**:
- `dtp_core.models` ORM classes for every table; JSONB columns typed `Mapped[dict]`.
- `BaseRepository[T]` with `get(id)`, `list(filters)`, `create(obj)`, `update(id, patch)`, `delete(id)` — all tenant-scoped via the session's `dtp.tenant_id`.
- `AssetRepository.subtree(root_id) -> list[Asset]` using `ancestor_path`.
- Pydantic v2 schemas in `dtp_core.schemas` for API I/O, separate from ORM (e.g. `AssetCreate`, `AssetRead`, `AssetUpdate`).

```python
class AssetCreate(BaseModel):
    asset_type_id: UUID
    parent_asset_id: UUID | None = None
    name: str
    serial_number: str | None = None
    properties: dict = {}          # validated against asset_type.property_schema
    ontology_refs: dict = {}        # {iso_81346, dtdl_id, aas_id}
    geo_location: GeoLocation | None = None
```

**Testing**:
- `Unit: AssetCreate rejects unknown asset_type → handled at service layer (FK), schema accepts`
- `Integration: AssetRepository.create then get → round-trips JSONB properties intact`
- `Integration: AssetRepository.subtree(root) → ordered list incl. descendants only`

#### 1.4 — Identity, OAuth2/OIDC, JWT, tenant context

**What**: Authentication (local password + OIDC), JWT issuance/validation, and per-request tenant context binding.

**Design**:
- Local auth: argon2id password hashing; `POST /auth/login` → `{access_token, refresh_token, token_type:"bearer"}`. JWT (RFC 7519) claims: `sub` (user_id), `tid` (tenant_id), `scopes`, `exp`, `iss`.
- OIDC: `authlib` Authorization Code + PKCE against configured `oidc_issuer`; map IdP subject → local user (auto-provision in tenant if allowed).
- `get_current_principal()` FastAPI dependency: validate Bearer JWT (or `api_keys.key_hash` for machine clients/edge agents), return `Principal(user_id, tenant_id, scopes)`.
- DB session dependency executes `SET LOCAL dtp.tenant_id = :tid` from the principal at transaction start, activating RLS.
- API-key auth path for edge agents and SDKs (`X-API-Key` header → `api_keys` lookup, scope-limited).

**Testing**:
- `Unit: hash+verify password → verify true for correct, false for wrong`
- `Unit: JWT encode/decode round-trips tid+scopes; expired token → 401`
- `Integration (mocked OIDC): callback with valid code → user provisioned, JWT issued`
- `Integration: request with token for tenant A cannot read tenant B asset → 404 (RLS yields no row)`
- `Integration: missing/invalid token → 401`

---

## Phase 2: Asset Hierarchy, Ontology & 3D Model Management

### Purpose
Deliver the asset domain: CRUD for asset types and assets, the fleet/site/zone/equipment/component/sensor hierarchy, ontology references (DTDL/AAS/RealEstateCore/ISO 81346), and 3D model file upload with a tiling pipeline that produces web-renderable 3D Tiles. After this phase a user can model their facility and attach geometry — the spatial substrate the twin renders against.

### Tasks

#### 2.1 — Asset type & asset CRUD with hierarchy and RBAC

**What**: REST endpoints for asset types and assets, with subtree-scoped RBAC.

**Design**:
Endpoints (all tenant-scoped, RBAC-enforced):
- `POST /asset-types`, `GET /asset-types`, `GET/PUT/DELETE /asset-types/{id}` — `property_schema` (JSON-Schema-like) defines per-type properties.
- `POST /assets`, `GET /assets?type=&status=&parent=&q=`, `GET/PUT/DELETE /assets/{id}`.
- `GET /assets/{id}/subtree`, `GET /assets/{id}/ancestors`.
- On create, validate `properties` against the referenced `asset_type.property_schema` (jsonschema lib); 422 on mismatch.
- RBAC: `roles.permissions` JSONB `[{resource, actions, scope:"tenant"|"subtree", asset_id}]`. A `requires(resource, action)` dependency resolves the principal's roles, and for `scope:"subtree"` checks the target asset's `ancestor_path` contains the granted `asset_id` (guards OWASP API1 BOLA).

**Testing**:
- `Unit: properties valid against property_schema → accepted; missing required → 422 naming field`
- `Integration: create site→zone→equipment chain → ancestor_path correct at each level`
- `Integration: user with subtree-scoped role on Site A reads asset under Site A → 200; under Site B → 403`
- `Integration: DELETE asset with children → 409 (must decommission/reparent first)`

#### 2.2 — Ontology layer (DTDL / AAS / RealEstateCore / ISO 81346)

**What**: Parse and store standard ontology models; expose them for asset-type binding and validation.

**Design**:
- `dtp_core.ontology` loads **DTDL v3** interfaces (JSON-LD), **RealEstateCore** DTDL building ontology, and **AAS (IEC 63278)** submodels via `basyx`/`aas-core` if available else a thin parser. ISO 81346 reference designations validated by regex/grammar for `=`(function) `+`(location) `-`(product) aspect prefixes.
- `OntologyClass` records (iri, label, standard, parent_iri) loaded into the metadata DB; `asset_types.property_schema.dtdl_model_id` links a type to a DTDL interface; properties derive defaults/units from the interface.
- `GET /ontology/classes?standard=dtdl|aas|rec|iso81346`, `GET /ontology/classes/{iri}`.

**Testing**:
- `Unit: parse a DTDL v3 Interface fixture → properties/telemetry/relationships extracted`
- `Unit: valid ISO 81346 '=A1.M01' parses; '!!bad' → ValidationError`
- `Fixture: load RealEstateCore building ontology → expected class count, subclass edges present`

#### 2.3 — 3D model upload & storage

**What**: Upload endpoint and storage abstraction for model files with checksum + dedupe.

**Design**:
- `POST /assets/{id}/models` multipart upload → store to `storage_backend` (local FS or S3), compute SHA-256, insert `model_files` row (`format`, `file_size_bytes`, `checksum_sha256`, `lod_level`, `spatial_anchor` JSONB transform). Enqueue `process_model` job (2.4).
- Accept `ifc, gltf, glb, obj, las, laz, e57`; reject others with 415.
- `GET /assets/{id}/models`, `GET /models/{id}` (returns processed tileset URL when ready).

**Testing**:
- `Integration: upload sample.glb → 201, row created, SHA-256 matches, job enqueued`
- `Integration: upload .exe → 415`
- `Integration: re-upload identical file (same checksum) → dedup, no duplicate storage write`

#### 2.4 — Tiling pipeline (IFC/glTF → 3D Tiles)

**What**: Async worker job converting uploaded geometry to OGC 3D Tiles 1.1 tilesets and generating LOD/collision meshes.

**Design**:
- `process_model(model_file_id)` worker:
  1. IFC → glTF via IfcOpenShell; OBJ/glTF normalised via `trimesh`.
  2. Generate simplified LOD meshes (quadric decimation) at lod 0–3.
  3. Produce a **3D Tiles 1.1** tileset (`tileset.json` + `.glb` content) via `py3dtiles`; compute bounding box.
  4. Write `processing_meta` JSONB (`tile_set_url`, `mesh_count`, `vertex_count`, `bounding_box`, `collision_mesh_url`); set status ready.
- Errors recorded on the row; job retried up to 3× with backoff.

**Testing**:
- `Integration: process sample.ifc fixture → tileset.json produced, bounding_box populated, status=ready`
- `Unit: LOD decimation reduces vertex_count monotonically lod0>lod1>lod2`
- `Integration: corrupt IFC → status=error, message recorded, no partial tileset served`

---

## Phase 3: IoT Ingest & Real-Time State Synchronisation

### Purpose
The heart of the product: connect to live sensor sources, normalise heterogeneous payloads to sensor records, persist telemetry to TimescaleDB, and push live state to subscribers over WebSocket. After this phase the twin is *live* — sensor values flow in and update connected clients in real time.

### Tasks

#### 3.1 — Connector framework & sensor provisioning

**What**: Pluggable connector abstraction plus sensor CRUD and connector–sensor mapping.

**Design**:
- `Connector` ABC: `async def connect()`, `async def stream() -> AsyncIterator[RawMessage]`, `async def health() -> ConnectorHealth`, `async def close()`. Registered by `protocol`.
- `RawMessage(source_path: str, payload: dict|bytes, received_at: datetime)`.
- `data_connectors.connection_config` JSONB carries protocol-specific config (per suggestion 3 examples: MQTT broker/topics, OPC-UA endpoint/node_ids/security_mode, Kafka bootstrap/topics).
- Endpoints: `POST /connectors`, `GET /connectors`, `POST /connectors/{id}:test`, `POST /sensors`, `POST /sensors/{id}/bindings` (maps to connector + `source_path` + `transform_rules` JSONB with JMESPath value/timestamp extraction).
- Connector lifecycle managed by a supervisor in `dtp-worker`; status + `health_meta` (messages/sec, error_count_24h) updated on `data_connectors`.

**Testing**:
- `Unit: register fake connector by protocol → factory returns instance`
- `Unit: JMESPath transform_rules extract value+timestamp from nested payload`
- `Integration: POST /connectors:test with unreachable broker → status error + message`

#### 3.2 — MQTT, OPC-UA, REST connectors

**What**: Concrete adapters for the three MVP protocols (MQTT v5, OPC-UA/IEC 62541, REST webhook).

**Design**:
- **MQTT** (`aiomqtt`): subscribe to configured topics (incl. wildcards), MQTT v5 shared subscriptions for horizontal scale; TLS + X.509 client cert support (standards.md device auth).
- **OPC-UA** (`asyncua`): connect with `security_mode` (None/Sign/SignAndEncrypt), subscribe to `node_ids`, map NodeId→`source_path`. X.509 cert auth.
- **REST webhook**: `POST /ingest/{connector_id}` authenticated by per-connector HMAC signature header; body mapped via `transform_rules`. Returns 202 on accept.
- Each adapter yields `RawMessage`; bad-signature webhook → 401, no persist (OWASP API2).

**Testing**:
- `Integration (mosquitto testcontainer): publish to topic → connector yields RawMessage with payload`
- `Integration (asyncua test server): subscribe node → value change yields RawMessage`
- `Integration: webhook with valid HMAC → 202 + reading persisted; invalid HMAC → 401, nothing persisted`

#### 3.3 — Normalisation pipeline & telemetry persistence

**What**: The ingest pipeline: map raw → sensor reading, apply quality rules, persist to TimescaleDB, publish live update.

**Design**:
Pipeline stages (per message): `resolve sensor (by connector+source_path) → extract value/ts via transform_rules → apply transform (scale/offset/clamp) → quality check → buffer → batch-insert sensor_readings → publish to Redis channel dtp:{tenant}:sensor:{sensor_id}`.
- Quality (`sensors.mapping_config.quality_rules`): `quality=3 stale` if older than `stale_after_ms`; `quality=2 bad` if outside `[min_value,max_value]`; else `0 good`. Bad/stale still stored but flagged and **not** propagated to visual state.
- Batched writes (configurable `batch_size`, `flush_ms`) via Redis Stream buffer → bulk `COPY`/`executemany` into `sensor_readings` for throughput.

**Testing**:
- `Unit: value outside max_value → quality=2; older than stale_after_ms → quality=3`
- `Unit: transform scale/offset applied (e.g. K→°C offset -273.15)`
- `Integration: 1000 messages → 1000 sensor_readings rows, batched in ≤ N inserts`
- `Integration: good reading → message published on dtp:{tenant}:sensor:{id}`

#### 3.4 — WebSocket live state hub

**What**: WebSocket endpoint streaming live sensor/anomaly state to the 3D viewer, fed by Redis Pub/Sub.

**Design**:
- `WS /ws?token=<jwt>` authenticates, then client sends `{action:"subscribe", asset_ids:[...], sensor_ids:[...]}`.
- Server subscribes to matching Redis channels; forwards `{type:"sensor_update", sensor_id, value, quality, ts}` and `{type:"anomaly", ...}` frames.
- Per-connection subscription state; auto-resubscribe on asset subtree changes; heartbeat ping/pong; backpressure via per-client send queue with drop-oldest for slow consumers.

**Testing**:
- `Integration: connect, subscribe sensor S, publish reading → client receives sensor_update frame`
- `Integration: connect without token → closed 4401`
- `Integration: subscribe to asset not in tenant → ignored, no frames`

---

## Phase 4: Historical Playback & Time-Series Query API

### Purpose
Expose the time-series store for charting, correlation, and the README's signature capability — scrubbing backward in time to replay any past asset state. After this phase users can query history, view trends, and replay the twin's visual state at any timestamp.

### Tasks

#### 4.1 — Time-series query API

**What**: Endpoints to query raw and aggregated sensor history.

**Design**:
- `GET /telemetry?sensor_ids=&from=&to=&agg=raw|1min|1hr|1day&fn=avg|min|max|last` → routes to `sensor_readings` or the matching continuous aggregate; auto-selects aggregate when range × sensors exceeds a point budget (downsampling).
- `GET /telemetry/latest?sensor_ids=` → latest value per sensor (from `sensor_readings_1min` `last`).
- Response: `{sensor_id, points:[{t, v, q}], aggregation}`. Enforces tenant + asset RBAC on each sensor.
- Optional `format=csv` export.

**Testing**:
- `Integration: query 30-day range → server selects 1hr aggregate automatically`
- `Integration: raw query within budget → raw points returned`
- `Integration: CSV export → valid CSV header + rows`
- `Integration: sensor outside RBAC scope → 403`

#### 4.2 — State-at-time (playback) endpoint

**What**: Reconstruct the visual state of an asset subtree at an arbitrary past timestamp.

**Design**:
- `GET /assets/{id}/state-at?ts=<iso>&depth=N` → for each sensor under the subtree, the reading at-or-before `ts` (TimescaleDB `last(value,time)` with `time <= ts`), plus the asset metadata as it stood (assets are slowly-changing; current row is acceptable for MVP). Returns `{ts, sensors:[{sensor_id, value, quality, ts}], assets:[...]}` for the viewer to colour-code.
- Playback in UI scrubs by repeatedly calling this (or a batched `/state-range` returning keyframes at an interval).

**Testing**:
- `Integration: readings at t1<t2<t3, query state-at(t2) → returns t2's value (≤ ts) per sensor`
- `Integration: ts before any reading → sensor value null, quality unknown`
- `Integration: depth limits subtree returned`

#### 4.3 — Maintenance/anomaly correlation overlay

**What**: Return maintenance events and anomalies aligned to a sensor timeline for correlation charts.

**Design**:
- `GET /assets/{id}/timeline?from=&to=` → merges `anomaly_events`, `maintenance_tickets` state changes, and `maintenance_events` (TimescaleDB) into a unified, time-ordered event list keyed by asset, for overlaying markers on time-series charts.

**Testing**:
- `Integration: anomaly + ticket in range → both appear ordered by time with type tags`
- `Integration: empty range → empty list, 200`

---

## Phase 5: Web Frontend — 3D Viewer & Dashboard Builder

### Purpose
Deliver the operator-facing SPA: a WebGL 3D viewport rendering the asset tilesets with live colour-coded state, time scrubbing, and a no-code dashboard builder. After this phase non-technical operators can see and interact with their twin. Can be developed in parallel with Phases 3–4 against mocked APIs.

### Tasks

#### 5.1 — App shell, auth, generated TS SDK

**What**: React app shell with login, tenant context, routing, and the generated TypeScript SDK.

**Design**:
- Generate `sdks/typescript` from the live `/openapi.json` via `openapi-typescript-codegen`; wrap in `web/src/api` with token injection + refresh.
- Routes: `/login`, `/assets`, `/twin/:assetId`, `/dashboards`, `/dashboards/:id`, `/connectors`, `/anomalies`, `/maintenance`, `/admin`.
- Auth context stores JWT, handles OIDC redirect callback.

**Testing**:
- `Unit (Vitest): SDK wrapper attaches Authorization header; 401 → triggers refresh`
- `E2E (Playwright): login → redirected to /assets, token persisted`

#### 5.2 — 3D viewport with 3D Tiles streaming + live state

**What**: `react-three-fiber` viewport that streams the asset tileset and colour-codes components from live WebSocket data.

**Design**:
- Load `tileset.json` via `3d-tiles-renderer`; instanced meshes + frustum culling for scale.
- Subscribe via WS to the asset subtree's sensors; map sensor→mesh by `spatial_anchor`/component id; colour ramp by value vs. thresholds (green/amber/red), animate flagged components.
- Camera presets, click-to-select component → side panel shows live metrics + digital-thread links.
- Stale/bad-quality (quality≥2) rendered greyed, never as a live value.

**Testing**:
- `E2E (Playwright): open /twin/:id → tileset renders (canvas non-empty), WS connected`
- `E2E: simulate sensor_update over WS → component colour changes`
- `Unit: value→colour mapping (below/within/above thresholds)`

#### 5.3 — Time scrubber & history charts

**What**: A timeline scrubber driving `/state-at` and time-series charts with anomaly/maintenance overlays.

**Design**:
- Scrubber bar with play/pause/speed; on scrub, debounce calls to `/assets/{id}/state-at` and recolour the 3D scene to the historical state.
- Chart panel (uPlot) renders `/telemetry` series with `/timeline` markers.

**Testing**:
- `E2E: scrub to past timestamp → scene recolours to historical values`
- `E2E: chart shows series + anomaly marker at correct time`

#### 5.4 — No-code dashboard builder

**What**: Drag-and-drop grid dashboard with the widget catalogue persisted to `dashboards.widgets` JSONB.

**Design**:
- Grid layout (react-grid-layout); widget types: `3d_viewport`, `time_series_chart`, `gauge`, `table`, `alert_feed`, `kpi_card`, `map` (matching suggestion-3 `dashboards.widgets` shape).
- `GET/POST/PUT/DELETE /dashboards`; layout+widgets stored as JSONB; `is_public` toggles tenant-wide visibility.

**Testing**:
- `Unit: widget config schema validates per type`
- `E2E: add gauge widget, save, reload → widget persists with config`

---

## Phase 6: Alerting & Maintenance Workflows

### Purpose
Close the operational loop: threshold/anomaly alerts routed to channels, and maintenance tickets linked to assets and the digital thread. After this phase the platform drives action, not just observation — satisfying the MVP "threshold alerting with webhook/email" and the digital-thread should-have.

### Tasks

#### 6.1 — Threshold rules & alert routing

**What**: Rule evaluation on incoming readings and dispatch to alert channels.

**Design**:
- `alert_rules.filter_criteria` JSONB (severity, asset_scope_id, include_subtree, asset_types) + threshold rules on sensors. During ingest (3.3), threshold breach raises an event → `dispatch_alert` worker job.
- Channels (`alert_channels.config` JSONB): `email` (SMTP), `webhook` (HMAC-signed POST), `slack`, `pagerduty`, `scada`, `sms`. Dispatch deduplicates/debounces (no alert storms) and records delivery status.
- `POST/GET /alert-channels`, `POST/GET /alert-rules`, `POST /alert-channels/{id}:test`.

**Testing**:
- `Unit: reading above threshold within rule scope → alert event raised; outside scope → none`
- `Integration (mock SMTP): critical alert → email sent to configured recipients`
- `Integration: webhook channel → signed POST delivered; failure → retried, status=failed after N`
- `Unit: debounce — repeated breaches within window → single alert`

#### 6.2 — Maintenance tickets & digital thread

**What**: Ticket CRUD linked to assets/anomalies, plus digital-thread document links.

**Design**:
- `POST/GET/PUT /maintenance-tickets` with lifecycle `open→assigned→in_progress→resolved→closed`; `anomaly_event_id` optional link; `external_refs` JSONB (CMMS/SCADA/SAP), `resolution` JSONB (notes, parts_used, labour_hours, root_cause).
- `POST/GET/DELETE /assets/{id}/thread` for `digital_thread_links` (procurement/maintenance/compliance/manual/drawing/photo).
- Optional outbound connector to one CMMS (webhook-based) — generic webhook is MVP; named CMMS adapters are backlog.

**Testing**:
- `Integration: create ticket from anomaly → linked, status=open`
- `Unit: illegal transition (closed→open) → 409`
- `Integration: add thread link → appears in /assets/{id}/thread and on 3D component panel`

---

## Phase 7: Anomaly Detection with Explainable Causal Attribution

### Purpose
Deliver the differentiating AI-native capability: ML models trained on historical telemetry that surface deviations and **explain** them via causal attribution to specific components — not opaque alerts. After this phase the twin is predictive and self-explaining.

### Tasks

#### 7.1 — Detector framework & training

**What**: Pluggable detectors trained on TimescaleDB history, with artifacts persisted.

**Design**:
- `AnomalyDetector` ABC: `fit(readings_df, config)`, `score(window) -> AnomalyScore`, `explain(window) -> Explanation`. Built-ins: `IsolationForest`, `seasonal-residual (STL) z-score`, multivariate `Mahalanobis`.
- `anomaly_models.training_config` JSONB (window_size, features, hyperparams, training_range); `model_artifact_url` stores the serialised model; `metrics` JSONB (auc/precision/recall/f1).
- `POST /anomaly-models`, `POST /anomaly-models/{id}:train` (worker job over the configured training range), `GET /anomaly-models/{id}`.

**Testing**:
- `Unit: IsolationForest fit on synthetic normal data, score injected spike → high score`
- `Integration: train job over fixture range → status=trained, artifact stored, metrics populated`
- `Unit: STL detector flags seasonal residual beyond z-threshold`

#### 7.2 — Online scoring & explainable events

**What**: Score live windows and emit `anomaly_events` with causal attribution.

**Design**:
- Scoring worker consumes the live stream per model's `target_sensors`; on anomaly, compute the `explanation` JSONB exactly per suggestion-3 shape: `causal_component_id/name`, `confidence`, `contributing_factors[{sensor_tag, deviation_sigma, direction}]`, `similar_past_events`, `recommended_action`.
- Causal attribution: rank contributing sensors by deviation magnitude × graph proximity to candidate components (component = the asset owning the top contributing sensor). `similar_past_events` via nearest-neighbour over past event feature vectors.
- Emits `anomaly_events` row + WS `anomaly` frame + triggers alert rules (Phase 6).

**Testing**:
- `Integration: inject correlated spike on bearing-vibration sensor → event with that component as causal, vib as top factor`
- `Unit: contributing_factors sorted by deviation_sigma desc`
- `Integration: anomaly event → WS frame emitted + alert dispatched`

#### 7.3 — LLM explanation enrichment

**What**: Use the LLM to turn the structured explanation into an operator-readable narrative and recommended action.

**Design**:
- `dtp_ai.LLMClient` (Anthropic, prompt-caching on the system prompt). Input: structured explanation + asset context + recent similar resolutions. Output: concise `narrative` + `recommended_action` written back into `explanation`.
- System prompt template (cached): role, safety framing ("describe likely cause and inspection steps; never assert certainty"), output schema. User prompt: the JSON explanation + similar past `resolution` notes.

**Testing**:
- `Unit (mocked LLM): explanation in → narrative+recommended_action merged into JSONB`
- `Unit: LLM failure → event still stored with structured explanation, narrative omitted (graceful degrade)`

---

## Phase 8: AI-Assisted Ontology Mapping & Natural Language Query

### Purpose
Attack the single biggest deployment failure point (sensor schema mapping) and deliver the NL spatial query interface. After this phase onboarding a new sensor source is largely automated and operators can query the twin in plain English.

### Tasks

#### 8.1 — LLM-assisted sensor-tag → ontology mapping

**What**: Suggest ontology mappings for raw sensor tags, with human confirmation.

**Design**:
- `POST /mapping/suggest` body `{connector_id, sample_tags:[...], sample_payloads:[...]}` → LLM matches each raw tag to candidate ontology IRIs (DTDL/RealEstateCore/AAS) loaded in Phase 2, returns `[{tag, suggested_iri, unit, confidence, rationale}]`. Rule-based matcher (string similarity + unit heuristics) runs first; LLM resolves the ambiguous remainder.
- Mappings are *suggestions*; applying writes `sensors.mapping_config.ontology_iri` only after explicit user/API confirmation (`POST /mapping/apply`).

**Testing**:
- `Unit (mocked LLM): tag 'PUMP_P101_TEMP' °C → suggests temperature IRI, high confidence`
- `Unit: rule-based exact-unit match resolves without LLM call`
- `Integration: apply mapping → sensors.mapping_config updated; suggest never mutates`

#### 8.2 — Natural language spatial/state query

**What**: Translate NL questions into safe structured queries over assets + telemetry.

**Design**:
- `POST /query/nl` body `{question}` → LLM (tool/function-calling, constrained) emits a structured query plan: target asset types, hierarchy scope (subtree of named asset), telemetry predicates (sensor role + comparator + value), spatial predicate (within radius of a named asset using `geo_location`). The platform **executes** the plan against the DB (never executes LLM-generated SQL directly — guards against injection). Returns matched assets + the interpreted plan for transparency.
- Example: "all pumps above 80% capacity near the cooling tower" → type=pump, telemetry capacity>80, spatial within R of asset matching "cooling tower".

**Testing**:
- `Unit (mocked LLM): example question → expected structured plan`
- `Integration: plan executes → returns only matching assets within tenant RBAC`
- `Security: LLM emitting raw SQL is impossible — only the fixed plan schema is executed`

#### 8.3 — MCP server

**What**: Expose twin state to AI agents via a Model Context Protocol server.

**Design**:
- `dtp_ai.mcp` MCP server with tools: `get_asset(id)`, `query_assets(plan)`, `get_telemetry(sensor_ids, range)`, `get_active_anomalies(asset_id)`, `get_state_at(asset_id, ts)`. Auth via API key → tenant scope; all calls RBAC-enforced. Mounted at `/mcp`.

**Testing**:
- `Integration (MCP client): list tools → expected catalogue`
- `Integration: get_active_anomalies via MCP → tenant-scoped results only`

---

## Phase 9: Simulation & What-If Analysis

### Purpose
Add pluggable simulation on the virtual twin (FMI/FMU + Python callables) with the safety guardrail the research mandates: simulation outputs are explicitly labelled predictive and require human acknowledgement before any action. After this phase users can run what-if scenarios without touching the physical asset.

### Tasks

#### 9.1 — Simulation backend plugin interface

**What**: Define and load FMU and Python simulation backends.

**Design**:
- `SimulationBackend` ABC: `load(config)`, `run(inputs, params) -> SimResult`. **FMU** backend uses `fmpy` (FMI 2.0/3.0), mapping `backend_config.input_ports[].sensor_id` to live/historical values. **Python** backend imports `module:class` in an isolated subprocess (resource-limited).
- `simulation_definitions.backend_config` JSONB per suggestion-3 (fmu_url/ports/step_size or python module/class/requirements).

**Testing**:
- `Integration: load fixture FMU, run with inputs → outputs returned`
- `Unit: python backend runs in subprocess; exceptions captured as failed result`

#### 9.2 — Run orchestration with acknowledgement guardrail

**What**: Execute runs asynchronously and enforce the predictive-output acknowledgement.

**Design**:
- `POST /simulations/{def_id}/runs` → enqueue worker; lifecycle `pending→running→completed|failed|cancelled` on `simulation_runs`; `results` JSONB (per suggestion-3: predicted_failure_hours, max_stress, time_series_url, warnings).
- Results are flagged `requires_acknowledgement:true`; any endpoint that would convert a sim result into a physical action or auto-ticket is **blocked** until `POST /simulations/runs/{id}:acknowledge` sets `acknowledged_by/at`. All sim outputs in API/UI carry a `predictive_estimate` label.

**Testing**:
- `Integration: run → status transitions to completed, results stored`
- `Integration: attempt to act on unacknowledged result → 409; after acknowledge → allowed`
- `Unit: results payload always carries predictive_estimate=true`

---

## Phase 10: Edge Agent

### Purpose
Serve constrained/OT environments and reduce egress: a local agent that aggregates, filters, and buffers sensor data, operating offline and syncing upstream with store-and-forward. Addresses the research's egress-cost and OT-cybersecurity risks.

### Tasks

#### 10.1 — Edge agent core

**What**: Standalone agent running the connector framework locally with local buffering.

**Design**:
- Reuses `dtp-ingest` connectors; local SQLite/Parquet ring buffer; configurable aggregation (downsample, dead-band filtering) before upstream send.
- Upstream sync: authenticated (API key / X.509) batched POST to `/ingest/{connector_id}` or MQTT bridge; **store-and-forward** persists during disconnect and replays on reconnect in timestamp order.
- One-way egress posture (no inbound control channel) to respect OT segmentation / data-diode deployments (IEC 62443).

**Testing**:
- `Integration: agent buffers during simulated upstream outage → replays in order on reconnect`
- `Unit: dead-band filter drops sub-threshold deltas`
- `Integration: agent → API ingest path produces sensor_readings identical to direct ingest`

#### 10.2 — Edge packaging & provisioning

**What**: Distributable agent (wheel + Docker image) with config + enrolment.

**Design**:
- `docker-compose.edge.yml`; enrolment via tenant API key → agent registers a `data_connector` and obtains a scoped credential. Config file (YAML) maps local sources → sensors.

**Testing**:
- `Integration: docker compose -f docker-compose.edge.yml up with fixture broker → readings appear upstream`
- `Unit: enrolment with invalid key → refused`

---

## Phase 11: Optional Integrations & Scale-Out (Backlog)

### Purpose
Additive capabilities that extend reach without altering the core: graph projection for deep impact analysis, Kafka ingest at scale, OpenUSD/Omniverse bridge for photorealistic rendering, and WebXR AR overlay. Each is independently optional.

### Tasks

#### 11.1 — Graph projection (impact analysis)

**What**: Optional Neo4j/Apache AGE projection of the asset knowledge graph synced from Postgres.

**Design**:
- Implements **data-model-suggestion-4**'s graph layer as a *read projection* synced via Redis Stream events (asset/sensor/anomaly changes). Adds `FEEDS_INTO`/`DEPENDS_ON` edges for impact analysis Cypher queries; powers "if pump X fails, what is affected?". Apache AGE keeps it inside Postgres for SME single-node.

**Testing**:
- `Integration: create assets+dependency edges → impact query returns downstream set ordered by distance`
- `Integration: asset change in Postgres → projected to graph within sync interval`

#### 11.2 — Kafka connector, OpenUSD/Omniverse bridge, WebXR

**What**: Kafka ingest adapter; OpenUSD import + optional Omniverse Cloud rendering bridge; WebXR AR viewer mode.

**Design**:
- Kafka connector (`aiokafka`) reusing the 3.3 pipeline. OpenUSD ingest via OpenUSD Exchange SDK → glTF/3D Tiles; optional Omniverse Cloud API render stream embedded in the viewer. WebXR mode in the 3D viewport for in-field overlay.

**Testing**:
- `Integration (kafka testcontainer): produce message → sensor_readings persisted`
- `Integration: import sample .usd → tileset produced like IFC path`
- `E2E: WebXR mode activates on supported device (mocked)`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (scaffold, DB, auth)        ─── required by everything
    │
Phase 2: Assets / Ontology / 3D models          ─── requires P1
    │
Phase 3: IoT Ingest + Real-time sync            ─── requires P2  ← core value
    │
    ├── Phase 4: Historical playback / TS query  ─── requires P3
    │       │
    │       └── Phase 5: Web 3D viewer + dashboards ─ requires P2,P3,P4
    │                   (UI can start vs. mocks after P2)
    │
    ├── Phase 6: Alerting + Maintenance          ─── requires P3 (║ with P4/P5)
    │
    └── Phase 7: Anomaly detection (explainable) ─── requires P3,P4 (uses P6 to alert)
            │
            └── Phase 8: Ontology mapping + NL query + MCP ─ requires P2,P3,P7
                    │
            Phase 9: Simulation / what-if         ─── requires P2,P4 (║ with P7/P8)
                    │
            Phase 10: Edge agent                  ─── requires P3 (║ with P7–P9)
                    │
            Phase 11: Graph / Kafka / USD / WebXR ─── optional, requires P3 (+P5 for WebXR)
```

**Parallelism opportunities:**
- After **Phase 3**: Phases **4**, **6**, and **10** can proceed concurrently.
- **Phase 5** (frontend) can begin against mocked APIs right after **Phase 2** and integrate as 3/4 land.
- After **Phase 4**: Phases **7** and **9** can proceed concurrently; **8** follows **7**.
- **Phase 11** items are mutually independent backlog tasks.

---

## Definition of Done (per phase)

A phase is complete only when **all** of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass (`pytest`, Vitest); e2e (Playwright) pass where the phase touches the UI.
3. Linting and formatting pass: `ruff check`, `ruff format --check`, ESLint/Prettier on `web/`.
4. Type checking passes: `mypy --strict packages`, `tsc --noEmit` on `web/`.
5. Docker build succeeds and `docker compose up` brings the stack to healthy (`/healthz` green).
6. The phase's headline capability works end-to-end against a real Postgres/TimescaleDB/Redis (via testcontainers or the Compose stack).
7. New configuration options are documented in `docs/` and reflected in `.env.example` and the Helm `values.yaml`.
8. New API endpoints appear in the auto-generated OpenAPI 3.1 spec, and the TypeScript SDK regenerates without error.
9. Alembic migrations are created, reversible (`upgrade`/`downgrade` tested), and include any TimescaleDB hypertable/continuous-aggregate/policy changes.
10. Multi-tenant isolation is verified (RLS holds; no cross-tenant leakage) and relevant OWASP API Security Top-10 items (BOLA, broken auth, mass assignment) are covered by tests for any new endpoints.
```
