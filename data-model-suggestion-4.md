# Data Model Suggestion 4: Graph Database + Time-Series Hybrid (Neo4j + TimescaleDB)

## Approach

A polyglot persistence architecture that uses Neo4j (labeled property graph) as the primary store for the asset knowledge graph and relationship-heavy queries, paired with TimescaleDB (PostgreSQL extension) for high-frequency sensor telemetry. A thin PostgreSQL instance handles transactional CRUD for tenancy, users, and authentication. This is the domain-specific specialty approach -- digital twins are fundamentally knowledge graphs of interconnected physical entities, and graph databases are purpose-built for this.

## Why This Suits a Digital Twin Platform

The core data challenge in a digital twin platform is representing and querying deeply interconnected entities: assets contain sub-assets, sensors attach to components, components relate to maintenance records, maintenance records link to procurement, anomalies trace causal chains through component hierarchies, and simulations span subsets of the asset graph. These are all graph traversal problems.

1. **Asset hierarchy is a graph problem.** "Find all descendants of Site X that are pumps with an active anomaly" is a variable-depth traversal that relational databases handle awkwardly (recursive CTEs or closure tables) but graph databases handle natively in milliseconds.
2. **Impact analysis is path-finding.** "If this pump fails, what downstream processes are affected?" requires multi-hop relationship traversal -- the core strength of graph engines.
3. **Ontology alignment maps naturally to graphs.** DTDL models, Asset Administration Shell submodels, and ISO 81346 hierarchies are all graph structures. Storing them in a graph database preserves their native form.
4. **The digital thread is a web of connections.** Linking components to procurement records, compliance documents, maintenance history, and simulation results creates a rich knowledge graph that supports natural-language-style queries.
5. **Time-series data needs a specialist.** Sensor telemetry at industrial scale (millions of data points per second) requires a purpose-built time-series engine with columnar compression and continuous aggregates. TimescaleDB delivers this while remaining PostgreSQL-compatible.

**Trade-off:** Polyglot persistence adds operational complexity (two or three database engines to maintain, back up, and monitor). Graph databases have weaker transactional guarantees for bulk writes compared to relational databases. The team needs graph query expertise (Cypher). Data consistency across stores must be managed at the application layer.

---

## Neo4j Graph Schema

### Node Labels and Properties

