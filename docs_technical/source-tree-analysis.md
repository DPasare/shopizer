# Source Tree Analysis — ShopizerApp

## Complete Directory Tree

```
shopizer-v1.1.5/                    # Project root
├── sm-core/                        # ★ CORE BUSINESS LOGIC MODULE
│   ├── .classpath                  # Eclipse classpath configuration (6520 bytes)
│   ├── .project                    # Eclipse project configuration (560 bytes)
│   ├── .settings/                  # Eclipse IDE settings
│   ├── build.xml                   # ★ Ant build configuration (3146 bytes)
│   ├── conf/                       # Configuration files
│   │   ├── META-INF/               # Manifest and metadata
│   │   ├── hibernate/              # Hibernate ORM configuration
│   │   ├── properties/             # Application properties
│   │   └── spring/                 # Spring framework configuration
│   ├── lib/                        # External dependencies
│   │   ├── compile/                # Compile-time dependencies
│   │   ├── hibernate/              # Hibernate libraries
│   │   ├── misc/                   # Miscellaneous libraries
│   │   ├── spring/                 # Spring framework
│   │   ├── struts/                 # Apache Struts
│   │   ├── axis/                   # Apache Axis (SOAP)
│   │   └── jax-ws/                 # JAX-WS web services
│   ├── src/                        # ★ Java source code
│   │   └── com/salesmanager/       # Package structure
│   │       ├── core/               # Core business logic
│   │       │   ├── entity/         # Entity classes (JPA/Hibernate)
│   │       │   ├── service/        # Business services
│   │       │   └── module/         # Module definitions
│   │       └── central/            # Central management interfaces
│   └── working/                    # Build working directory
│
├── sm-central/                     # ★ CENTRAL MANAGEMENT MODULE
│   ├── src/com/salesmanager/       # Web interface source
│   │   ├── central/                # Central management actions
│   │   │   ├── AuthorizationException.java
│   │   │   ├── BaseAction.java
│   │   │   ├── CentralBaseInterceptor.java
│   │   │   ├── CountrySelectBaseAction.java
│   │   │   └── PageBaseAction.java
│   │   ├── cart/                   # Shopping cart management
│   │   │   ├── AddProduct.java
│   │   │   ├── CartPropertiesAction.java
│   │   │   ├── IntegrationAction.java
│   │   │   └── SiteMapAction.java
│   │   └── catalog/                # Product catalog management
│   │       ├── EditCategoryAction.java
│   │       ├── EditDiscountAction.java
│   │       ├── EditImagesAction.java
│   │       ├── EditProductAction.java
│   │       ├── EditProductAttributesAction.java
│   │       ├── EditProductOptionsAction.java
│   │       ├── EditProductOptionsValuesAction.java
│   │       ├── EditProductPriceAction.java
│   │       ├── EditProductPriceDiscountAction.java
│   │       ├── EditProductUploadAction.java
│   │       ├── FieldMetaData.java
│   │       ├── FileUploadDirective.java
│   │       ├── ProductListAction.java
│   │       ├── ProductOptionDisplay.java
│   │       ├── ProductOptionValueDisplay.java
│   │       ├── ProductPreviewAction.java
│   │       ├── ProductReviewAction.java
│   │       ├── ProductSearchFilterCriteria.java
│   │       ├── Property.java
│   │       ├── RelationShipAction.java
│   │       ├── RelationShipcrosssellItemsAction.java
│   │       └── RelationShipfeaturedItemsAction.java
│   └── [configuration files]
│
└── [other modules]                 # Additional modules (web, etc.)
```

## Critical Folders

| Folder | Purpose | Criticality |
|---|---|---|
| `sm-core/src/` | Core business logic and entity management | **Critical** |
| `sm-core/conf/` | Hibernate, Spring, and application configuration | **Critical** |
| `sm-core/lib/` | External dependencies (Spring, Hibernate, Struts, Axis) | **Critical** |
| `sm-central/src/` | Web interface and management actions | **High** |
| `sm-core/build.xml` | Ant build configuration | **High** |

## Entry Points

| Entry Point | File | Description |
|---|---|---|
| **Core Module** | `sm-core/build.xml` | Main build configuration for core libraries |
| **Central Module** | `sm-central/src/com/salesmanager/central/` | Web management interface |
| **Product Management** | `EditProductAction.java` | Product CRUD operations |
| **Cart Management** | `AddProduct.java` | Shopping cart functionality |
| **Catalog Management** | `ProductListAction.java` | Product catalog interface |

## Package Structure

```
com.salesmanager
├── core/                           # Core business logic
│   ├── entity/                     # Entity classes (JPA/Hibernate)
│   ├── service/                    # Business services
│   └── module/                     # Module definitions
└── central/                        # Web management interfaces
    ├── cart/                       # Shopping cart management
    └── catalog/                    # Product catalog management
```

## Integration Points

- **Hibernate ORM**: Database abstraction layer
- **Spring Framework**: Dependency injection and configuration
- **Apache Struts**: Web MVC framework
- **Apache Axis**: SOAP web services
- **JAX-WS**: Java web services

## Notable Observations

1. **Legacy Java Architecture** — Built for Java 1.5.0 (very outdated)
2. **Ant Build System** — Traditional Ant instead of Maven/Gradle
3. **Multiple Web Service Technologies** — Both Axis and JAX-WS
4. **Eclipse Project Structure** — `.classpath` and `.project` files present
5. **Comprehensive Product Management** — Extensive catalog management actions
6. **Shopping Cart Integration** — Full cart functionality with properties
7. **File Upload Support** — Product image upload capabilities
8. **Internationalization** — Country selection and multi-language support
