# Standards & API Reference

> Project: Digital Twin Platform · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

**ISO 23247-1:2021 — Digital Twin Framework for Manufacturing, Part 1: Overview and General Principles**
- URL: https://www.iso.org/standard/75066.html
- Defines the core vocabulary, scope, and high-level architecture for digital twins in manufacturing contexts. Establishes that a digital twin must comprise a digital representation of a manufacturing element synchronised with its physical counterpart, and partitions a twinning system into user, digital twin, and device communication domains.

**ISO 23247-2:2021 — Digital Twin Framework for Manufacturing, Part 2: Reference Architecture**
- URL: https://www.iso.org/standard/78743.html
- Defines the reference architecture for a manufacturing digital twin system, specifying the four architectural layers (device, digital twin, synchronisation, and user domains) and the interfaces between them. Required reading for any platform targeting the manufacturing vertical.

**ISO 23247-3:2021 — Digital Twin Framework for Manufacturing, Part 3: Digital Representation of Manufacturing Elements**
- URL: https://www.iso.org/standard/78744.html
- Specifies the information model for representing manufacturing elements digitally, including geometric, kinematic, material, process, and status attributes. Directly relevant to the platform's asset model and ontology layer.

**ISO/FDIS 23247-6 — Digital Twin Framework for Manufacturing, Part 6: Digital Twin Composition**
- URL: https://www.iso.org/standard/87426.html
- Forthcoming standard defining how to compose complex digital twins from simpler sub-twins. Relevant to multi-asset and system-of-systems use cases such as a complete factory or smart-city deployment.

**ISO/IEC 81346 (IEC 81346) — Reference Designation System (RDS)**
- URL: https://www.81346.com/rds-81346
- Provides unambiguous, hierarchical identifier codes for industrial systems, installations, and equipment across functional, product, and location aspects. Used as a semantic anchor to link sensor tags, maintenance records, and 3D model components to the same physical element. Industry-specific sub-standards include RDS-PS (power supply / energy), RDS-PP (power plant), and RDS-CW (construction).

**IEC 62541 (OPC UA) — OPC Unified Architecture**
- URL: https://opcfoundation.org/
- Cross-platform, open-source standard (also IEC 62541) for secure, reliable data exchange from industrial sensors and controllers to cloud applications. Defines both client-server and publish-subscribe communication patterns with built-in authentication (X.509), encryption, and access control. The dominant protocol for operational technology (OT) data egress to digital twin platforms. Complementary to MQTT; OPC-UA carries rich semantic context while MQTT carries lightweight telemetry.

**IEC 63278 — Asset Administration Shell (AAS)**
- URL: https://www.plattform-i40.de/IP/Redaktion/EN/Standardartikel/specification-administrationshell.html
- The Industry 4.0 standard for representing a product or asset as a machine-readable digital twin encompassing its properties, capabilities, and documentation. AAS defines a standard API and information model that enables interoperability across vendor boundaries. Bosch IoT Suite and Siemens Xcelerator both support AAS; implementing it enables twin content portability.

---

### W3C & IETF Standards

**W3C Web of Things (WoT) Architecture 1.1**
- URL: https://w3c.github.io/wot-architecture/
- W3C Recommendation defining the architectural framework for IoT interoperability using web technologies. Defines WoT Thing Descriptions as the standard metadata format for IoT devices and their interaction affordances, applicable to the platform's device connectivity and ontology layers.

**W3C WoT Thing Description 1.1**
- URL: https://www.w3.org/TR/wot-thing-description11/
- W3C Recommendation specifying the JSON-LD format for describing IoT device capabilities (properties, actions, events) and their protocol bindings. Enables vendor-neutral device onboarding: a WoT-compliant sensor can describe itself to the twin platform without manual configuration.

**W3C WoT Discovery**
- URL: https://w3c.github.io/wot-discovery/
- W3C Recommendation specifying how WoT Thing Descriptions are registered and discovered, enabling automated asset onboarding in a digital twin deployment.