```cypher
// ============================================================
// CORE ASSET KNOWLEDGE GRAPH
// ============================================================

// -- Tenant --
CREATE CONSTRAINT tenant_id_unique IF NOT EXISTS
FOR (t:Tenant) REQUIRE t.tenant_id IS UNIQUE;

// Properties: tenant_id (UUID), name, slug, plan_tier, created_at

// -- Asset Type --
CREATE CONSTRAINT asset_type_unique IF NOT EXISTS
FOR (at:AssetType) REQUIRE (at.tenant_id, at.code) IS UNIQUE;

// Properties: asset_type_id (UUID), tenant_id, code, label, level,
//             dtdl_model_id, aas_submodel_id, icon

// -- Asset --
CREATE CONSTRAINT asset_id_unique IF NOT EXISTS
FOR (a:Asset) REQUIRE a.asset_id IS UNIQUE;

CREATE INDEX asset_status IF NOT EXISTS FOR (a:Asset) ON (a.status);
CREATE INDEX asset_serial IF NOT EXISTS FOR (a:Asset) ON (a.serial_number);

// Properties: asset_id (UUID), tenant_id, name, description, serial_number,
//             status, iso_81346_ref, commissioned_at,
//             geo_latitude, geo_longitude, geo_altitude,
//             -- Flexible properties as top-level node properties:
//             flow_rate_lpm, impeller_diameter_mm, btu_rating, etc.
//             (graph nodes can have arbitrary properties without schema migration)

// -- Sensor --
CREATE CONSTRAINT sensor_id_unique IF NOT EXISTS
FOR (s:Sensor) REQUIRE s.sensor_id IS UNIQUE;

CREATE INDEX sensor_tag IF NOT EXISTS FOR (s:Sensor) ON (s.sensor_tag);

// Properties: sensor_id (UUID), tenant_id, name, sensor_tag, unit_of_measure,
//             data_type, protocol, sampling_interval_ms, ontology_iri, is_active

// -- ModelFile --
CREATE CONSTRAINT model_file_id_unique IF NOT EXISTS
FOR (m:ModelFile) REQUIRE m.model_file_id IS UNIQUE;

// Properties: model_file_id, file_name, format, storage_url, file_size_bytes,
//             lod_level, checksum_sha256, uploaded_at,
//             transform_matrix (list), bounding_box_min (list), bounding_box_max (list)

// -- AnomalyEvent --
CREATE CONSTRAINT anomaly_id_unique IF NOT EXISTS
FOR (ae:AnomalyEvent) REQUIRE ae.event_id IS UNIQUE;

CREATE INDEX anomaly_severity IF NOT EXISTS FOR (ae:AnomalyEvent) ON (ae.severity);
CREATE INDEX anomaly_detected IF NOT EXISTS FOR (ae:AnomalyEvent) ON (ae.detected_at);

// Properties: event_id, severity, title, description, confidence, detected_at, resolved_at

// -- MaintenanceTicket --
CREATE CONSTRAINT ticket_id_unique IF NOT EXISTS
FOR (mt:MaintenanceTicket) REQUIRE mt.ticket_id IS UNIQUE;

CREATE INDEX ticket_status IF NOT EXISTS FOR (mt:MaintenanceTicket) ON (mt.status);

// Properties: ticket_id, title, description, priority, status,
//             external_ticket_ref, created_at, resolved_at, resolution_notes

// -- SimulationDef & SimulationRun --
CREATE CONSTRAINT sim_def_id_unique IF NOT EXISTS
FOR (sd:SimulationDef) REQUIRE sd.simulation_def_id IS UNIQUE;

CREATE CONSTRAINT sim_run_id_unique IF NOT EXISTS
FOR (sr:SimulationRun) REQUIRE sr.run_id IS UNIQUE;

// -- DataConnector --
CREATE CONSTRAINT connector_id_unique IF NOT EXISTS
FOR (dc:DataConnector) REQUIRE dc.connector_id IS UNIQUE;

// -- DigitalThreadDoc --
CREATE CONSTRAINT doc_id_unique IF NOT EXISTS
FOR (d:DigitalThreadDoc) REQUIRE d.link_id IS UNIQUE;

// Properties: link_id, link_type, title, url, vendor, po_number, expiry_date

// -- OntologyClass (for semantic layer) --
CREATE CONSTRAINT ontology_iri_unique IF NOT EXISTS
FOR (oc:OntologyClass) REQUIRE oc.iri IS UNIQUE;

// Properties: iri, label, standard (dtdl/aas/rec/iso81346), description
```

### Relationship Types

```cypher
// ============================================================
// RELATIONSHIPS
// ============================================================

// -- Asset hierarchy --
// (:Asset)-[:CHILD_OF]->(:Asset)
// (:Asset)-[:INSTANCE_OF]->(:AssetType)
// (:Asset)-[:BELONGS_TO]->(:Tenant)

// -- Sensors --
// (:Sensor)-[:MONITORS]->(:Asset)
// (:Sensor)-[:CONNECTED_VIA]->(:DataConnector)

// -- 3D Models --
// (:ModelFile)-[:REPRESENTS]->(:Asset)

// -- Anomalies (causal chain) --
// (:AnomalyEvent)-[:DETECTED_ON]->(:Asset)
// (:AnomalyEvent)-[:TRIGGERED_BY]->(:Sensor)
// (:AnomalyEvent)-[:CAUSED_BY]->(:Asset)          // causal attribution
// (:AnomalyEvent)-[:SIMILAR_TO]->(:AnomalyEvent)  // pattern matching

// -- Maintenance --
// (:MaintenanceTicket)-[:FOR_ASSET]->(:Asset)
// (:MaintenanceTicket)-[:TRIGGERED_BY]->(:AnomalyEvent)
// (:MaintenanceTicket)-[:ASSIGNED_TO]->(:User)

// -- Digital Thread --
// (:DigitalThreadDoc)-[:DOCUMENTS]->(:Asset)
// (:DigitalThreadDoc)-[:RELATES_TO]->(:MaintenanceTicket)

// -- Simulations --
// (:SimulationDef)-[:TARGETS]->(:Asset)
// (:SimulationRun)-[:EXECUTION_OF]->(:SimulationDef)
// (:SimulationRun)-[:ACKNOWLEDGED_BY]->(:User)

// -- Ontology mapping --
// (:Asset)-[:CLASSIFIED_AS]->(:OntologyClass)
// (:Sensor)-[:MAPS_TO]->(:OntologyClass)
// (:OntologyClass)-[:SUBCLASS_OF]->(:OntologyClass)  // ontology hierarchy

// -- Connectors --
// (:DataConnector)-[:BELONGS_TO]->(:Tenant)

// -- Impact relationships (for impact analysis) --
// (:Asset)-[:FEEDS_INTO]->(:Asset)     // process flow
// (:Asset)-[:DEPENDS_ON]->(:Asset)     // operational dependency
// (:Asset)-[:SHARES_CIRCUIT]->(:Asset) // electrical/hydraulic circuit
```

