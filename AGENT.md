# AGENT.md — Shopizer

Guidance for AI coding agents working inside this directory. Pair this with
the top-level `/CLAUDE.md` of the parent SRE PoC repo for project-wide rules.

## What this repo is

Shopizer 3.2.7 — Java open-source headless ecommerce platform (catalog, cart,
checkout, orders, customers). Spring Boot REST API exposed on port `8080`,
Swagger at `/swagger-ui.html`. In this PoC it is the **target application**
the AppTwin + Agent-SRE demo observes and modifies.

## Repo status here

- This directory is a **git submodule** (see `.git` → `gitdir: ../../.git/modules/...`).
  Commits made here belong to the submodule's own history. Bump the parent
  pointer with a separate commit in the parent repo.
- Upstream: `https://github.com/shopizer-ecommerce/shopizer`. Do not push to
  upstream — the PoC uses a fork/local-only branch.
- Vendored, not auto-generated. Direct edits are expected.

## Layout

Multi-module Maven build (parent `pom.xml`):

| Module | Purpose |
|---|---|
| `sm-core-model/` | JPA entities, value objects (shared domain) |
| `sm-core-modules/` | Pluggable subsystems (payment, shipping, tax, search) |
| `sm-core/` | Service layer, business logic, Hibernate config |
| `sm-shop-model/` | DTOs / request-response models for the REST API |
| `sm-shop/` | Spring Boot app, REST controllers, security, entry point |

Runnable artifact: `sm-shop/target/shopizer.jar`.

Docs already in the repo — read these before refactoring:
- `docs_technical/architecture.md`, `development-guide.md`, `integration-architecture.md`
- `docs_functional/` (api, architecture, configuration, getting-started, model, tools)
- `docs_technical/TECH_DEBT_AUDIT.md` for known smells

Note: `docs_technical/development-guide.md` references an older Ant-based
layout. The actual build is **Maven** (`mvnw`, `pom.xml`). Trust the code.

## Build & run

```bash
# Full build (from this directory)
./mvnw clean install -DskipTests

# Run the API (H2 in-memory DB, profile from sm-shop/src/main/resources/profiles/docker)
cd sm-shop && ../mvnw spring-boot:run

# Run tests
./mvnw test                 # all modules
./mvnw -pl sm-shop test     # one module
./mvnw -pl sm-shop -Dtest=ClassName#method test
```

Container build for the demo stack: `docker build -f Dockerfile.dev -t shopizer:dev .`
(Java 11 + H2, used by the PoC's `02_Tools/27_run_sre_demo_stack.sh`).

Java versions: README says 17+ but `Dockerfile.dev` and CI use Java 11. Match
what you find in the module's effective POM before changing toolchain.

## Conventions for agents

- **Edit only what the task requires.** Shopizer is large; avoid drive-by
  reformatting, dependency bumps, or "while I'm here" refactors.
- **Database config**: never commit `sm-shop/src/main/resources/database.properties`
  (it's gitignored). Use the profile copies under `profiles/{docker,h2,mysql,postgres}`.
- **Secrets**: no `.env`, keys, or tokens. Use Spring profiles + env vars.
- **Module boundaries**: domain entities go in `sm-core-model`, request/response
  shapes in `sm-shop-model`, business logic in `sm-core`, HTTP plumbing in `sm-shop`.
  Don't import controllers from `sm-core`.
- **Tests live next to code** under each module's `src/test/java`. Add tests for
  any service-layer change.
- **Maven wrapper**: always use `./mvnw` (not a system `mvn`) so the toolchain
  is reproducible.
- **No `.iml`, `.project`, `target/`, `.idea/` in commits** — already in `.gitignore`.

## Tooling available in this dir

- **Repowise MCP** (`.mcp.json`): codebase intelligence — symbol/graph queries,
  git signals, dead-code, decision history. Prefer it over raw grep for
  cross-module questions ("where is `OrderService.process` called from?",
  "what changed in checkout last quarter?").
- **`.repowise/`**: cached index. Safe to ignore; do not edit by hand.
- **`agent-browser`** (project-wide rule, see parent `CLAUDE.md`): the only
  sanctioned browser-automation tool for UI checks against `http://localhost:8080`.

## Verifying changes

Minimum bar before declaring a task done:

1. `./mvnw -pl <module> -am compile` for the touched module + its dependents.
2. Targeted unit tests for the changed class.
3. If a REST endpoint changed, smoke-test via `agent-browser open http://localhost:8080/swagger-ui.html`
   or `curl` and confirm the response shape.
4. For DB-schema changes, verify Hibernate startup against the H2 profile.

## When in doubt

- Functional question → `docs_functional/` first.
- "How does X work today?" → ask Repowise MCP, then read the code.
- "Should I add a new module / framework / library?" → don't, unless the task
  explicitly calls for it. This is a demo target; surface area should stay
  small and stable.
