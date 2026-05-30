# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Approach

An event-sourced architecture with Command Query Responsibility Segregation (CQRS). Instead of storing the current state of each entity, the system persists an immutable log of every domain event that has ever occurred. Current state is derived by replaying events. Read-optimised projections are maintained separately for query workloads.

This model uses PostgreSQL as the event store (for durability and transactional guarantees) and Apache Kafka as the event bus for distributing events to projection builders and external consumers.

## Why This Suits a Digital Twin Platform

A digital twin is fundamentally about maintaining a virtual replica that faithfully reflects the history and current state of a physical asset. Event sourcing is a natural fit because:

1. **Historical playback is built in.** The README calls for scrubbing backward in time to replay any past asset state. With event sourcing, this is a first-class capability -- replay events up to any timestamp to reconstruct the twin's state at that moment.
2. **Audit trail for compliance.** The digital thread linking components to maintenance, procurement, and compliance records benefits from an immutable event log that proves exactly when changes occurred.
3. **Causal attribution for anomalies.** Events form a causal chain. When an anomaly is detected, the system can trace back through events to identify the root cause.
4. **Decoupled consumers.** Different teams need different views -- 3D rendering needs spatial state, maintenance needs ticket state, analytics needs aggregated metrics. CQRS projections serve each consumer optimally.

**Trade-off:** Event sourcing adds complexity. Developers must think in terms of events and projections rather than CRUD. Schema evolution of events requires careful versioning. The event store grows continuously and requires archival/compaction strategies.

---

## Event Store Schema

```sql
-- ============================================================
-- CORE EVENT STORE
-- ============================================================

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       VARCHAR(512) NOT NULL,   -- e.g., "asset::{uuid}" or "ticket::{uuid}"
    stream_type     VARCHAR(100) NOT NULL,    -- aggregate type: Asset, Sensor, Ticket, etc.
    event_type      VARCHAR(255) NOT NULL,    -- e.g., "AssetRegistered", "SensorReadingReceived"
    event_version   INT NOT NULL,             -- schema version for this event type
    sequence_number BIGINT NOT NULL,          -- order within stream (optimistic concurrency)
    tenant_id       UUID NOT NULL,
    correlation_id  UUID,                     -- links related events across streams
    causation_id    UUID,                     -- the event or command that caused this event
    payload         JSONB NOT NULL,           -- event data
    metadata        JSONB NOT NULL DEFAULT '{}',  -- actor, ip, source system
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_number)
);

-- Partitioned by month for manageability at scale
-- In production, use declarative partitioning:
-- CREATE TABLE event_store (...) PARTITION BY RANGE (recorded_at);

CREATE INDEX idx_events_stream ON event_store(stream_id, sequence_number);
CREATE INDEX idx_events_type ON event_store(event_type, recorded_at);
CREATE INDEX idx_events_tenant_time ON event_store(tenant_id, recorded_at);
CREATE INDEX idx_events_correlation ON event_store(correlation_id);

-- Snapshot table for performance: avoid replaying entire stream
CREATE TABLE event_snapshots (
    stream_id       VARCHAR(512) NOT NULL,
    snapshot_version BIGINT NOT NULL,        -- sequence_number at snapshot time
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);
```

---

## Domain Events Catalog

### Asset Lifecycle Events

```json
{
  "event_type": "AssetRegistered",
  "event_version": 1,
  "payload": {
    "asset_id": "uuid",
    "tenant_id": "uuid",
    "asset_type": "equipment",
    "name": "Pump P-101",
    "parent_asset_id": "uuid|null",
    "serial_number": "SN-12345",
    "iso_81346_ref": "=A1.M01",
    "dtdl_model_id": "dtmi:com:example:Pump;1",
    "geo_location": { "lat": 51.5074, "lng": -0.1278, "alt": 12.5 }
  }
}

{
  "event_type": "AssetRelocated",
  "event_version": 1,
  "payload": {
    "asset_id": "uuid",
    "new_parent_asset_id": "uuid",
    "new_geo_location": { "lat": 51.51, "lng": -0.13, "alt": 12.5 },
    "reason": "Site reorganisation"
  }
}

{
  "event_type": "AssetDecommissioned",
  "event_version": 1,
  "payload": {
    "asset_id": "uuid",
    "reason": "End of service life",
    "replacement_asset_id": "uuid|null"
  }
}
```

### Sensor and Telemetry Events