### Example Queries

```cypher
// 1. Find all pumps under Site X running above 80% capacity (spatial query via NL)
MATCH (site:Asset {name: "Site Alpha"})<-[:CHILD_OF*]-(pump:Asset)-[:INSTANCE_OF]->(t:AssetType {code: "pump"})
WHERE pump.status = 'active'
MATCH (s:Sensor)-[:MONITORS]->(pump)
WHERE s.sensor_tag CONTAINS 'CAPACITY' AND s.latest_value > 80
RETURN pump.name, pump.asset_id, s.latest_value

// 2. Impact analysis: what is affected if Pump P-101 fails?
MATCH (p:Asset {name: "Pump P-101"})-[:FEEDS_INTO|DEPENDS_ON*1..5]->(affected:Asset)
RETURN affected.name, affected.status, length(path) AS distance
ORDER BY distance

// 3. Causal chain for an anomaly
MATCH (ae:AnomalyEvent {event_id: $eventId})-[:CAUSED_BY]->(cause:Asset),
      (ae)-[:TRIGGERED_BY]->(s:Sensor),
      (ae)-[:DETECTED_ON]->(a:Asset)
OPTIONAL MATCH (ae)-[:SIMILAR_TO]->(past:AnomalyEvent)
RETURN ae, cause.name AS root_cause, s.sensor_tag, a.name AS affected_asset,
       collect(past.event_id) AS similar_events

// 4. Digital thread: all documents for an asset and its components
MATCH (a:Asset {asset_id: $assetId})<-[:CHILD_OF*0..]-(component:Asset)
OPTIONAL MATCH (doc:DigitalThreadDoc)-[:DOCUMENTS]->(component)
RETURN component.name, collect({type: doc.link_type, title: doc.title, url: doc.url}) AS documents

// 5. Ontology traversal: all assets classified under a standard class
MATCH (oc:OntologyClass {iri: "dtmi:com:example:RotatingEquipment;1"})<-[:SUBCLASS_OF*0..]-
      (sub:OntologyClass)<-[:CLASSIFIED_AS]-(a:Asset)
RETURN a.name, a.asset_id, sub.label AS specific_class
```

---

## TimescaleDB Schema (Sensor Telemetry)

