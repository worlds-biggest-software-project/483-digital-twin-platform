# Digital Twin Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source platform that creates continuously synchronised 3D virtual replicas of physical assets, integrating live sensor streams with spatial models to enable simulation, anomaly detection, and predictive maintenance.

Digital Twin Platform gives operators, engineers, and facility managers a spatially and temporally coherent model of their physical assets -- factories, buildings, power grids, vehicles -- rather than disconnected dashboards of isolated metrics. It ingests IoT sensor data in real time, renders it against 3D geometry, and supports simulation and what-if analysis on the virtual replica instead of the physical object.

---

## Why Digital Twin Platform?

- **No open-source platform covers the full stack.** No existing offering combines high-fidelity 3D visualization, real-time IoT ingest, physics simulation, and maintenance workflows in a single open-source product. Eclipse Ditto handles device state but has no visualization or simulation. NVIDIA Omniverse has best-in-class rendering but no native IoT ingest or maintenance workflows.
- **Enterprise pricing locks out SMEs.** Every capable platform -- Siemens Xcelerator, PTC ThingWorx, Ansys Twin Builder, IBM Maximo -- requires enterprise-tier licensing. There is no cost-effective digital twin tooling accessible to small and mid-sized manufacturers or building operators.
- **Vendor lock-in fragments the ecosystem.** Azure Digital Twins ties users to Azure services; AWS IoT TwinMaker requires Lambda-backed connectors and IoT Core; Siemens delivers full value only within the Siemens equipment ecosystem. An open platform with standard protocols (MQTT, OPC-UA, OpenUSD) avoids these constraints.
- **Semantic interoperability is fragmented.** No platform implements all the relevant ontologies (RealEstateCore for buildings, Asset Administration Shell for Industry 4.0, ISO 81346 for industrial assets). Cross-domain queries remain impossible without one-off integration work.
- **AI-native anomaly detection with explainability does not exist.** All reviewed platforms either lack built-in anomaly detection or treat it as a black box. Operators need causal attribution linking sensor deviations to specific components, not opaque alerts.

---

## Key Features

### Asset Model Ingestion and 3D Visualization

- Import 3D models from CAD, BIM (IFC/Revit), point-cloud scans (LiDAR/photogrammetry), and OpenUSD format
- Auto-generate simplified collision and LOD meshes for performant web rendering
- WebGL/WebGPU 3D viewport with tile-based streaming (OGC 3D Tiles) for industrial-scale scenes
- No-code dashboard builder embedded alongside the 3D viewer for non-technical operators

### IoT Data Integration and Real-Time Sync

- Connect to MQTT, OPC-UA, REST, and Kafka streams with configurable sensor-to-ontology mapping
- Normalise heterogeneous sensor schemas via a semantic mapping layer aligned to ISO 81346, RealEstateCore, and Asset Administration Shell standards
- Continuously update the 3D model's visual state (colour coding, animated components, live metric overlays) as sensor values change
- Edge agent for local data aggregation, filtering, and offline operation before cloud synchronisation

### Historical Playback and Time-Series Query

- Scrub backward in time to replay any past asset state
- Correlate sensor timelines with maintenance events and production records
- Time-series database backend (InfluxDB, TimescaleDB, or AWS Timestream) for high-frequency ingestion

### Simulation and What-If Analysis

- Run physics-based or data-driven simulations on the virtual twin to predict outcomes of operational changes
- Plugin architecture for custom simulation backends (FMU/FMI standard, Python callable)
- Clear labelling of simulation outputs as predictive estimates, with explicit user acknowledgement before simulation-derived actions trigger physical changes

### Anomaly Detection, Alerting, and Maintenance

- ML models trained on historical sensor data surface deviations from expected behaviour
- Explainable causal attribution linking anomalies to specific components
- Route alerts to maintenance ticketing systems, SCADA consoles, or webhook/email endpoints
- Digital thread linking each virtual component to procurement records, maintenance history, and compliance documents

### Access Control and Multi-Tenancy

- Asset hierarchy management (fleet / site / equipment levels) with DTDL or AAS-compatible data model
- Role-based access control scoped to individual assets or asset groups
- REST API with OpenAPI specification and Python/JavaScript SDKs
- Spatial querying API: query the model by geometry, asset type, or operational zone

---

## AI-Native Advantage

The platform uses AI to solve problems that incumbents leave to manual configuration or ignore entirely. Automated sensor schema mapping uses LLM-based field matching to align raw sensor tags to standard ontologies, eliminating the per-asset integration work that is the most common deployment failure point. A natural language spatial query interface lets operators ask questions like "show me all pumps running above 80% capacity near the cooling tower" instead of writing structured queries. Anomaly detection goes beyond threshold alerts to provide explainable causal attribution, and generative what-if scenario suggestion proposes simulations based on maintenance history patterns rather than requiring engineers to manually configure each scenario.

---

## Tech Stack and Deployment

- **Deployment modes:** Self-hosted (Docker Compose, Kubernetes/Helm), cloud, or hybrid with edge agents for constrained environments
- **3D rendering:** Three.js or Babylon.js with instanced mesh rendering and frustum culling; OGC 3D Tiles for streaming large facilities; optional NVIDIA Omniverse Cloud API integration for photorealistic rendering
- **Data layer:** Time-series database (InfluxDB/TimescaleDB) for sensor ingestion; graph or document store for asset hierarchy and digital thread; Apache Kafka or Kinesis for stream decoupling
- **Standards:** OpenUSD (Apache 2.0) for scene composition; DTDL (MIT) for ontology modelling; FMI/FMU (BSD-like) for simulation interoperability; OPC-UA and MQTT for industrial connectivity; Asset Administration Shell (IEC 63278) for Industry 4.0 alignment
- **Open-source foundations:** Eclipse Ditto (EPL-2.0) for device twin state management; iTwin.js (MIT) as a candidate viewer library; all key dependencies use permissive or weak-copyleft licences

---

## Market Context

The digital twin market is projected to grow from USD 24.5 billion in 2025 to USD 259.3 billion by 2032, with the smart-city segment alone expected to reach $3.77 billion by 2026 (ABI Research). Every capable incumbent -- NVIDIA Omniverse, Siemens Xcelerator, PTC ThingWorx, Ansys Twin Builder, IBM Maximo -- requires enterprise-tier pricing or cloud consumption fees that exclude small and mid-sized organisations. Primary buyers are operations and maintenance teams in manufacturing, infrastructure, energy, and smart building management.

---

## Project Status

> This project is in the **research and specification phase**.
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
