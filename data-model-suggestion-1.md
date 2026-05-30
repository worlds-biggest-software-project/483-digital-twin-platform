# Data Model Suggestion 1: Normalized Relational Model (PostgreSQL)

## Approach

A fully normalized (3NF+) relational schema in PostgreSQL. Every entity occupies its own table with strict foreign keys, check constraints, and comprehensive indexes. This is the most conservative and well-understood approach, providing strong consistency guarantees, mature tooling, and straightforward backup/restore procedures.

## Why This Suits a Digital Twin Platform

A digital twin platform has clearly identifiable entities -- tenants, sites, assets, sensors, maintenance records, users, roles -- that map naturally to relational tables. The asset hierarchy (fleet > site > zone > equipment > component) is a classic adjacency-list or closure-table problem. Relational databases also provide transactional guarantees important for maintenance workflows and access-control mutations.

**Trade-off:** The high-frequency sensor telemetry data is the weak point. PostgreSQL can handle moderate time-series workloads with partitioning, but at industrial scale (millions of data points per second), a dedicated time-series store is recommended alongside this schema. This model assumes telemetry is offloaded to TimescaleDB or InfluxDB, with this schema managing the metadata, configuration, and business logic layers.

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
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    user_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local'
                    CHECK (auth_provider IN ('local', 'oidc', 'saml', 'ldap')),
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
    description     TEXT,
    is_system_role  BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE permissions (
    permission_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(100) NOT NULL UNIQUE,
    description     TEXT,
    resource_type   VARCHAR(50) NOT NULL
                    CHECK (resource_type IN ('asset', 'site', 'simulation', 'maintenance', 'admin'))
);