**W3C Web Ontology Language (OWL) / RDF**
- URL: https://www.w3.org/OWL/
- Foundational semantic web standards used by building ontologies (RealEstateCore, Building Topology Ontology, ifcOWL). Relevant when the platform's semantic layer needs to represent and reason over complex asset relationships.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Standard for compact, URL-safe token encoding of claims between parties. The baseline token format for the platform's authentication layer, used alongside OAuth 2.0 for API access control.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The standard authorisation protocol for REST API access. Required for multi-tenant deployments where clients and IoT edge agents need scoped, revocable credentials.

**MQTT v5.0 (OASIS Standard)**
- URL: https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html
- Lightweight publish-subscribe messaging protocol designed for constrained IoT devices and unreliable networks. The most widely deployed protocol for sensor-to-cloud telemetry ingest. MQTT v5 adds shared subscriptions, message expiry, and reason codes relevant to high-reliability twin synchronisation.

---

### Data Model & API Specifications

**Digital Twins Definition Language (DTDL) v3**
- URL: https://azure.github.io/opendigitaltwins-dtdl/DTDL/v3/DTDL.v3.html
- Microsoft open-source (MIT) JSON-LD-based language for describing digital twin models across any domain. Defines Interface, Component, Property, Relationship, Telemetry, and Command metamodel classes. Used natively by Azure Digital Twins; adopted by RealEstateCore and several Industrial ontologies. A strong candidate for the platform's model definition language.

**RealEstateCore Ontology (DTDL-based)**
- URL: https://github.com/azure/opendigitaltwins-building
- MIT-licensed DTDL ontology for smart buildings. Aligns with W3C Building Topology Ontology (BOT) and BRICK Schema. Provides standardised models for spaces, systems (HVAC, electrical, plumbing), equipment, and sensor streams. Directly usable for the platform's building-domain vertical.

**OpenAPI Specification (OAS) 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- Industry standard for describing REST APIs in a machine-readable YAML/JSON format. Required for the platform's public API documentation to enable SDK generation, API gateways, and developer onboarding.

**OGC 3D Tiles 1.1**
- URL: https://docs.ogc.org/cs/22-025r4/22-025r4.html
- OGC-approved open standard for streaming and rendering massive heterogeneous 3D geospatial datasets including photogrammetry, 3D buildings, BIM/CAD, instanced features, and point clouds. The standard format for tile-based streaming of large-scale 3D scenes in the platform's WebGL viewer.

**OpenUSD (Universal Scene Description)**
- URL: https://openusd.org/
- Pixar-originated, now open-source (Apache 2.0) scene description format adopted by NVIDIA Omniverse as the basis for photorealistic digital twin composition. Relevant if the platform integrates with Omniverse or provides high-fidelity 3D visualization. OpenUSD Exchange SDK is the official ingestion library.

**IFC (Industry Foundation Classes) — ISO 16739**
- URL: https://technical.buildingsmart.org/standards/ifc/
- buildingSMART International open standard for Building Information Modelling (BIM) data exchange. The primary format for importing building geometry and metadata. IFC4 and IFC4.3 are current versions; ifcOWL provides a semantic web representation for integration with RDF-based ontologies.

**FMI/FMU (Functional Mock-up Interface)**
- URL: https://fmi-standard.org/
- Modelica Association open standard (BSD-like licence) for exchanging simulation models as Functional Mock-up Units (FMUs). Enables the platform to execute physics-based simulation models exported from Ansys, Dymola, MATLAB/Simulink, and others, without embedding a specific simulation engine.

---

### Security & Authentication Standards

**OAuth 2.0 + OpenID Connect (OIDC)**
- URL: https://openid.net/connect/
- Standard identity and authorisation framework for multi-tenant SaaS platforms. OIDC adds authentication on top of OAuth 2.0, providing ID tokens with user identity claims. Required for enterprise SSO integration with Azure AD, Okta, and similar identity providers.

**X.509 Public Key Infrastructure**
- URL: https://datatracker.ietf.org/doc/html/rfc5280
- Standard for digital certificates used in device authentication (OPC-UA, MQTT TLS). IoT edge agents and OT-to-cloud data paths should authenticate with X.509 device certificates to prevent unauthorised sensor spoofing.

