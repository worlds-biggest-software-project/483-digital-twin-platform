# Data Model Suggestion 3: Hybrid Relational + JSONB Model (PostgreSQL)

## Approach

A pragmatic middle ground that uses PostgreSQL relational tables for core entities and relationships, while leveraging JSONB columns for flexible, schema-variable data such as sensor metadata, simulation parameters, ontology mappings, dashboard configurations, and asset-type-specific properties. GIN indexes on JSONB columns provide performant queries without sacrificing flexibility.

## Why This Suits a Digital Twin Platform

Digital twin platforms face a fundamental tension: the core domain (tenants, users, assets, sensors) has well-defined structure, but the per-asset-type metadata is wildly heterogeneous. A pump has flow rate, head pressure, and impeller diameter. An HVAC unit has BTU rating, refrigerant type, and zone assignments. Forcing all possible properties into relational columns creates a sparse, unmanageable schema. Forcing everything into JSON loses referential integrity.

The hybrid approach resolves this by keeping the stable relational backbone (foreign keys, constraints, indexes) while using JSONB for:

1. **Asset-type-specific properties** -- each asset type defines its own property schema via DTDL/AAS models, stored and queried through JSONB.
2. **Ontology mappings** -- sensor-to-ontology mappings are complex nested structures that vary per integration.
3. **Simulation parameters** -- input/output configs for FMU/Python backends have no fixed schema.
4. **Dashboard layouts and widget configs** -- inherently dynamic, user-defined structures.
5. **Connector configurations** -- each protocol (MQTT, OPC-UA, Kafka) has different connection parameters.

**Trade-off:** JSONB queries are slower than indexed relational columns for simple lookups, but GIN indexes close most of the gap. The schema is harder to document exhaustively since JSONB contents are partially self-describing. Developers must discipline themselves to keep core relationships relational rather than lazily dumping everything into JSON.

---

## Schema Definition