CREATE TABLE role_permissions (
    role_id         UUID NOT NULL REFERENCES roles(role_id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permissions(permission_id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(role_id) ON DELETE CASCADE,
    scope_asset_id  UUID,  -- NULL = tenant-wide, non-NULL = scoped to asset subtree
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id, COALESCE(scope_asset_id, '00000000-0000-0000-0000-000000000000'))
);

-- ============================================================
-- ASSET HIERARCHY
-- ============================================================

CREATE TABLE asset_types (
    asset_type_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    code            VARCHAR(100) NOT NULL,
    label           VARCHAR(255) NOT NULL,
    icon            VARCHAR(100),
    dtdl_model_id   VARCHAR(512),    -- DTDL @id for ontology alignment
    aas_id          VARCHAR(512),    -- Asset Administration Shell identifier
    level           VARCHAR(50) NOT NULL
                    CHECK (level IN ('fleet', 'site', 'zone', 'equipment', 'component', 'sensor')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE assets (
    asset_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    asset_type_id   UUID NOT NULL REFERENCES asset_types(asset_type_id),
    parent_asset_id UUID REFERENCES assets(asset_id) ON DELETE SET NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    serial_number   VARCHAR(255),
    iso_81346_ref   VARCHAR(255),    -- ISO 81346 reference designation
    status          VARCHAR(50) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'inactive', 'maintenance', 'decommissioned')),
    commissioned_at TIMESTAMPTZ,
    geo_latitude    DOUBLE PRECISION,
    geo_longitude   DOUBLE PRECISION,
    geo_altitude    DOUBLE PRECISION,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_assets_tenant ON assets(tenant_id);
CREATE INDEX idx_assets_parent ON assets(parent_asset_id);
CREATE INDEX idx_assets_type ON assets(asset_type_id);
CREATE INDEX idx_assets_status ON assets(tenant_id, status);

-- Closure table for efficient ancestor/descendant queries on the asset hierarchy
CREATE TABLE asset_closure (
    ancestor_id     UUID NOT NULL REFERENCES assets(asset_id) ON DELETE CASCADE,
    descendant_id   UUID NOT NULL REFERENCES assets(asset_id) ON DELETE CASCADE,
    depth           INT NOT NULL CHECK (depth >= 0),
    PRIMARY KEY (ancestor_id, descendant_id)
);

CREATE INDEX idx_asset_closure_desc ON asset_closure(descendant_id);

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
    lod_level       SMALLINT NOT NULL DEFAULT 0 CHECK (lod_level BETWEEN 0 AND 5),
    checksum_sha256 VARCHAR(64) NOT NULL,
    uploaded_by     UUID REFERENCES users(user_id),
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_model_files_asset ON model_files(asset_id);

CREATE TABLE spatial_anchors (
    anchor_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(asset_id) ON DELETE CASCADE,
    model_file_id   UUID REFERENCES model_files(model_file_id) ON DELETE SET NULL,
    transform_matrix DOUBLE PRECISION[16] NOT NULL, -- 4x4 column-major
    bounding_box_min DOUBLE PRECISION[3],
    bounding_box_max DOUBLE PRECISION[3],
    coordinate_system VARCHAR(50) NOT NULL DEFAULT 'local'
                    CHECK (coordinate_system IN ('local', 'wgs84', 'utm', 'site_relative')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- SENSORS AND DATA STREAMS
-- ============================================================

CREATE TABLE sensors (
    sensor_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    asset_id        UUID NOT NULL REFERENCES assets(asset_id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    sensor_tag      VARCHAR(255) NOT NULL,  -- raw tag from source system
    unit_of_measure VARCHAR(50),
    data_type       VARCHAR(50) NOT NULL
                    CHECK (data_type IN ('float', 'integer', 'boolean', 'string', 'geo', 'vector')),
    protocol        VARCHAR(50) NOT NULL
                    CHECK (protocol IN ('mqtt', 'opcua', 'rest', 'kafka', 'modbus')),
    sampling_interval_ms INT,
    ontology_mapping VARCHAR(512),   -- mapped ontology property IRI
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, sensor_tag)
);

CREATE INDEX idx_sensors_asset ON sensors(asset_id);
CREATE INDEX idx_sensors_active ON sensors(tenant_id, is_active);

CREATE TABLE data_connectors (
    connector_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    protocol        VARCHAR(50) NOT NULL
                    CHECK (protocol IN ('mqtt', 'opcua', 'rest', 'kafka', 'modbus')),
    endpoint_url    TEXT NOT NULL,
    auth_method     VARCHAR(50) NOT NULL DEFAULT 'none'
                    CHECK (auth_method IN ('none', 'basic', 'token', 'certificate', 'oauth2')),
    auth_credentials_ref VARCHAR(512),  -- reference to secrets manager
    status          VARCHAR(50) NOT NULL DEFAULT 'disconnected'
                    CHECK (status IN ('connected', 'disconnected', 'error', 'paused')),
    last_heartbeat  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sensor_connector_map (
    sensor_id       UUID NOT NULL REFERENCES sensors(sensor_id) ON DELETE CASCADE,
    connector_id    UUID NOT NULL REFERENCES data_connectors(connector_id) ON DELETE CASCADE,
    source_path     VARCHAR(512) NOT NULL,  -- topic, node ID, or endpoint path
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
    backend_config  TEXT,           -- serialized configuration (kept in plain text for relational purity)
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
    input_parameters TEXT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    result_summary  TEXT,
    acknowledged_by UUID REFERENCES users(user_id),
    acknowledged_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sim_runs_def ON simulation_runs(simulation_def_id);
CREATE INDEX idx_sim_runs_status ON simulation_runs(tenant_id, status);

-- ============================================================
-- ANOMALY DETECTION AND ALERTING
-- ============================================================

CREATE TABLE anomaly_models (
    model_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    algorithm       VARCHAR(100) NOT NULL,
    target_sensors  UUID[] NOT NULL,  -- array of sensor IDs monitored
    training_status VARCHAR(50) NOT NULL DEFAULT 'untrained'
                    CHECK (training_status IN ('untrained', 'training', 'trained', 'failed')),
    last_trained_at TIMESTAMPTZ,
    model_artifact_url TEXT,
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
    description     TEXT,
    causal_component_id UUID REFERENCES assets(asset_id),  -- explainable attribution
    confidence      DOUBLE PRECISION CHECK (confidence BETWEEN 0 AND 1),
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ,
    acknowledged_by UUID REFERENCES users(user_id)
);

CREATE INDEX idx_anomaly_events_asset ON anomaly_events(asset_id, detected_at DESC);
CREATE INDEX idx_anomaly_events_severity ON anomaly_events(tenant_id, severity, detected_at DESC);

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
    external_ticket_ref VARCHAR(255),  -- SCADA / CMMS reference
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_tickets_asset ON maintenance_tickets(asset_id);
CREATE INDEX idx_tickets_status ON maintenance_tickets(tenant_id, status);

CREATE TABLE digital_thread_links (
    link_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(asset_id) ON DELETE CASCADE,
    link_type       VARCHAR(50) NOT NULL
                    CHECK (link_type IN ('procurement', 'maintenance', 'compliance',
                                         'certification', 'manual', 'drawing', 'photo')),
    title           VARCHAR(512) NOT NULL,
    url             TEXT NOT NULL,
    metadata        TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_digital_thread_asset ON digital_thread_links(asset_id);

-- ============================================================
-- ALERT ROUTING
-- ============================================================

CREATE TABLE alert_channels (
    channel_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    channel_type    VARCHAR(50) NOT NULL
                    CHECK (channel_type IN ('email', 'webhook', 'scada', 'sms', 'pagerduty', 'slack')),
    config          TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alert_rules (
    rule_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    channel_id      UUID NOT NULL REFERENCES alert_channels(channel_id) ON DELETE CASCADE,
    severity_filter VARCHAR(20)[]  NOT NULL DEFAULT '{critical}',
    asset_scope_id  UUID REFERENCES assets(asset_id),  -- NULL = all assets
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- DASHBOARDS
-- ============================================================

CREATE TABLE dashboards (
    dashboard_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    owner_user_id   UUID NOT NULL REFERENCES users(user_id),
    is_public       BOOLEAN NOT NULL DEFAULT FALSE,
    layout_config   TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE dashboard_widgets (
    widget_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dashboard_id    UUID NOT NULL REFERENCES dashboards(dashboard_id) ON DELETE CASCADE,
    widget_type     VARCHAR(50) NOT NULL
                    CHECK (widget_type IN ('3d_viewport', 'time_series_chart', 'gauge',
                                           'table', 'alert_feed', 'kpi_card', 'map')),
    config          TEXT NOT NULL,
    position_x      INT NOT NULL DEFAULT 0,
    position_y      INT NOT NULL DEFAULT 0,
    width           INT NOT NULL DEFAULT 4,
    height          INT NOT NULL DEFAULT 3
);
```

---

## Trade-offs

| Dimension | Assessment |
|-----------|-----------|
| **Consistency** | Excellent. Full ACID, foreign keys prevent orphans, check constraints enforce domain rules. |
| **Query flexibility** | Good for metadata/config queries. Complex hierarchy traversals need the closure table. |
| **Schema evolution** | Moderate. Adding columns requires ALTER TABLE and possible backfills. Online DDL tools (pg_osc, pgroll) mitigate downtime. |
| **Time-series fit** | Poor at scale. This schema deliberately excludes raw telemetry; pair with TimescaleDB or InfluxDB. |
| **Operational simplicity** | Excellent. One database engine, well-understood backup and replication (pg_basebackup, streaming replication). |
| **Multi-tenancy** | Row-level via tenant_id columns and Row-Level Security policies. Schema-per-tenant is an alternative for strict isolation. |

## Scalability Considerations

- Enable PostgreSQL Row-Level Security (RLS) policies on all tenant-scoped tables to enforce isolation at the database layer.
- The closure table (`asset_closure`) should be maintained via application triggers or a stored procedure on INSERT/DELETE to `assets`.
- For deployments exceeding ~50 million asset rows, consider Citus for horizontal sharding on `tenant_id`.
- Telemetry data must be stored externally. Reference sensor readings by `sensor_id` and timestamp range in the time-series store.

## Migration Path

This normalized schema serves well as the starting point. If flexible/dynamic properties become necessary (e.g., custom metadata per asset type), consider migrating select columns to JSONB (see Suggestion 3). If graph traversal queries dominate (e.g., impact analysis across deeply nested hierarchies), consider adding a graph layer (see Suggestion 4) while keeping this schema as the system of record.