```json
{
  "event_type": "SensorProvisioned",
  "event_version": 1,
  "payload": {
    "sensor_id": "uuid",
    "asset_id": "uuid",
    "sensor_tag": "PUMP_P101_TEMP",
    "unit_of_measure": "celsius",
    "data_type": "float",
    "protocol": "opcua",
    "sampling_interval_ms": 1000,
    "ontology_mapping": "dtmi:com:example:Pump;1:temperature"
  }
}

{
  "event_type": "SensorReadingBatchReceived",
  "event_version": 1,
  "payload": {
    "sensor_id": "uuid",
    "readings": [
      { "timestamp": "2026-05-26T10:00:00Z", "value": 72.3 },
      { "timestamp": "2026-05-26T10:00:01Z", "value": 72.5 }
    ]
  }
}

{
  "event_type": "ConnectorStatusChanged",
  "event_version": 1,
  "payload": {
    "connector_id": "uuid",
    "protocol": "mqtt",
    "old_status": "connected",
    "new_status": "error",
    "error_message": "Connection refused"
  }
}
```

### 3D Model Events

```json
{
  "event_type": "ModelFileUploaded",
  "event_version": 1,
  "payload": {
    "model_file_id": "uuid",
    "asset_id": "uuid",
    "format": "ifc",
    "storage_url": "s3://models/abc.ifc",
    "file_size_bytes": 52428800,
    "checksum_sha256": "a1b2c3..."
  }
}

{
  "event_type": "SpatialAnchorSet",
  "event_version": 1,
  "payload": {
    "anchor_id": "uuid",
    "asset_id": "uuid",
    "transform_matrix": [1,0,0,0, 0,1,0,0, 0,0,1,0, 10,20,0,1],
    "coordinate_system": "site_relative"
  }
}
```

### Anomaly and Maintenance Events

```json
{
  "event_type": "AnomalyDetected",
  "event_version": 1,
  "payload": {
    "event_id": "uuid",
    "model_id": "uuid",
    "asset_id": "uuid",
    "sensor_id": "uuid",
    "severity": "critical",
    "title": "Temperature spike on Pump P-101",
    "description": "Temperature exceeded 3-sigma threshold for 5 minutes",
    "causal_component_id": "uuid",
    "confidence": 0.92
  }
}

{
  "event_type": "MaintenanceTicketCreated",
  "event_version": 1,
  "payload": {
    "ticket_id": "uuid",
    "asset_id": "uuid",
    "anomaly_event_id": "uuid|null",
    "title": "Inspect Pump P-101 bearing",
    "priority": "high"
  }
}

{
  "event_type": "MaintenanceTicketResolved",
  "event_version": 1,
  "payload": {
    "ticket_id": "uuid",
    "resolved_by": "uuid",
    "resolution_notes": "Bearing replaced, vibration levels normalised",
    "parts_used": ["BRG-7204B"]
  }
}
```

### Simulation Events

```json
{
  "event_type": "SimulationRunStarted",
  "event_version": 1,
  "payload": {
    "run_id": "uuid",
    "simulation_def_id": "uuid",
    "target_asset_id": "uuid",
    "input_parameters": { "flow_rate": 120, "ambient_temp": 35 },
    "backend_type": "fmu"
  }
}

{
  "event_type": "SimulationRunCompleted",
  "event_version": 1,
  "payload": {
    "run_id": "uuid",
    "result_summary": { "predicted_failure_in_hours": 168, "confidence": 0.87 },
    "requires_acknowledgement": true
  }
}
```

---

## Command Handlers (Pseudocode)

```python
class AssetCommandHandler:
    def handle_register_asset(self, cmd: RegisterAsset) -> list[Event]:
        # Validate: no duplicate serial_number in tenant
        # Validate: parent_asset_id exists if provided
        return [AssetRegistered(
            asset_id=uuid4(),
            tenant_id=cmd.tenant_id,
            asset_type=cmd.asset_type,
            name=cmd.name,
            parent_asset_id=cmd.parent_asset_id,
            serial_number=cmd.serial_number,
            iso_81346_ref=cmd.iso_81346_ref,
            dtdl_model_id=cmd.dtdl_model_id,
            geo_location=cmd.geo_location,
        )]

    def handle_decommission_asset(self, cmd: DecommissionAsset) -> list[Event]:
        state = self.load_aggregate(cmd.asset_id)
        if state.status == 'decommissioned':
            raise AssetAlreadyDecommissioned(cmd.asset_id)
        # Check no active child assets
        return [AssetDecommissioned(
            asset_id=cmd.asset_id,
            reason=cmd.reason,
            replacement_asset_id=cmd.replacement_asset_id,
        )]


class MaintenanceCommandHandler:
    def handle_create_ticket(self, cmd: CreateMaintenanceTicket) -> list[Event]:
        return [MaintenanceTicketCreated(
            ticket_id=uuid4(),
            asset_id=cmd.asset_id,
            anomaly_event_id=cmd.anomaly_event_id,
            title=cmd.title,
            priority=cmd.priority,
        )]

    def handle_resolve_ticket(self, cmd: ResolveTicket) -> list[Event]:
        state = self.load_aggregate(cmd.ticket_id)
        if state.status in ('resolved', 'closed'):
            raise TicketAlreadyResolved(cmd.ticket_id)
        return [MaintenanceTicketResolved(
            ticket_id=cmd.ticket_id,
            resolved_by=cmd.user_id,
            resolution_notes=cmd.notes,
            parts_used=cmd.parts_used,
        )]
```

