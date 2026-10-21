# WSO2 Micro Integrator - Project Architecture

## Overview

**WSO2 Micro Integrator (MI)** is an open-source, lightweight integration runtime built on top of the WSO2 Carbon platform. It provides a config-driven approach to integrate applications, services, data, and cloud systems. The project is designed for both centralized ESB-style and decentralized microservices-style deployments, with first-class support for Docker/Kubernetes.

- **Language:** Java
- **Build System:** Apache Maven (multi-module POM)
- **Runtime Framework:** Eclipse Equinox OSGi
- **Mediation Engine:** Apache Synapse
- **Version:** 4.7.0-SNAPSHOT
- **License:** Apache License 2.0
- **JDK Requirement:** JDK 21 (build), JDK 21 or 25 (tests)

## Key Technologies

| Technology | Purpose |
|---|---|
| **Java** | Primary language |
| **Apache Maven** | Build and dependency management |
| **Eclipse Equinox (OSGi)** | Modular runtime container; bundles loaded dynamically |
| **Apache Synapse** | Mediation engine for message routing, transformation, and protocol switching |
| **Apache Axis2** | SOAP/REST transport framework |
| **WSO2 Carbon Kernel** | Base platform providing server lifecycle, configuration, and service registry |
| **Docker** | Container image distribution (Alpine, CentOS, Ubuntu variants) |
| **Log4j2** | Logging framework |
| **BouncyCastle** | Cryptography (JCE/JSSE providers) |
| **TOML (deployment.toml)** | Primary configuration format |

## Directory Structure

```
product-integrator-mi/
├── pom.xml                      # Root parent POM (multi-module build)
├── components/                  # Core OSGi bundles (the main source code)
│   ├── org.wso2.micro.integrator.bootstrap/      # JVM bootstrap & classloader setup
│   ├── org.wso2.micro.integrator.server/          # Main entry point (main() method)
│   ├── org.wso2.micro.integrator.core/            # Core server initializer & OSGi activator
│   ├── org.wso2.micro.integrator.initializer/     # Synapse mediation engine initializer
│   ├── org.wso2.micro.integrator.extensions/      # Management APIs & health probes
│   │   ├── org.wso2.micro.integrator.management.apis/  # REST management APIs
│   │   └── org.wso2.micro.integrator.probes/            # Liveness/readiness probes
│   ├── org.wso2.micro.integrator.coordination/    # Cluster coordination
│   ├── org.wso2.micro.integrator.security/        # Security handler components
│   ├── org.wso2.micro.integrator.utils/           # Shared utilities
│   ├── org.wso2.micro.integrator.log4j2.plugins/  # Custom Log4j2 plugins
│   ├── org.wso2.micro.integrator.logging.updater/ # Dynamic logging configuration
│   ├── org.wso2.micro.integrator.osgi.security/   # OSGi security manager
│   ├── org.wso2.carbon.securevault/               # Secure vault for secrets
│   ├── javax.cache/                               # JSR-107 caching
│   ├── crypto-service/                            # Encryption/decryption services
│   ├── data/data-services/                        # Data services: core, OData, SQL driver, deployer
│   ├── mediation/                                 # Mediation layer (largest subsystem)
│   │   ├── connectors/         # Connector framework
│   │   ├── data-publishers/    # Analytics & observability publishers
│   │   ├── extensions/         # Security & transport handler extensions
│   │   ├── inbound-endpoints/  # Inbound endpoint framework (polling, listening)
│   │   ├── mediators/          # Built-in mediators (cache, datamapper, XSLT, OAuth, etc.)
│   │   ├── mediation-commons/  # Shared mediation utilities
│   │   ├── mediation-diagnostics/ # Diagnostic tools
│   │   ├── message-relay/      # Message relay & lazy building
│   │   ├── registry/           # MI registry implementation
│   │   ├── security/           # Mediation-level security
│   │   ├── startup/            # Mediation startup sequences
│   │   ├── tasks/              # Scheduled task framework (ntask)
│   │   └── transports/         # WebSocket and custom transports
│   ├── org.wso2.micro.integrator.ndatasource.*/   # Named datasources (core, common, RDBMS, deployer)
│   └── business-adaptors/                         # Business protocol adaptors (HL7)
│
├── features/                    # OSGi P2 feature definitions (packaging bundles for provisioning)
├── p2-profile/                  # Carbon P2 profile generation (assembles features into a runtime)
│
├── distribution/                # Final distribution assembly
│   └── src/
│       ├── assembly/            # Maven assembly descriptors
│       ├── conf/                # Config (deployment.toml, synapse-configs, datasources, security)
│       ├── scripts/             # Shell/bat startup scripts
│       ├── dbscripts/           # DB schemas (MySQL, PostgreSQL, Oracle, MSSQL, DB2)
│       ├── docker-distribution/ # Dockerfiles (Alpine, CentOS, Ubuntu)
│       └── resources/           # Config tool, CApps, security resources
│
├── integration/                 # Integration test suites
├── product-scenarios/           # End-to-end product scenario tests
├── performance/                 # Performance test configurations
├── migration/                   # Migration tools for version upgrades
├── samples/                     # Sample integration projects
├── doc/                         # Architecture diagrams
└── .github/workflows/           # CI: JDK 17/21, smoke, cluster, compatibility, Windows, coverage
```