**OWASP API Security Top 10**
- URL: https://owasp.org/www-project-api-security/
- De-facto checklist for securing REST APIs. Particularly relevant items for a twin platform: Broken Object Level Authorization (BOLA) when scoping asset access, Broken Authentication for IoT device credentials, and Mass Assignment when accepting complex JSON payloads from devices.

**IEC 62443 — Industrial Cybersecurity**
- URL: https://www.iec.ch/iecstandards
- International standard series for cybersecurity of industrial automation and control systems (IACS). Defines security levels for OT networks and product development processes. Required for enterprise deployments in manufacturing, energy, and critical infrastructure where the twin platform ingests OT data.

**NIST Cybersecurity Framework (CSF) 2.0**
- URL: https://www.nist.gov/cyberframework
- US federal framework for managing cybersecurity risk, widely adopted in enterprise security programs. Relevant for positioning the platform to enterprise buyers in regulated industries.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://spec.modelcontextprotocol.io/
- Anthropic open-source protocol for connecting AI language models to external data sources and tools via a standardised server interface. A digital twin platform exposing an MCP server would allow AI agents to query current asset state, historical sensor data, and spatial relationships using natural language — directly enabling the AI-native query interface identified as a differentiating opportunity.

---

## Similar Products — Developer Documentation & APIs

### Azure Digital Twins
- **Description:** Microsoft cloud PaaS for creating graph-based digital twins of any environment using DTDL ontologies, with event routing to downstream Azure services.
- **API Documentation:** https://learn.microsoft.com/en-us/rest/api/azure-digitaltwins/
- **SDKs/Libraries:** .NET: `Azure.DigitalTwins.Core`; Java: `azure-digitaltwins-core`; JavaScript: `@azure/digital-twins-core`; Python: `azure-digitaltwins-core`
- **Developer Guide:** https://learn.microsoft.com/en-us/azure/digital-twins/
- **Standards:** REST/JSON, DTDL v3, OpenAPI 3.0, Azure AD (OIDC/OAuth 2.0)
- **Authentication:** Azure Active Directory (OAuth 2.0 + OIDC), managed identity

### AWS IoT TwinMaker
- **Description:** AWS service for building operational digital twins by federating data across existing AWS data stores without centralising it, with Grafana-based 3D visualization.
- **API Documentation:** https://docs.aws.amazon.com/iot-twinmaker/latest/apireference/
- **SDKs/Libraries:** AWS SDK for Python (Boto3), JavaScript/TypeScript, Java, .NET, Go
- **Developer Guide:** https://docs.aws.amazon.com/iot-twinmaker/latest/guide/what-is-twinmaker.html
- **Standards:** REST/JSON, AWS SigV4 auth, OpenAPI 3.0
- **Authentication:** AWS IAM (SigV4 request signing), IAM roles for service-to-service

### Bentley iTwin Platform
- **Description:** Open cloud platform providing APIs and the iTwin.js open-source library for developing digital twin applications for infrastructure assets.
- **API Documentation:** https://developer.bentley.com/apis/
- **SDKs/Libraries:** iTwin.js (TypeScript/JavaScript, MIT licence): https://github.com/iTwin/itwinjs-core
- **Developer Guide:** https://developer.bentley.com/itwinplatform/
- **Standards:** REST/JSON, OpenAPI 3.0, OGC 3D Tiles, IFC, OAuth 2.0
- **Authentication:** OAuth 2.0 (Bentley IMS identity provider), service app credentials

### Eclipse Ditto
- **Description:** Open-source IoT digital twin framework providing a REST/WebSocket API for device state management, fine-grained access control, and protocol-agnostic connectivity.
- **API Documentation:** https://eclipse.dev/ditto/httpapi-overview.html
- **SDKs/Libraries:** Java client: `org.eclipse.ditto:ditto-client`; JavaScript community client; direct HTTP/WebSocket usable from any language
- **Developer Guide:** https://eclipse.dev/ditto/
- **Standards:** REST/JSON, WebSocket, AMQP 0.9.1 / 1.0, MQTT 3/5, Apache Kafka
- **Authentication:** Basic auth, pre-authenticated (JWT-based via nginx/gateway), OAuth 2.0 (via external IDP integration)