---

## Read Projections

### Projection: Asset Current State

```sql
CREATE TABLE proj_assets (
    asset_id        UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    asset_type      VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    parent_asset_id UUID,
    serial_number   VARCHAR(255),
    status          VARCHAR(50) NOT NULL,
    geo_latitude    DOUBLE PRECISION,
    geo_longitude   DOUBLE PRECISION,
    last_event_seq  BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_assets_tenant ON proj_assets(tenant_id);
CREATE INDEX idx_proj_assets_parent ON proj_assets(parent_asset_id);
```

### Projection: Asset Hierarchy (Materialized Closure)

```sql
CREATE TABLE proj_asset_hierarchy (
    ancestor_id     UUID NOT NULL,
    descendant_id   UUID NOT NULL,
    depth           INT NOT NULL,
    PRIMARY KEY (ancestor_id, descendant_id)
);
```

### Projection: Sensor Latest Values

```sql
CREATE TABLE proj_sensor_latest (
    sensor_id       UUID PRIMARY KEY,
    asset_id        UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    sensor_tag      VARCHAR(255) NOT NULL,
    latest_value    DOUBLE PRECISION,
    latest_timestamp TIMESTAMPTZ,
    status          VARCHAR(50) NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_sensor_asset ON proj_sensor_latest(asset_id);
```

### Projection: Active Anomalies Dashboard

```sql
CREATE TABLE proj_active_anomalies (
    event_id        UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    asset_id        UUID NOT NULL,
    asset_name      VARCHAR(255),
    sensor_tag      VARCHAR(255),
    severity        VARCHAR(20) NOT NULL,
    title           VARCHAR(512) NOT NULL,
    causal_component_name VARCHAR(255),
    confidence      DOUBLE PRECISION,
    detected_at     TIMESTAMPTZ NOT NULL,
    is_resolved     BOOLEAN NOT NULL DEFAULT FALSE,
    ticket_id       UUID
);

CREATE INDEX idx_proj_anomalies_tenant ON proj_active_anomalies(tenant_id, is_resolved, severity);
```

### Projection: Maintenance Board

```sql
CREATE TABLE proj_maintenance_board (
    ticket_id       UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    asset_id        UUID NOT NULL,
    asset_name      VARCHAR(255),
    title           VARCHAR(512) NOT NULL,
    priority        VARCHAR(20) NOT NULL,
    status          VARCHAR(50) NOT NULL,
    assigned_to_name VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL,
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_proj_maint_tenant ON proj_maintenance_board(tenant_id, status);
```

---

## Trade-offs

| Dimension | Assessment |
|-----------|-----------|
| **Historical playback** | Excellent. First-class capability -- replay events to any point in time. |
| **Audit and compliance** | Excellent. Immutable event log is a complete audit trail. |
| **Complexity** | High. Requires event versioning, projection rebuilding, and eventual consistency handling. |
| **Query performance** | Good for reads via projections, but projections must be explicitly designed for each query pattern. |
| **Storage growth** | High. Event store grows monotonically. Requires archival, compaction, or tiered storage. |
| **Schema evolution** | Moderate. Event upcasting and versioning are needed; old events must remain readable forever. |
| **Debugging** | Good. Every state change is traceable, but understanding current state requires following event chains. |
| **Team familiarity** | Low in most organizations. Event sourcing is less mainstream than CRUD. |

## Scalability Considerations

- Partition the event store by `recorded_at` (monthly or weekly) for manageability.
- Use snapshots every N events per stream (e.g., every 100) to avoid replaying long streams.
- Kafka topics per aggregate type enable parallel projection building.
- Projections can be rebuilt from scratch by replaying all events -- useful after schema changes.
- Consider an event archival policy: move events older than N years to cold storage (S3/GCS) while retaining snapshots.

## Migration Path

This model pairs naturally with a time-series database (InfluxDB/TimescaleDB) for raw sensor telemetry, where `SensorReadingBatchReceived` events are the write path but high-frequency queries go directly to the time-series store. If the event-sourcing overhead proves excessive for simple CRUD entities (dashboards, user preferences), those can be carved out into a traditional relational model while retaining event sourcing for the core domain (assets, sensors, maintenance, anomalies).