```sql
-- ============================================================
-- TIME-SERIES DATA (TimescaleDB / PostgreSQL Extension)
-- ============================================================

-- Enable TimescaleDB
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Core telemetry hypertable
CREATE TABLE sensor_readings (
    time            TIMESTAMPTZ NOT NULL,
    sensor_id       UUID NOT NULL,
    value_numeric   DOUBLE PRECISION,
    value_string    VARCHAR(255),     -- for non-numeric sensors
    quality         SMALLINT NOT NULL DEFAULT 0,  -- 0=good, 1=uncertain, 2=bad, 3=stale
    PRIMARY KEY (sensor_id, time)
);

SELECT create_hypertable('sensor_readings', 'time',
    chunk_time_interval => INTERVAL '1 day',
    create_default_indexes => FALSE
);

-- Optimised indexes for common query patterns
CREATE INDEX idx_readings_sensor_time ON sensor_readings (sensor_id, time DESC);
CREATE INDEX idx_readings_time ON sensor_readings (time DESC);

-- Enable columnar compression for historical data
ALTER TABLE sensor_readings SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'sensor_id',
    timescaledb.compress_orderby = 'time DESC'
);

-- Auto-compress chunks older than 7 days
SELECT add_compression_policy('sensor_readings', INTERVAL '7 days');

-- Retention policy: drop raw data older than 2 years (configurable per tenant)
SELECT add_retention_policy('sensor_readings', INTERVAL '2 years');

-- ============================================================
-- CONTINUOUS AGGREGATES FOR DASHBOARD QUERIES
-- ============================================================

-- 1-minute aggregates
CREATE MATERIALIZED VIEW sensor_readings_1min
WITH (timescaledb.continuous) AS
SELECT
    sensor_id,
    time_bucket('1 minute', time) AS bucket,
    avg(value_numeric) AS avg_value,
    min(value_numeric) AS min_value,
    max(value_numeric) AS max_value,
    count(*) AS sample_count,
    last(value_numeric, time) AS last_value
FROM sensor_readings
GROUP BY sensor_id, time_bucket('1 minute', time)
WITH NO DATA;

SELECT add_continuous_aggregate_policy('sensor_readings_1min',
    start_offset => INTERVAL '2 hours',
    end_offset => INTERVAL '1 minute',
    schedule_interval => INTERVAL '1 minute'
);

-- 1-hour aggregates
CREATE MATERIALIZED VIEW sensor_readings_1hr
WITH (timescaledb.continuous) AS
SELECT
    sensor_id,
    time_bucket('1 hour', time) AS bucket,
    avg(value_numeric) AS avg_value,
    min(value_numeric) AS min_value,
    max(value_numeric) AS max_value,
    count(*) AS sample_count,
    last(value_numeric, time) AS last_value
FROM sensor_readings
GROUP BY sensor_id, time_bucket('1 hour', time)
WITH NO DATA;

SELECT add_continuous_aggregate_policy('sensor_readings_1hr',
    start_offset => INTERVAL '1 day',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

-- 1-day aggregates
CREATE MATERIALIZED VIEW sensor_readings_1day
WITH (timescaledb.continuous) AS
SELECT
    sensor_id,
    time_bucket('1 day', time) AS bucket,
    avg(value_numeric) AS avg_value,
    min(value_numeric) AS min_value,
    max(value_numeric) AS max_value,
    count(*) AS sample_count
FROM sensor_readings
GROUP BY sensor_id, time_bucket('1 day', time)
WITH NO DATA;

SELECT add_continuous_aggregate_policy('sensor_readings_1day',
    start_offset => INTERVAL '7 days',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day'
);

-- ============================================================
-- MAINTENANCE EVENT LOG (for correlation with sensor data)
-- ============================================================

CREATE TABLE maintenance_events (
    time            TIMESTAMPTZ NOT NULL,
    asset_id        UUID NOT NULL,
    ticket_id       UUID NOT NULL,
    event_type      VARCHAR(50) NOT NULL,  -- 'created', 'started', 'completed'
    description     TEXT,
    PRIMARY KEY (asset_id, time)
);

SELECT create_hypertable('maintenance_events', 'time',
    chunk_time_interval => INTERVAL '30 days'
);
```

---

## PostgreSQL Schema (Identity and Transactional Data)