```sql
-- ============================================================
-- MULTI-TENANCY AND IDENTITY
-- ============================================================

CREATE TABLE tenants (
    tenant_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free'
                    CHECK (plan_tier IN ('free', 'starter', 'professional', 'enterprise')),
    settings        JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- settings shape: { "max_assets": 1000, "retention_days": 365,
    --                    "features": ["simulation", "anomaly_detection"],
    --                    "branding": { "logo_url": "...", "primary_color": "#006" } }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    user_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
    profile         JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- profile shape: { "timezone": "Europe/London", "preferences": { "theme": "dark" },
    --                   "notification_channels": ["email", "slack"] }
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE roles (
    role_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- permissions shape: [
    --   { "resource": "asset", "actions": ["read", "write"], "scope": "subtree", "asset_id": "uuid" },
    --   { "resource": "simulation", "actions": ["read", "execute"], "scope": "tenant" }
    -- ]
    is_system_role  BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_roles_permissions ON roles USING GIN (permissions);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(role_id) ON DELETE CASCADE,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id)
);

-- ============================================================
-- ASSET HIERARCHY WITH FLEXIBLE PROPERTIES
-- ============================================================

CREATE TABLE asset_types (
    asset_type_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    code            VARCHAR(100) NOT NULL,
    label           VARCHAR(255) NOT NULL,
    level           VARCHAR(50) NOT NULL
                    CHECK (level IN ('fleet', 'site', 'zone', 'equipment', 'component', 'sensor')),
    property_schema JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- property_schema defines the expected shape for assets of this type.
    -- Uses a JSON Schema-like format for validation:
    -- {
    --   "properties": {
    --     "flow_rate_lpm": { "type": "number", "unit": "l/min", "required": true },
    --     "impeller_diameter_mm": { "type": "number", "unit": "mm" },
    --     "material": { "type": "string", "enum": ["cast_iron", "stainless", "bronze"] }
    --   },
    --   "dtdl_model_id": "dtmi:com:example:Pump;1",
    --   "aas_submodel_id": "urn:aas:submodel:pump_properties:1.0"
    -- }
    icon            VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE INDEX idx_asset_types_schema ON asset_types USING GIN (property_schema);

CREATE TABLE assets (
    asset_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    asset_type_id   UUID NOT NULL REFERENCES asset_types(asset_type_id),
    parent_asset_id UUID REFERENCES assets(asset_id) ON DELETE SET NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    serial_number   VARCHAR(255),
    status          VARCHAR(50) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'inactive', 'maintenance', 'decommissioned')),
    -- Type-specific properties stored as JSONB, validated against asset_type.property_schema
    properties      JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Example for a pump: { "flow_rate_lpm": 120, "impeller_diameter_mm": 200, "material": "cast_iron" }
    -- Example for HVAC:   { "btu_rating": 48000, "refrigerant": "R-410A", "zones": ["Z1", "Z2"] }
    -- Ontology references
    ontology_refs   JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- { "iso_81346": "=A1.M01", "dtdl_id": "dtmi:...", "aas_id": "urn:aas:..." }
    -- Geolocation
    geo_location    JSONB,
    -- { "latitude": 51.5074, "longitude": -0.1278, "altitude": 12.5, "crs": "WGS84" }
    commissioned_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_assets_tenant ON assets(tenant_id);
CREATE INDEX idx_assets_parent ON assets(parent_asset_id);
CREATE INDEX idx_assets_type ON assets(asset_type_id);
CREATE INDEX idx_assets_status ON assets(tenant_id, status);
CREATE INDEX idx_assets_properties ON assets USING GIN (properties);
CREATE INDEX idx_assets_ontology ON assets USING GIN (ontology_refs);

-- Materialised path for hierarchy queries (alternative to closure table)
-- Stored as an array of ancestor IDs from root to self
ALTER TABLE assets ADD COLUMN ancestor_path UUID[] NOT NULL DEFAULT '{}';
CREATE INDEX idx_assets_ancestor_path ON assets USING GIN (ancestor_path);
-- Query: all descendants of asset X => WHERE 'X' = ANY(ancestor_path)
-- Query: all ancestors of asset Y => SELECT unnest(ancestor_path) FROM assets WHERE asset_id = Y

-- ============================================================
-- 3D MODELS AND SPATIAL DATA
-- ============================================================

CREATE TABLE model_files (
    model_file_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    asset_id        UUID NOT NULL REFERENCES assets(asset_id) ON DELETE CASCADE,
    file_name       VARCHAR(512) NOT NULL,
    format          VARCHAR(50) NOT NULL
                    CHECK (format IN ('ifc', 'revit', 'usd', 'usda', 'usdc', 'usdz',
                                      'gltf', 'glb', 'obj', 'fbx', 'las', 'laz', 'e57')),
    storage_url     TEXT NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    lod_level       SMALLINT NOT NULL DEFAULT 0,
    checksum_sha256 VARCHAR(64) NOT NULL,
    -- Processing and tiling metadata
    processing_meta JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- { "tile_set_url": "...", "mesh_count": 1234, "vertex_count": 500000,
    --   "bounding_box": { "min": [0,0,0], "max": [10,5,3] },
    --   "collision_mesh_url": "..." }
    spatial_anchor  JSONB,
    -- { "transform_matrix": [1,0,0,0,...], "coordinate_system": "site_relative",
    --   "origin_offset": [10, 20, 0] }
    uploaded_by     UUID REFERENCES users(user_id),
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_model_files_asset ON model_files(asset_id);

-- ============================================================
-- SENSORS AND DATA CONNECTORS
-- ============================================================

CREATE TABLE sensors (
    sensor_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    asset_id        UUID NOT NULL REFERENCES assets(asset_id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    sensor_tag      VARCHAR(255) NOT NULL,
    unit_of_measure VARCHAR(50),
    data_type       VARCHAR(50) NOT NULL
                    CHECK (data_type IN ('float', 'integer', 'boolean', 'string', 'geo', 'vector')),
    -- Flexible mapping configuration
    mapping_config  JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {
    --   "protocol": "opcua",
    --   "sampling_interval_ms": 1000,
    --   "ontology_iri": "dtmi:com:example:Pump;1:temperature",
    --   "real_estate_core_mapping": "https://w3id.org/rec#Temperature",
    --   "transform": { "scale": 1.0, "offset": -273.15, "clamp_min": -50, "clamp_max": 500 },
    --   "quality_rules": { "stale_after_ms": 5000, "min_value": -50, "max_value": 500 }
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, sensor_tag)
);

CREATE INDEX idx_sensors_asset ON sensors(asset_id);
CREATE INDEX idx_sensors_mapping ON sensors USING GIN (mapping_config);

CREATE TABLE data_connectors (
    connector_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    protocol        VARCHAR(50) NOT NULL
                    CHECK (protocol IN ('mqtt', 'opcua', 'rest', 'kafka', 'modbus')),
    -- Protocol-specific configuration as JSONB
    connection_config JSONB NOT NULL,
    -- MQTT: { "broker_url": "mqtt://...", "port": 1883, "tls": true,
    --         "client_id": "dt-agent-01", "topics": ["plant/+/telemetry"] }
    -- OPC-UA: { "endpoint_url": "opc.tcp://...", "security_mode": "SignAndEncrypt",
    --           "auth": "certificate", "cert_ref": "vault://...",
    --           "node_ids": ["ns=2;s=Pump.Temperature"] }
    -- Kafka: { "bootstrap_servers": "kafka:9092", "group_id": "dt-ingest",
    --          "topics": ["sensor-data"], "schema_registry_url": "http://..." }
    status          VARCHAR(50) NOT NULL DEFAULT 'disconnected'
                    CHECK (status IN ('connected', 'disconnected', 'error', 'paused')),
    health_meta     JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- { "last_heartbeat": "...", "messages_per_sec": 1200, "error_count_24h": 3 }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_connectors_config ON data_connectors USING GIN (connection_config);

CREATE TABLE sensor_connector_map (
    sensor_id       UUID NOT NULL REFERENCES sensors(sensor_id) ON DELETE CASCADE,
    connector_id    UUID NOT NULL REFERENCES data_connectors(connector_id) ON DELETE CASCADE,
    source_path     VARCHAR(512) NOT NULL,
    transform_rules JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- { "jmespath": "payload.readings[0].value", "timestamp_field": "payload.ts",
    --   "timestamp_format": "epoch_ms" }
    PRIMARY KEY (sensor_id, connector_id)
);

-- ============================================================
-- SIMULATIONS
-- ============================================================

CREATE TABLE simulation_definitions (
    simulation_def_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    backend_type    VARCHAR(50) NOT NULL
                    CHECK (backend_type IN ('fmu', 'python', 'custom_plugin')),
    -- Backend-specific configuration
    backend_config  JSONB NOT NULL,
    -- FMU: { "fmu_url": "s3://...", "fmi_version": "2.0",
    --        "input_ports": [{"name": "T_in", "sensor_id": "uuid", "unit": "K"}],
    --        "output_ports": [{"name": "stress", "unit": "MPa"}],
    --        "step_size_ms": 100 }
    -- Python: { "module": "sim.thermal_model", "class": "ThermalSim",
    --           "requirements": ["numpy>=1.24", "scipy>=1.11"] }
    target_asset_id UUID REFERENCES assets(asset_id),
    created_by      UUID NOT NULL REFERENCES users(user_id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE simulation_runs (
    run_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    simulation_def_id UUID NOT NULL REFERENCES simulation_definitions(simulation_def_id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'running', 'completed', 'failed', 'cancelled')),
    input_params    JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- { "flow_rate": 120, "ambient_temp": 35, "scenario": "peak_summer" }
    results         JSONB,
    -- { "predicted_failure_hours": 168, "max_stress_mpa": 245,
    --   "time_series_url": "s3://results/run-xyz.parquet",
    --   "warnings": ["Stress exceeds 80% yield at t=120h"] }
    acknowledged_by UUID REFERENCES users(user_id),
    acknowledged_at TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sim_runs_status ON simulation_runs(tenant_id, status);

-- ============================================================
-- ANOMALY DETECTION AND ALERTING
-- ============================================================

CREATE TABLE anomaly_models (
    model_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    algorithm       VARCHAR(100) NOT NULL,
    target_sensors  UUID[] NOT NULL,
    training_config JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- { "window_size": 3600, "features": ["mean", "std", "fft_peak"],
    --   "hyperparams": { "n_estimators": 100, "contamination": 0.01 },
    --   "training_range": { "from": "2025-01-01", "to": "2026-01-01" } }
    training_status VARCHAR(50) NOT NULL DEFAULT 'untrained',
    model_artifact_url TEXT,
    metrics         JSONB,
    -- { "auc_roc": 0.94, "precision": 0.89, "recall": 0.91, "f1": 0.90 }
    last_trained_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE anomaly_events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    model_id        UUID REFERENCES anomaly_models(model_id) ON DELETE SET NULL,
    asset_id        UUID NOT NULL REFERENCES assets(asset_id),
    sensor_id       UUID REFERENCES sensors(sensor_id),
    severity        VARCHAR(20) NOT NULL
                    CHECK (severity IN ('info', 'warning', 'critical')),
    title           VARCHAR(512) NOT NULL,
    -- Explainability and causal attribution stored as JSONB
    explanation     JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {
    --   "causal_component_id": "uuid",
    --   "causal_component_name": "Bearing B-101",
    --   "confidence": 0.92,
    --   "contributing_factors": [
    --     { "sensor_tag": "PUMP_P101_VIB", "deviation_sigma": 3.2, "direction": "high" },
    --     { "sensor_tag": "PUMP_P101_TEMP", "deviation_sigma": 2.1, "direction": "high" }
    --   ],
    --   "similar_past_events": ["uuid1", "uuid2"],
    --   "recommended_action": "Inspect bearing; similar failures resolved by replacement"
    -- }
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ,
    acknowledged_by UUID REFERENCES users(user_id)
);

CREATE INDEX idx_anomaly_asset ON anomaly_events(asset_id, detected_at DESC);
CREATE INDEX idx_anomaly_severity ON anomaly_events(tenant_id, severity, detected_at DESC);
CREATE INDEX idx_anomaly_explanation ON anomaly_events USING GIN (explanation);

-- ============================================================
-- MAINTENANCE AND DIGITAL THREAD
-- ============================================================

CREATE TABLE maintenance_tickets (
    ticket_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    asset_id        UUID NOT NULL REFERENCES assets(asset_id),
    anomaly_event_id UUID REFERENCES anomaly_events(event_id),
    title           VARCHAR(512) NOT NULL,
    description     TEXT,
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium'
                    CHECK (priority IN ('low', 'medium', 'high', 'critical')),
    status          VARCHAR(50) NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open', 'assigned', 'in_progress', 'resolved', 'closed')),
    assigned_to     UUID REFERENCES users(user_id),
    external_refs   JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- { "cmms_ticket": "WO-2026-1234", "scada_alarm": "ALM-567",
    --   "sap_notification": "100045678" }
    resolution      JSONB,
    -- { "notes": "Bearing replaced", "parts_used": [{"sku": "BRG-7204B", "qty": 1}],
    --   "labour_hours": 2.5, "root_cause": "normal_wear" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_tickets_asset ON maintenance_tickets(asset_id);
CREATE INDEX idx_tickets_status ON maintenance_tickets(tenant_id, status);
CREATE INDEX idx_tickets_ext_refs ON maintenance_tickets USING GIN (external_refs);

CREATE TABLE digital_thread_links (
    link_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(asset_id) ON DELETE CASCADE,
    link_type       VARCHAR(50) NOT NULL
                    CHECK (link_type IN ('procurement', 'maintenance', 'compliance',
                                         'certification', 'manual', 'drawing', 'photo', 'video')),
    title           VARCHAR(512) NOT NULL,
    url             TEXT NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- { "vendor": "Grundfos", "po_number": "PO-2025-789",
    --   "expiry_date": "2027-06-01", "tags": ["warranty", "critical_spare"] }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_thread_asset ON digital_thread_links(asset_id);
CREATE INDEX idx_thread_meta ON digital_thread_links USING GIN (metadata);

-- ============================================================
-- ALERT ROUTING AND DASHBOARDS
-- ============================================================

CREATE TABLE alert_channels (
    channel_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    channel_type    VARCHAR(50) NOT NULL
                    CHECK (channel_type IN ('email', 'webhook', 'scada', 'sms', 'pagerduty', 'slack')),
    config          JSONB NOT NULL,
    -- email:     { "recipients": ["ops@co.com"], "template": "anomaly_alert" }
    -- webhook:   { "url": "https://...", "method": "POST", "headers": {...} }
    -- pagerduty: { "routing_key": "...", "severity_map": {"critical": "critical"} }
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alert_rules (
    rule_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    channel_id      UUID NOT NULL REFERENCES alert_channels(channel_id) ON DELETE CASCADE,
    filter_criteria JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- { "severity": ["critical", "warning"], "asset_scope_id": "uuid",
    --   "include_subtree": true, "asset_types": ["pump", "motor"] }
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_rules_filter ON alert_rules USING GIN (filter_criteria);

CREATE TABLE dashboards (
    dashboard_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    owner_user_id   UUID NOT NULL REFERENCES users(user_id),
    is_public       BOOLEAN NOT NULL DEFAULT FALSE,
    layout          JSONB NOT NULL DEFAULT '{"columns": 12, "row_height": 60}'::jsonb,
    widgets         JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [
    --   { "id": "w1", "type": "3d_viewport", "x": 0, "y": 0, "w": 8, "h": 6,
    --     "config": { "asset_id": "uuid", "camera_preset": "overview" } },
    --   { "id": "w2", "type": "time_series_chart", "x": 8, "y": 0, "w": 4, "h": 3,
    --     "config": { "sensor_ids": ["uuid1", "uuid2"], "range": "24h" } },
    --   { "id": "w3", "type": "alert_feed", "x": 8, "y": 3, "w": 4, "h": 3,
    --     "config": { "severity_filter": ["critical"], "max_items": 20 } }
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Trade-offs

| Dimension | Assessment |
|-----------|-----------|
| **Flexibility** | Excellent. New asset types, connector protocols, and widget types require no schema migration. |
| **Referential integrity** | Good. Core relationships use foreign keys; JSONB references (e.g., sensor_ids in widgets) are not enforced by the database. |
| **Query performance** | Good. GIN indexes support efficient `@>`, `?`, `?&` operators on JSONB. Complex nested queries may need materialized views. |
| **Schema documentation** | Moderate. JSONB shapes must be documented via conventions, JSON Schema validation in the application layer, or check constraints using `jsonb_matches_schema()` (PG 17+). |
| **Migration effort** | Low. Adding new JSONB fields requires no ALTER TABLE. Changing relational columns still needs standard migrations. |
| **Tooling compatibility** | Good. PostgreSQL JSONB is widely supported by ORMs (SQLAlchemy, Prisma, TypeORM) and reporting tools. |
| **Storage efficiency** | Moderate. JSONB is more compact than EAV but less compact than fixed columns. TOAST compression helps for large documents. |

## Scalability Considerations

- Use partial GIN indexes on JSONB columns when only specific keys are queried frequently: `CREATE INDEX idx_assets_material ON assets USING GIN ((properties -> 'material'))`.
- For tenants with >1M assets, consider partitioning the `assets` table by `tenant_id` using declarative partitioning.
- The `ancestor_path` array column enables subtree queries via `@>` operator without a separate closure table, reducing join overhead.
- Validate JSONB shapes at the application layer using JSON Schema or, on PostgreSQL 17+, using the `jsonb_matches_schema()` function in CHECK constraints.
- Raw telemetry data should still be offloaded to TimescaleDB/InfluxDB; this schema handles metadata and configuration.

## Migration Path

This is the recommended starting schema if the team wants rapid iteration without early over-engineering. The relational backbone can be tightened later by extracting frequently-queried JSONB fields into dedicated columns (a non-breaking change). If the asset hierarchy queries become a bottleneck, the `ancestor_path` array can be supplemented with a closure table or migrated to a graph database for that specific concern. The JSONB columns also serve as a natural staging area before committing to a fully normalized column for a property that has proven stable.
