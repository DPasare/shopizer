# Shopizer 3.2.7

Shopizer is an open-source headless commerce platform for Java-based ecommerce stores. It provides REST APIs and the core services needed to run a catalog, shopping cart, checkout, order, customer, merchant, and user experience.

[![last_version](https://img.shields.io/badge/last_version-v3.2.7-blue.svg?style=flat)](https://github.com/shopizer-ecommerce/shopizer/tree/3.2.7)
[![Official site](https://img.shields.io/website-up-down-green-red/https/shields.io.svg?label=official%20site)](http://www.shopizer.com/)
[![Docker Pulls](https://img.shields.io/docker/pulls/shopizerecomm/shopizer.svg)](https://hub.docker.com/r/shopizerecomm/shopizer)
[![stackoverflow](https://img.shields.io/badge/shopizer-stackoverflow-orange.svg?style=flat)](http://stackoverflow.com/questions/tagged/shopizer)
[![CircleCI](https://circleci.com/gh/shopizer-ecommerce/shopizer.svg?style=svg)](https://circleci.com/gh/shopizer-ecommerce/shopizer)

## What Shopizer includes

- Catalog management
- Shopping cart and checkout flows
- Merchant and customer management
- Order processing
- User and role management
- REST APIs documented with Swagger

## Quick start

The project is a multi-module Maven build. The main runnable application is `sm-shop`, which starts the Spring Boot API on port `8080`.

### Prerequisites

- Java 11 is supported by the Docker and CI setup used in this repository.
- Maven Wrapper (`./mvnw`) is provided and should be used instead of a system Maven install.

### Build and run locally

```bash
./mvnw clean install -DskipTests
cd sm-shop
../mvnw spring-boot:run
```

Then open the API documentation at:

- `http://localhost:8080/swagger-ui.html`

### Run with Docker

```bash
docker run -p 8080:8080 shopizerecomm/shopizer:latest
```

### Related applications

- Admin app: `shopizerecomm/shopizer-admin`
- React storefront: `shopizerecomm/shopizer-shop-reactjs`

Both require the Java backend to be running.

## Project structure

- `sm-core-model/` — shared entities and value objects
- `sm-core-modules/` — payment, shipping, tax, and search integrations
- `sm-core/` — business logic and service layer
- `sm-shop-model/` — REST request/response models
- `sm-shop/` — Spring Boot app, REST controllers, and security

## Documentation

Additional documentation is available in the repository:

- `docs_functional/` — functional guides and setup notes
- `docs_technical/` — architecture, development, and integration docs
- `RELEASE-NOTES.md` — recent release notes

External resources:

- Shopizer website: <http://www.shopizer.com>
- Shopizer docs: <https://shopizer-ecommerce.github.io/documentation/>
- Community Slack: <https://communityinviter.com/apps/shopizer/shopizer>

## Contributing

1. Fork the repository.
2. Create a feature branch.
3. Make your changes.
4. Run the build and relevant tests.
5. Open a pull request.

Please keep changes focused and follow the existing module boundaries when contributing.
