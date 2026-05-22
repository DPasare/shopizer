# ShopizerApp — Documentation Index

> **Generated**: Exhaustive scan by BMAD Document Project workflow
> **Project Root**: `shopizer-v1.1.5/`
> **Project Type**: Monolithic Java enterprise application (e-commerce platform)

---

## Core Documentation

| Document | Description |
|---|---|
| [Project Overview](./project-overview.md) | Executive summary, technology stack, key features, known issues |
| [Source Tree Analysis](./source-tree-analysis.md) | Complete directory tree, critical folders, entry points, package structure |
| [Architecture](./architecture.md) | System architecture, component hierarchy, design patterns, security & performance, technical debt |

## Technical Reference

| Document | Description |
|---|---|
| [Integration Architecture](./integration-architecture.md) | Module communication, web services, database integration, modernization recommendations |

## Development

| Document | Description |
|---|---|
| [Development Guide](./development-guide.md) | Setup, Ant build system, configuration, common tasks, troubleshooting, modernization |

---

## Quick Reference

### Technology Stack

| Layer | Technology |
|---|---|
| **Backend** | Java Enterprise (1.5.0+) |
| **Web Framework** | Apache Struts (legacy) |
| **Dependency Injection** | Spring Framework (legacy) |
| **ORM** | Hibernate (legacy) |
| **Build System** | Apache Ant |
| **Web Services** | Apache Axis, JAX-WS (SOAP) |
| **Database** | Hibernate ORM (vendor-agnostic) |

### Module Structure

| Module | Purpose |
|---|---|
| **sm-core** | Core business logic and entity management |
| **sm-central** | Web management interface and actions |

### Key Packages

| Package | Purpose |
|---|---|
| `com.salesmanager.core` | Core business logic and services |
| `com.salesmanager.central` | Web management interface |
| `com.salesmanager.central.cart` | Shopping cart management |
| `com.salesmanager.central.catalog` | Product catalog management |

### Key Findings

1. **Legacy Java Architecture** — Built for Java 1.5.0 (20+ years old)
2. **Deprecated Frameworks** — Old Struts, Spring, Hibernate versions
3. **Ant Build System** — No modern Maven/Gradle dependency management
4. **SOAP Web Services** — Using Apache Axis (deprecated)
5. **Windows-specific Paths** — Hard-coded Windows paths in build.xml
6. **No Test Framework** — No visible unit tests or testing framework
7. **XML-heavy Configuration** — Extensive XML configuration files
8. **Eclipse Project Structure** — Includes Eclipse IDE configuration

### Critical Technical Debt

- **TD-01**: Java 1.5.0 — Security vulnerabilities, compatibility issues
- **TD-02**: Legacy Frameworks — Security risks, missing features
- **TD-03**: Ant Build System — Dependency management issues
- **TD-04**: Apache Axis — Deprecated SOAP framework
- **TD-05**: No Modern Testing — Quality assurance issues

### Modernization Priority

1. **High Priority**: Java version upgrade, framework modernization
2. **Medium Priority**: Build system migration (Ant → Maven), REST API addition
3. **Low Priority**: Containerization, CI/CD pipeline, test framework addition