## Entry Point and Startup Flow

### 1. Bootstrap (`Bootstrap.main()`)

`components/org.wso2.micro.integrator.bootstrap/src/main/java/org/wso2/micro/integrator/bootstrap/Bootstrap.java`

- The JVM's initial entry point, invoked by the startup scripts.
- Loads system properties from `carbon.properties`.
- Builds a `URLClassLoader` from the JARs in `lib/`.
- Loads and invokes `Main.main()` via reflection through that classloader.

### 2. Main (`Main.main()`)

`components/org.wso2.micro.integrator.server/src/main/java/org/wso2/micro/integrator/server/Main.java`

- Sets system properties (`CARBON_HOME`, `AXIS2_HOME`, logging paths, etc.).
- Parses command-line arguments.
- Loads and processes `deployment.toml` via `ConfigParser`.
- Installs BouncyCastle JCE/JSSE providers if configured.
- Runs startup extensions (patches, dropins bundle deployer).
- Launches the **Eclipse Equinox OSGi framework** via `EclipseStarter.startup()` using a `ChildFirstURLClassLoader`.

### 3. OSGi Activation Phase

#### 3a. Core Activator

`components/org.wso2.micro.integrator.core/src/main/java/org/wso2/micro/integrator/core/internal/Activator.java`

- OSGi `BundleActivator`; logs server metadata (JVM, OS, paths).
- Reads `carbon.xml` and registers `CarbonServerConfigurationService`.
- Initializes `CoreServerInitializer` and registers core services.

#### 3b. ServiceBusInitializer

`components/org.wso2.micro.integrator.initializer/src/main/java/org/wso2/micro/integrator/initializer/ServiceBusInitializer.java`

- Declarative Services `@Component`, activated once its `@Reference` dependencies (`Axis2ConfigurationContextService`, `CarbonServerConfigurationService`, `SecretCallbackHandlerService`, ...) are available.
- Initializes the **Apache Synapse** engine (`ServerManager`, `ServerContextInformation`).
- Registers `SynapseConfigurationService`, `SynapseEnvironmentService`, `SynapseRegistrationsService`.
- Sets up mediation persistence and Synapse handlers.

#### 3c. StartupFinalizer

`components/org.wso2.micro.integrator.initializer/src/main/java/org/wso2/micro/integrator/initializer/StartupFinalizer.java`

- Starts Axis2 transport listeners (HTTP/HTTPS) and inbound endpoint listeners.
- Marks the server as started and logs the startup time.

### Startup Sequence

```
JVM Start
  │
  ▼
Bootstrap.main()            ── load JARs, create classloader
  │
  ▼
Main.main()                 ── parse config, set up security, launch OSGi
  │
  ▼
Equinox OSGi Framework      ── resolve & activate bundles
  │
  ├─▶ Core Activator        ── register Carbon services, init server
  ├─▶ ServiceBusInitializer ── init Synapse engine, register mediation services
  ├─▶ Other components      ── data services, connectors, tasks, ...
  └─▶ StartupFinalizer      ── start transports, mark server ready
```

## Key Architectural Concepts

### OSGi Modular Architecture
Each component is an OSGi bundle. Bundles declare dependencies via `Import-Package` and communicate through the OSGi service registry rather than calling each other directly.

### Carbon Services
Infrastructure services registered by the Carbon kernel layer (configuration, Axis2 context, secret resolution, crypto) that higher-level components such as the mediation engine consume through OSGi dependency injection.

### Mediation Engine (Apache Synapse)
Messages flow through sequences of **mediators** (Log, Filter, PayloadFactory, DataMapper, Cache, ...). Proxy services, APIs, sequences, and endpoints are defined as Synapse XML artifacts, typically deployed as Carbon Applications (CApps).

### Data Services
`components/data/data-services/` exposes databases as REST/SOAP services through declarative configuration, including OData support.

### Inbound Endpoints
Polling and listening endpoints (HTTP, JMS, Kafka, File, ...) that receive messages and inject them directly into mediation sequences.

### Management APIs & Probes
REST management APIs for runtime introspection and control, plus liveness/readiness probes for Kubernetes.

### Configuration
- `deployment.toml` - primary TOML configuration, parsed at startup.
- `carbon.xml` - Carbon kernel configuration.
- `synapse-configs/` - Synapse mediation artifacts.

## Build and Distribution

```bash
mvn clean install                     # full build
mvn clean install -Ddocker.skip=false # also build the Docker image
```

Output: `distribution/target/wso2mi-<version>.zip`, containing `bin/` (startup scripts), `conf/` (configuration), `lib/` (runtime JARs), `repository/` (deployed artifacts), and `dbscripts/` (database schemas).