### NVIDIA Omniverse (Cloud APIs)
- **Description:** Platform of APIs and SDKs for building photorealistic, physics-accurate industrial digital twin applications using OpenUSD as the scene format.
- **API Documentation:** https://docs.omniverse.nvidia.com/services/latest/
- **SDKs/Libraries:** Omniverse Kit SDK (Python/C++); OpenUSD Exchange SDK; Isaac Sim SDK
- **Developer Guide:** https://docs.nvidia.com/omniverse/index.html
- **Standards:** OpenUSD (Apache 2.0), REST microservices, NVIDIA PhysX
- **Authentication:** NVIDIA NGC API key, OAuth 2.0 for cloud services

### Siemens Building X (Smart Building APIs)
- **Description:** Siemens smart building operations platform with a documented REST API for managing building digital twin entities (portfolios, assets, users, operational tools).
- **API Documentation:** https://www.siemens.com/en-us/products/building-x/apis/
- **SDKs/Libraries:** REST-only; OpenAPI Spec (OAS) documentation provided for all endpoints
- **Developer Guide:** https://developer.siemens.com/building-x/
- **Standards:** REST/JSON, OpenAPI 3.0, OAuth 2.0
- **Authentication:** OAuth 2.0 (Siemens identity provider)

### Ansys Twin Builder / TwinAI
- **Description:** Physics-simulation digital twin platform for building high-fidelity reduced-order models and deploying them as FMUs to IIoT platforms for predictive maintenance.
- **API Documentation:** https://www.ansys.com/products/digital-twin/ansys-twin-builder
- **SDKs/Libraries:** Python scripting API for Twin Builder; FMU export compatible with any FMI 2.0/3.0 host
- **Developer Guide:** https://ansyshelp.ansys.com/ (requires licence)
- **Standards:** FMI/FMU 2.0 and 3.0 (Modelica Association), Modelica language, REST connectors for IIoT platforms
- **Authentication:** Licence-server based; FMU deployment is standalone executable

### PTC ThingWorx REST API
- **Description:** IIoT rapid application development platform with a REST API for reading and writing digital twin (Thing) properties, services, and events.
- **API Documentation:** https://support.ptc.com/help/thingworx/platform/r9.6/en/
- **SDKs/Libraries:** ThingWorx Java SDK, .NET SDK, REST over HTTP from any language
- **Developer Guide:** https://support.ptc.com/help/thingworx/
- **Standards:** REST/JSON, OData, WebSocket; Kepware OPC-UA, Modbus, PROFINET connectors
- **Authentication:** API key (application key), Basic auth; OAuth 2.0 available via SSO integration

---

## Notes

- **Asset Administration Shell (AAS) / IEC 63278**: AAS is the emerging interoperability standard for Industry 4.0 twin content exchange. Bosch and Siemens are early adopters. AAS defines a standard API (Part 2) and information model (Part 3) that could be adopted as the platform's canonical data model for manufacturing use cases. The Eclipse AAS4J library (Apache 2.0) provides a Java reference implementation.
  - AAS specification: https://industrialdigitaltwin.org/en/content-hub/aasspecification

- **Digital Twin Consortium (DTC) Glossary and Reference Architecture**: The DTC, a subsidiary of the Object Management Group (OMG), publishes reference architecture documents and a glossary that inform interoperability requirements.
  - URL: https://www.digitaltwinconsortium.org/

- **BRICK Schema**: An open-source ontology (BSD-3-Clause) for describing HVAC, electrical, and plumbing systems in buildings, complementary to RealEstateCore and W3C BOT.
  - URL: https://brickschema.org/

- **Emerging gap — simulation interoperability**: No single open standard covers the full simulation-to-twin pipeline. FMI/FMU covers model portability; SysML v2 (OMG, 2024) is emerging for system modelling; but linking simulation results back to the twin's live state in a standardised way remains an open problem. A platform that solves this with an open API could define the emerging standard.
