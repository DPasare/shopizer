# ShopizerApp — Project Overview

## Executive Summary

ShopizerApp is a **Java-based e-commerce platform** built with Ant build system. It provides a comprehensive sales management system with multi-module architecture including core libraries, central management, and web interfaces. The project follows a traditional enterprise Java architecture with Struts, Spring, and Hibernate integration.

## Project Identity

| Field | Value |
|---|---|
| **Project Name** | ShopizerApp (shopizer-v1.1.5) |
| **Repository Type** | Monolithic Java application |
| **Primary Language** | Java |
| **Architecture** | Multi-module enterprise Java application |
| **Status** | Legacy e-commerce platform (v1.1.5) |

## Technology Stack Summary

| Category | Technology | Version |
|---|---|---|
| **Backend Framework** | Java Enterprise | Java 5+ (inferred from build.xml) |
| **Web Framework** | Apache Struts | Legacy version |
| **Dependency Injection** | Spring Framework | Legacy version |
| **ORM** | Hibernate | Legacy version |
| **Build System** | Apache Ant | Traditional Ant build |
| **Web Services** | Apache Axis, JAX-WS | Legacy SOAP services |
| **Database** | Hibernate ORM | Database-agnostic via Hibernate |

## Architecture Type

**Multi-module enterprise Java application**

- **Core Module**: Business logic and entity management (`sm-core`)
- **Central Module**: Web interface and management (`sm-central`) 
- **Web Module**: User-facing e-commerce interface
- **Integration Layer**: SOAP web services via Axis and JAX-WS

## Repository Structure

```
shopizer-v1.1.5/                    # Project root
├── sm-core/                        # ★ CORE BUSINESS LOGIC MODULE
│   ├── src/                        # Java source code
│   │   └── com/salesmanager/       # Package structure
│   │       ├── core/               # Core business logic
│   │       │   ├── entity/         # Entity classes (JPA/Hibernate)
│   │       │   ├── service/        # Business services
│   │       │   └── module/         # Module definitions
│   │       └── central/            # Central management interfaces
│   ├── conf/                       # Configuration files
│   │   ├── hibernate/              # Hibernate configuration
│   │   ├── properties/             # Application properties
│   │   └── spring/                 # Spring configuration
│   ├── lib/                        # External dependencies
│   │   ├── compile/                # Compile-time dependencies
│   │   ├── hibernate/              # Hibernate libraries
│   │   ├── spring/                 # Spring framework
│   │   ├── struts/                 # Struts framework
│   │   ├── axis/                   # Axis web services
│   │   └── jax-ws/                 # JAX-WS web services
│   └── build.xml                   # Ant build configuration
│
├── sm-central/                     # ★ CENTRAL MANAGEMENT MODULE
│   ├── src/com/salesmanager/       # Web interface source
│   │   ├── central/                # Central management actions
│   │   ├── cart/                   # Shopping cart management
│   │   └── catalog/                # Product catalog management
│   └── [configuration files]
│
└── [other modules]                 # Additional modules (web, etc.)
```

## Key Features

1. **Product Catalog Management** — CRUD operations for products, categories, attributes
2. **Shopping Cart** — Add/remove products, cart properties, integration
3. **Central Administration** — Web-based admin interface for store management
4. **Web Services** — SOAP-based integration via Axis and JAX-WS
5. **Multi-language Support** — Internationalization capabilities
6. **Database Abstraction** — Hibernate ORM for database independence

## Known Issues (from build.xml)

1. **Legacy Java Version** — Built for Java 1.5.0 (very outdated)
2. **Windows-specific Paths** — Hard-coded Windows paths in build configuration
3. **Legacy Framework Versions** — Using very old versions of Struts, Spring, Hibernate
4. **No Modern Build System** — Using Ant instead of Maven/Gradle
5. **Deprecated Web Services** — Using Axis instead of modern REST APIs

## Links to Detailed Documentation

- [Source Tree Analysis](./source-tree-analysis.md)
- [Architecture](./architecture.md)
- [Component Inventory](./component-inventory.md)
- [API Contracts](./api-contracts.md)
- [Development Guide](./development-guide.md)
- [Integration Architecture](./integration-architecture.md)