```sql
-- ============================================================
-- TRANSACTIONAL STORE (standard PostgreSQL, NOT TimescaleDB)
-- Handles auth, sessions, and data that requires ACID guarantees
-- but does not belong in the graph
-- ============================================================

CREATE TABLE tenants (
    tenant_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    neo4j_database  VARCHAR(100),  -- tenant-specific Neo4j database name
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    user_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE api_keys (
    key_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    key_hash        VARCHAR(255) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    scopes          TEXT[] NOT NULL DEFAULT '{}',
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE dashboard_configs (
    dashboard_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    owner_user_id   UUID NOT NULL REFERENCES users(user_id),
    name            VARCHAR(255) NOT NULL,
    is_public       BOOLEAN NOT NULL DEFAULT FALSE,
    layout          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alert_channels (
    channel_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    channel_type    VARCHAR(50) NOT NULL,
    config          JSONB NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Architecture Overview

```
                    +-----------------------+
                    |    Application Layer   |
                    |  (REST API / GraphQL)  |
                    +-----------+-----------+
                                |
          +---------------------+---------------------+
          |                     |                     |
  +-------v-------+   +--------v--------+   +--------v--------+
  |    Neo4j       |   |  TimescaleDB    |   |   PostgreSQL    |
  |  (Graph DB)    |   |  (Time-Series)  |   |  (Transactional)|
  |                |   |                 |   |                 |
  | - Assets       |   | - Sensor        |   | - Tenants       |
  | - Sensors      |   |   readings      |   | - Users         |
  | - Hierarchy    |   | - Continuous    |   | - API keys      |
  | - Anomalies    |   |   aggregates    |   | - Dashboards    |
  | - Tickets      |   | - Maintenance   |   | - Alert config  |
  | - Simulations  |   |   events        |   |                 |
  | - Ontology     |   |                 |   |                 |
  | - Digital      |   |                 |   |                 |
  |   thread       |   |                 |   |                 |
  +----------------+   +-----------------+   +-----------------+
```

---

## Trade-offs

| Dimension | Assessment |
|-----------|-----------|
| **Relationship queries** | Excellent. Multi-hop traversals, impact analysis, and ontology queries are native operations measured in milliseconds. |
| **Time-series performance** | Excellent. TimescaleDB provides columnar compression, continuous aggregates, and retention policies purpose-built for sensor data. |
| **Operational complexity** | High. Three database engines to deploy, monitor, back up, and upgrade. Requires DevOps maturity. |
| **Consistency model** | Complex. Cross-store transactions are not available; the application layer must handle saga patterns or eventual consistency. |
| **Query language** | Cypher for graph queries is powerful but less widely known than SQL. TimescaleDB uses standard SQL. |
| **Cost** | Neo4j Enterprise (clustering, multi-tenancy) has commercial licensing. Neo4j Community Edition is GPL with limitations. Alternatives: Apache AGE (PostgreSQL graph extension) for simpler deployments. |
| **Scalability** | Graph: Neo4j clusters scale reads well; write scaling requires sharding (Neo4j Fabric). Time-series: TimescaleDB scales horizontally with multi-node or Timescale Cloud. |
| **Ecosystem** | Good. Neo4j has mature drivers for Python, JavaScript, Java, Go. TimescaleDB is PostgreSQL-compatible. |

## Scalability Considerations

- **Multi-tenancy in Neo4j:** Use Neo4j 5+ multi-database feature (one database per tenant) or label-based tenant isolation with tenant_id properties on all nodes.
- **TimescaleDB partitioning:** Hypertables auto-partition by time. For very high cardinality (>100K sensors), add space partitioning by sensor_id hash.
- **Cross-store sync:** Use Apache Kafka as the event bus. Write events flow to Kafka, and separate consumers update Neo4j, TimescaleDB, and PostgreSQL. This provides eventual consistency with replay capability.
- **Read scaling:** Neo4j read replicas for graph queries; TimescaleDB continuous aggregates eliminate expensive real-time rollups; PostgreSQL connection pooling via PgBouncer.
- **Data tiering:** TimescaleDB compression reduces storage by 90-95% for historical data. Neo4j can archive resolved anomalies and closed tickets to a cold store.

## Migration Path

If the polyglot complexity proves too high for the team, the graph layer can be simplified by replacing Neo4j with Apache AGE (a PostgreSQL extension that adds Cypher/openCypher support), consolidating the graph and transactional stores into a single PostgreSQL instance while retaining TimescaleDB for time-series. This reduces the architecture to two PostgreSQL instances (one with AGE, one with TimescaleDB) or even a single instance with both extensions, at the cost of some graph query performance. Alternatively, if the project starts with Suggestion 1 or 3 and graph queries become a bottleneck, Neo4j can be introduced incrementally as a read-optimized projection of the asset knowledge graph, synced via Kafka.
