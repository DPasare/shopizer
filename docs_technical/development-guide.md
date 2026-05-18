# Development Guide — ShopizerApp

## Prerequisites

| Requirement | Version | Notes |
|---|---|---|
| **Java Development Kit** | 1.5.0+ (recommended: 8+) | Legacy requirement, modern versions preferred |
| **Apache Ant** | 1.7.0+ | Build system (consider migrating to Maven) |
| **Eclipse IDE** | Any version | Project includes Eclipse configuration |
| **Database** | MySQL/PostgreSQL/H2 | Hibernate supports multiple databases |
| **Application Server** | Tomcat/JBoss/WebLogic | Java EE compatible server |

## Getting Started

### 1. Environment Setup

```bash
# Install Java (preferably Java 8 or higher)
java -version

# Install Apache Ant
ant -version

# Clone or extract the project
cd shopizer-v1.1.5
```

### 2. Project Structure

```bash
shopizer-v1.1.5/
├── sm-core/          # Core business logic
├── sm-central/       # Web management interface
└── [other modules]   # Additional modules
```

### 3. Build Configuration

The project uses **Apache Ant** for building. Key build targets:

```bash
# Navigate to core module
cd sm-core

# Initialize build environment
ant init

# Compile source code
ant compile

# Build JAR file
ant jar

# Full build
ant build

# Generate Javadoc
ant javadoc
```

### 4. Configuration Files

**Core Module Configuration** (`sm-core/conf/`):
- `hibernate/` — ORM configuration
- `properties/` — Application properties
- `spring/` — Spring framework configuration
- `META-INF/` — Manifest and metadata

**Build Configuration** (`sm-core/build.xml`):
- Library dependencies in `lib/` subdirectories
- Classpath configuration for different frameworks
- Javadoc generation settings

---

## Project Scripts (build.xml)

| Target | Description | Dependencies |
|---|---|---|
| `init` | Clean and create build directories | — |
| `compile` | Compile Java source files | `init` |
| `copyconf` | Copy configuration files | — |
| `jar` | Create JAR file | `compile`, `copyconf` |
| `build` | Complete build process | `jar` |
| `javadoc` | Generate API documentation | `build` |

---

## Configuration Files

### build.xml

Key properties and configurations:

```xml
<!-- Library directories -->
<property name="lib.dir" value="lib" />
<property name="dist.dir" value="dist" />
<property name="src.dir" value="src" />
<property name="conf.dir" value="conf" />

<!-- Classpath configuration -->
<path id="master-classpath">
    <fileset dir="${lib.dir}">
        <include name="*.jar" />
    </fileset>
    <!-- Additional library directories -->
    <fileset dir="${lib.dir}/hibernate/">
        <include name="*.jar" />
    </fileset>
    <fileset dir="${lib.dir}/spring/">
        <include name="*.jar" />
    </fileset>
    <fileset dir="${lib.dir}/struts/">
        <include name="*.jar" />
    </fileset>
</path>
```

### Hibernate Configuration

Located in `sm-core/conf/hibernate/`:
- Database connection settings
- ORM mappings
- Caching configuration
- Transaction management

### Spring Configuration

Located in `sm-core/conf/spring/`:
- Bean definitions
- Dependency injection
- AOP configuration
- Transaction management

---

## Common Development Tasks

### Adding a New Product Management Action

1. **Create Action Class** in `sm-central/src/com/salesmanager/central/catalog/`:
   ```java
   public class NewProductAction extends BaseAction {
       // Implement action logic
   }
   ```

2. **Configure in Struts** (struts-config.xml):
   ```xml
   <action path="/newProduct"
           type="com.salesmanager.central.catalog.NewProductAction"
           name="productForm"
           scope="request">
       <forward name="success" path="/productList.jsp"/>
   </action>
   ```

3. **Create JSP View** for the interface

### Adding a New Entity

1. **Create Entity Class** in `sm-core/src/com/salesmanager/core/entity/`:
   ```java
   @Entity
   public class NewEntity {
       // Entity fields and mappings
   }
   ```

2. **Create Service Interface** in `sm-core/src/com/salesmanager/core/service/`:
   ```java
   public interface NewEntityService {
       // Service methods
   }
   ```

3. **Configure Hibernate Mapping**

### Database Schema Changes

1. **Update Entity Classes** with new fields
2. **Update Hibernate Mappings**
3. **Create Database Migration Script**
4. **Test with Different Databases**

---

## Testing

**No test framework is currently configured.** The project has no visible unit tests, integration tests, or test directories.

### Recommended Testing Setup

For future implementation:
- **Unit Tests**: JUnit 4 or 5
- **Integration Tests**: TestNG or JUnit
- **Web Tests**: Selenium or HtmlUnit
- **Database Tests**: DBUnit
- **Mock Framework**: Mockito or EasyMock

---

## Deployment

### Production Build

```bash
# Full build with documentation
ant build

# Generate API documentation
ant javadoc
```

### Deployment Steps

1. **Build Application**:
   ```bash
   cd sm-core
   ant build
   ```

2. **Deploy to Application Server**:
   - Copy generated JAR files to server
   - Deploy web applications (WAR files)
   - Configure database connections

3. **Configure Database**:
   - Set up database schema
   - Configure Hibernate connection
   - Import initial data

### Environment Requirements for Production

- Java Application Server (Tomcat, JBoss, WebLogic)
- Database server (MySQL, PostgreSQL, etc.)
- Java 1.5.0+ runtime (preferably Java 8+)
- Sufficient memory for Hibernate caching

---

## Legacy Issues and Modernization

### Critical Issues

1. **Java 1.5.0 Compatibility** — Very outdated Java version
2. **Legacy Framework Versions** — Old Struts, Spring, Hibernate
3. **Apache Axis** — Deprecated SOAP framework
4. **Ant Build System** — No modern dependency management
5. **Windows-specific Paths** — Hard-coded Windows paths

### Modernization Recommendations

1. **Upgrade Java** to Java 8 or higher
2. **Migrate to Maven** for dependency management
3. **Upgrade Frameworks** to modern versions
4. **Replace Axis** with REST APIs
5. **Add Unit Tests** with JUnit
6. **Containerize** with Docker
7. **Add CI/CD** pipeline

---

## Troubleshooting

### Common Issues

1. **Build Failures**:
   - Check Java version compatibility
   - Verify Ant installation
   - Ensure all dependencies in `lib/` directories

2. **Database Connection Issues**:
   - Verify Hibernate configuration
   - Check database server availability
   - Validate connection properties

3. **Web Service Issues**:
   - Check Axis/JAX-WS configuration
   - Verify service endpoints
   - Review SOAP message formats

4. **ClassNotFoundException**:
   - Verify classpath in build.xml
   - Check JAR files in lib directories
   - Ensure proper dependency ordering
