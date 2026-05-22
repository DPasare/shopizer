# Tech Debt Audit — ShopizerApp (shopizer-v1.1.5)

Generated: 2026-05-10
Repo: `/home/alina/CascadeProjects/ShopizerApp/shopizer-v1.1.5` (no git history — frozen snapshot)
Scale: 677 Java files / ~125k LOC, 268 JSPs, 132 XML configs, 4 Ant modules (`sm-core`, `sm-central`, `sm-shop`, `media`).

> Static audit. Dependencies were inspected via `.classpath`/`lib/*/readme.txt` only — actual `.jar` files are not in the tree, so version pins below come from those manifests. CVE counts are *not* run through `npm audit`/`pip-audit` equivalents because no Maven/Gradle manifest exists.

## Executive summary

Top-10 by impact, ranked. All ten are real and citable.

1. **Authentication is encryption-not-hashing.** Customer passwords are AES-CBC encrypted with a hardcoded key derived from the literal string `"100"` and a hardcoded IV `"fedcba9876543210"`, then compared by `Restrictions.eq("customerPassword", encrypted)`. Single DB read recovers every password in plaintext. ([F001], [F002], [F003])
2. **Struts 2.2.1.1 + xwork-core 2.2.1.1** ship in the build classpath. This branch covers OGNL RCEs S2-005, S2-008, S2-009, S2-013, S2-016, S2-019, S2-020, plus the ParametersInterceptor / DefaultActionMapper class. Internet-facing deploy = guaranteed RCE. ([F004])
3. **Arbitrary file delete via `BinUploadAction.deleteFile`.** User-supplied `deleteFilePath` is fed straight into `new File(deleteFilePath)` and deleted. No path canonicalization, no whitelist. ([F005])
4. **Arbitrary Spring-bean dispatch from `FilesAction.getFile`.** `mod` request parameter is passed to `SpringUtil.getBean(mod)`. Any bean implementing `DownloadFileModule` is instantiable by URL. The null-check on `mod` runs *after* the `getBean` call. ([F006])
5. **CVV and `transactionKey` written to debug log** in payment gateway impls (AuthorizeNet, Beanstream, Psigate, Paypal — all DEBUG by default in `log4j.properties`). PCI-DSS 3.2.x section 3.2 violation. ([F007])
6. **Auth filter only on `*.action`.** `/dwr/*`, `/salesManagerCustomerService`, `/salesManagerInvoiceService`, JSPs, statics are reachable unauthenticated. The HTTP transport-guarantee blocks are commented out — checkout flow can run over HTTP. ([F008], [F009])
7. **Hardcoded Facebook HMAC secret** `93b47625ec3dcc4172fda796899ae42d` in `FacebookIntegrationFactory.java:151`. ([F010])
8. **God classes everywhere.** `OrderService` 2333 LOC, `CatalogService` 2283, `Order` entity 1679, `EditProductAction` 1420, `ShippingCustomRatesAction` 1223, payment gateway impls 800–1200 LOC each. ([F011]–[F015])
9. **Zero tests.** No JUnit, no `@Test`, no `src/test`. 124k LOC business logic unverified. ([F016])
10. **Build system is Ant 1.5-era with hardcoded Windows paths**, no dependency lockfile, jars committed (well, *referenced* — they aren't even in tree, the build is unbuildable as shipped). ([F017], [F018])

Counts: **5 Critical, 18 High, 23 Medium, 14 Low** = 60 findings total.

## Architectural mental model

Three-WAR multi-tenant e-commerce monolith with a fourth shared core jar:

- **`sm-core`** — Spring 2.5 + Hibernate 3.2 jar. Holds entities, DAOs, services, payment/shipping/file modules, util "managers". Wired both via `@Service`/`@Autowired` (342 annotation hits) **and** via a `ServiceFactory.getService(String)` service-locator over a static `SpringUtil` (105 hits). Both mechanisms point at the same Spring beans — the codebase never picked one. New code uses `@Autowired`; old code uses the locator; both coexist permanently.
- **`sm-central`** — Struts 2.2.1.1 admin WAR. Custom `CustomAuthFilter` over `AuthFilter` checks a session attribute; `bypassUrl` uses `String.contains("/rest/")` and `String.contains("/anonymous/")` for substring-based bypass. Per-action role enforcement via `smStoreRoleStack` etc. interceptor stacks.
- **`sm-shop`** — Struts 2.2.1.1 storefront WAR. **No `AuthFilter`** at all (intentional — public storefront), but checkout/files are mixed into the same WAR with no transport-confidentiality guarantee.
- **`media`** — empty WAR shell. Nothing in it. Looks like a stub never finished.

Multi-tenancy lives in a single `merchantId` column on every entity, plus a `STORE` cookie / session attribute decided by `SalesManagerInterceptor`. The interceptor accepts `merchantId` as a request parameter and writes it to a session/cookie — there is no check that the user is *allowed* to see that merchant.

The central WAR also packages two SOAP endpoints (`salesManagerCustomerService`, `salesManagerInvoiceService`) via `WSServletContextListener`; these are mapped under non-`*.action` patterns so the auth filter does not cover them.

There is a Hibernate Search index on `Product` updated by post-update/insert/delete listeners, plus an OSCache layer.

This matches what `docs_bmad/architecture.md` claims at the surface, but the docs miss the ServiceFactory/Spring duplication, the bypass-substring trick, the absent `media` content, and the unprotected SOAP endpoints. **The docs are aspirational; the code is what shipped.**

## Findings

| ID | Category | File:Line | Severity | Effort | Description | Recommendation |
|----|----------|-----------|----------|--------|-------------|----------------|
| F001 | Security | `sm-core/src/com/salesmanager/core/util/EncryptionUtil.java:47,65,81` | **Critical** | M | Hardcoded IV `"fedcba9876543210"` reused for every encrypt under AES-CBC. Defeats CBC semantic security; same plaintext → same ciphertext, enables enumeration. | Generate a random 16-byte IV per message, prepend to ciphertext, parse on decrypt. Better: drop CBC, use `AES/GCM/NoPadding`. |
| F002 | Security | `sm-core/src/com/salesmanager/core/constants/SecurityConstants.java:21` + `EncryptionUtil.java:34` | **Critical** | M | Customer password "encryption" key is `EncryptionUtil.generatekey("100")` = `"1000000000000000"`. Single global static key. Anyone with source can decrypt every password in the DB. | Replace with bcrypt/scrypt/Argon2 password **hashing** (see F003); for at-rest secrets that must be reversible, load key from env / KMS. |
| F003 | Security | `sm-core/src/com/salesmanager/core/module/impl/application/logon/JAASSecurityCustomerLoginModule.java:122-127` and `sm-core/src/com/salesmanager/core/service/customer/impl/dao/CustomerDao.java:176` | **Critical** | L | Login compares a *reversible* AES-encrypted password to the stored value via `Restrictions.eq("customerPassword", encPassword)`. Database compromise = full credential plaintext. No salt, no work factor, no rate-limit. | Migrate to per-user-salted bcrypt; add a one-time rehash-on-next-login path; phase out the encrypted column. |
| F004 | Dependency / Security | `sm-core/.classpath:81-90` (`struts2-core-2.2.1.1.jar`, `xwork-core-2.2.1.1.jar`, `ognl-3.0.jar`, `commons-fileupload-1.2.1.jar`, `commons-collections-3.2.jar`) | **Critical** | L | Struts 2.2.1.1 is vulnerable to S2-005/008/009/013/016/019/020 OGNL RCE chain. commons-collections 3.2 = CVE-2015-7501 deserialization RCE. commons-fileupload 1.2.1 = CVE-2014-0050 / 2016-3092. ognl 3.0 reachable through Struts. | Either move to Struts 2.5.33+ (requires DTD bumps and OGNL allowlist work), or front the app with a WAF and treat the box as compromised by default. The Struts upgrade is non-trivial; see F019. |
| F005 | Security | `sm-central/src/com/salesmanager/central/merchantstore/BinUploadAction.java:103-113` | **Critical** | S | `deleteFile()` takes user-supplied `deleteFilePath`, wraps it in `new File(...)` and deletes. No canonicalization, no merchant-scope check. Authenticated admin can delete *any* file the JVM user can. | Resolve canonical path; verify it lives under `merchantid`'s allowed bin dir; reject `..` traversal. |
| F006 | Security | `sm-shop/src/com/salesmanager/checkout/files/FilesAction.java:211-224` | **Critical** | S | `mod` request parameter feeds `SpringUtil.getBean(mod)`. Anyone can instantiate any Spring bean in the context that happens to implement `DownloadFileModule` — including ones meant to be internal. The null check on `mod` runs *after* `getBean` (line 220 vs 218) so a missing param NPEs the lookup. | Allowlist the legal `mod` values to a small enum; check null first; verify the returned bean type. |
| F007 | Security / Compliance | `sm-core/src/com/salesmanager/core/module/impl/integration/payment/AuthorizeNetTransactionImpl.java:215,222`; `BeanStreamTransactionImpl.java`, `PsigateTransactionImpl.java`, `PaypalTransactionImpl.java` (analogous) and `sm-core/conf/properties/log4j.properties:92` | **Critical** | S | CVV (`x_card_code`) and `transactionKey` are appended to a `StringBuffer` and logged via `log.debug("Transaction sent -> " + sblog.toString())`. The payment logger is `DEBUG` by default, written to `sm-payment.log`. PCI-DSS forbids storing CVV at all. | Strip CVV before any log/serialize. Mask `transactionKey`. Default the payment logger to `INFO` and gate `DEBUG` on a per-deploy property. |
| F008 | Security | `sm-central/WebContent/WEB-INF/web.xml:16-19,54-68,84-87` | High | S | Auth filter is mapped only to `*.action`. SOAP endpoints `/salesManagerCustomerService`, `/salesManagerInvoiceService` and `/dwr/*` are reachable without authentication. | Add filter mappings for these patterns or wrap WS servlets in a JAX-WS handler that checks the session principal. |
| F009 | Security | `sm-central/WebContent/WEB-INF/web.xml:109-134` and `sm-shop/WebContent/WEB-INF/web.xml:71-86` | High | S | `<security-constraint>` with `<transport-guarantee>CONFIDENTIAL</transport-guarantee>` is commented out in both webapps. Checkout and admin can run over HTTP. | Uncomment, plus enforce HSTS at the reverse proxy. Login pages especially. |
| F010 | Security | `sm-core/src/com/salesmanager/core/util/www/integration/fb/FacebookIntegrationFactory.java:151` | High | S | `SecretKeySpec("93b47625ec3dcc4172fda796899ae42d".getBytes(), "HMACSHA256")` — Facebook app secret hardcoded. | Read from `MerchantConfiguration` (it already exists per-merchant for other modules) or env. |
| F011 | Architectural decay | `sm-core/src/com/salesmanager/core/service/order/OrderService.java:1` | High | L | 2333-LOC service. Mixes order CRUD, jasper invoice rendering, status transitions, file history, totals computation, e-mail send. Imports 100+ symbols. | Extract InvoiceRenderer, OrderTotalCalculator, OrderStatusService. Stop importing `JasperFillManager` from a service class. |
| F012 | Architectural decay | `sm-core/src/com/salesmanager/core/service/catalog/CatalogService.java:1` | High | L | 2283-LOC service spanning category tree, products, attributes, options, reviews, prices, specials. | Split along the 6 sub-domains it touches (one service per `entity.catalog.*` aggregate). |
| F013 | Architectural decay | `sm-core/src/com/salesmanager/core/entity/orders/Order.java:1` | High | L | 1679-LOC entity, 60+ private fields. JPA entity carries display-only computed fields and rendering helpers. | Move computed/derived fields to a separate `OrderView` projection; keep persistent fields on the entity. |
| F014 | Architectural decay | `sm-central/src/com/salesmanager/central/catalog/EditProductAction.java:1` | High | L | 1420-LOC Struts action. CRUD for product, descriptions, prices, attributes, images, categories — all in one action with one form bean. | Split per tab (one action per CRUD pane), share the entity loader. |
| F015 | Architectural decay | `sm-core/src/com/salesmanager/core/util/ProductUtil.java:1`, `CheckoutUtil.java:1` | High | M | 984-LOC and 826-LOC "util" classes that are really services in disguise (they call `ServiceFactory.getService` to fetch DAOs). | Promote to actual `@Service` beans, inject deps. The `Util` suffix is misleading. |
| F016 | Test debt | (whole repo) | High | L–XL | Zero JUnit/TestCase/`@Test` files. No `src/test*` directory. 124k LOC, every refactor is a leap of faith. | Start with characterization tests on `OrderService` totals, `CreditCardUtil`, `EncryptionUtil`, `ProductDao.search*`. Aim for 30% coverage on the four god services first. |
| F017 | Build / Dep | `sm-core/build.xml:100`, `sm-core/.classpath:11,82` | High | M | `<link offline="true" href="http://java.sun.com/j2se/1.5.0/docs/api/" packagelistLoc="C:\tmp"/>` — Windows path baked in. `.classpath` `sourcepath="C:/Documents and Settings/Carl Samson/.m2/repository/..."`. | Drop those `sourcepath` attrs; remove the offline javadoc link. |
| F018 | Build / Dep | `sm-core/lib/*/readme.txt`, all `build.xml` | High | XL | Build references jars under `lib/<group>/*.jar` but **the jar files are not in the tree** — only `readme.txt` listing the version names. The project as shipped will not compile without manually fetching ~80 named jars at unknowable mirror locations. | Migrate to Maven (or Gradle). Pin versions in a POM; let CI download. The current state is "build instructions in plain English." |
| F019 | Dependency | `sm-core/.classpath:30-69` | High | L | `spring.jar` (no version, likely 2.5.x), `hibernate3.jar` (3.2.7 per readme), `log4j-1.2.16` (EOL 2015), `mysql-connector-java-5.0.6` (2007), `jasperreports-3.7.4`, `iText-2.1.0`, `jackson-1.6.2`, `groovy-1.5.5`, `axis-1.4`, `freemarker-2.3.16`. Every one of these has known CVEs and is past EOL. | Once on Maven (F018), bulk-bump. iText 2.x → OpenPDF 1.3.x is a near-drop-in. Jackson 1.x → 2.17 needs API rewrites. |
| F020 | Security | `sm-core/src/com/salesmanager/core/security/CustomAuthFilter.java:42-51` | High | S | `bypassUrl` uses `url.contains("/rest/")` / `url.contains("/anonymous/")`. Any path that contains those substrings anywhere bypasses auth, including encoded forms. | Match on prefix after canonicalizing the URI, not `contains`. |
| F021 | Security | `sm-core/src/com/salesmanager/core/security/AuthFilter.java:69-101` | Medium | S | The branch `if (session == null) {...}` is dead — `req.getSession()` (line 62, no `false`) creates a session if none exists. Real auth check happens on lines 105-136. Misleading code, though not exploitable. | Delete the dead block, or pass `false` to `getSession()` and handle truly-missing session correctly. |
| F022 | Security | `sm-core/src/com/salesmanager/core/util/www/SalesManagerInterceptor.java:242-244` | Medium | S | `STORE` cookie set without `setSecure(true)` / `setHttpOnly(true)`, and survives 365 days. Determines which merchant tenant the user sees. | Add Secure (over HTTPS) and HttpOnly. Reduce maxAge or move to session-only. |
| F023 | SQL hygiene | `sm-core/src/com/salesmanager/core/service/order/impl/dao/OrderDao.java:128-130` | Medium | S | `createSQLQuery("INSERT INTO orders(orders_id) values (" + orderId + ")")` — string-concat SQL. `orderId` is a `long` (not user-string), so it's not exploitable, but the pattern is bad and someone will copy it. | Use `setLong(1, orderId)` via parameter binding. Or just `session.save(new Order(orderId))`. |
| F024 | SQL hygiene / bug | `sm-core/src/com/salesmanager/core/service/order/impl/dao/OrderDao.java:226` | High | S | `q.append(" and o.customerName like %:cName%")` — invalid HQL. `%` literal cannot surround a named parameter binding. Method `searchInvoice` will throw at runtime when `customerName` is set. | Build the parameter as `"%" + name + "%"` and bind without literal `%`. |
| F025 | SQL hygiene / bug | `sm-core/src/com/salesmanager/core/service/order/impl/dao/OrderDao.java:233-245` and `:338-350` | Medium | S | Both branches of the `searchCriteria.getSdate() != null` if/else are *identical* (`q.append(" and o.datePurchased > :sDate")`). Same for `getEdate`. | Collapse to single `if`. |
| F026 | Bug | `sm-core/src/com/salesmanager/core/service/order/impl/dao/OrderDao.java:388-392` | High | S | When end-date is null, the code defaults to `Restrictions.ge("datePurchased", today+1day)` — i.e. filters out *every* order. Should be `le(today+1day)`. | Flip to `le`. |
| F027 | Bug / Architecture | `sm-core/src/com/salesmanager/core/service/order/impl/dao/OrderDao.java:203-411` | Medium | M | `searchInvoice` / `searchOrder` build a Hibernate `Criteria` AND an HQL `Query` in parallel, then only one is used to fetch — the other accumulates restrictions and is discarded. Plus `criteria.setProjection(rowCount)` then `setProjection(null)` reuses the same Criteria for count + result, accumulating duplicate restrictions. | Pick one (HQL preferred for readability). Build separate count and list queries. |
| F028 | Bug / Concurrency | `sm-core/conf/spring/sm-core-config.xml:58-60` | High | S | C3P0 props are shuffled: `hibernate.c3p0.max_size=${database.pool.timeout}` and `hibernate.c3p0.timeout=${database.pool.max_size}`. So pool max=100 connections, idle timeout=25s — opposite of intent. | Swap. |
| F029 | Performance / Logging | `sm-core/conf/spring/sm-core-config.xml:51` | Medium | S | `hibernate.show_sql=true` hardcoded. Spams stdout in prod. | Externalize to a property; default false. |
| F030 | Concurrency | `sm-core/src/com/salesmanager/core/util/SpringUtil.java:33-36` | Medium | S | `if (instance == null) instance = new SpringUtil()` without synchronization. Two simultaneous calls before init can each create their own `SpringUtil` and resolve different `BeanFactory` instances. | Make `instance` `static final` initialized at class-load, or use double-checked-locking with `volatile`. Or just delete the singleton and use `ApplicationContextAware`. |
| F031 | Architectural decay / Consistency | (whole `sm-core/src/com/salesmanager/core/service/`) | High | L | Two parallel DI mechanisms: 342 `@Service`/`@Autowired`/`@Repository` annotations vs 105 `ServiceFactory.getService(...)` lookups + `SpringUtil.getBean(...)` calls. Both target the same beans. | Pick `@Autowired` and migrate the locator call sites in waves. The `ServiceFactory` constants list is small (10 entries) so it's tractable. |
| F032 | Consistency | (whole repo) | Medium | M | Two logging frameworks coexist: 200 `org.apache.log4j.Logger` imports vs 63 `org.apache.commons.logging.Log` + `LogFactory` imports. No SLF4J. | Standardize on SLF4J + log4j-over-slf4j bridge. |
| F033 | Consistency | `sm-core/src/com/salesmanager/core/service/catalog/impl/db/dao/CategoryDao.java:104-128` | Medium | S | `save(Category)` uses raw `PreparedStatement` via `session.connection()` — bypasses Hibernate L1/L2 cache, dirty-checking, version optimistic-lock. Every other DAO method on the same class uses Hibernate's `saveOrUpdate`. | Replace with `getHibernateTemplate().save(instance)`. |
| F034 | Concurrency | `sm-core/src/com/salesmanager/core/service/cache/RefCache.java:86`, `ServicesUtil.java:47-49`, `ModuleManagerImpl.java:42`, `LabelUtil.java:162`, `CurrencyUtil.java:40` | Medium | M | Static mutable `HashMap`/`ArrayList` shared across threads, no synchronization. (`CacheUtil.java:33` did the right thing with `synchronizedMap` — proves the team knew, just forgot the others.) | Switch to `ConcurrentHashMap` / `CopyOnWriteArrayList`, or front with synchronized accessor methods. |
| F035 | Type debt | (`grep -E "List\\s+\\w+\\s*=" sm-core/src/com/salesmanager/core/service`) | Medium | L | Heavy use of raw collections (`List`, `Map`, `Collection` without generics) — sample sites: `TransactionImpl.java:71`, `MerchantUserInformationDao.java:135`, `CustomerDao.java:143,188,201,223,292`, `CommonService.java:39,69,78`. ~1100 `@SuppressWarnings("unchecked"|"rawtypes")` repo-wide. | Add generics module-by-module; the IDE refactor handles 80% in one pass. |
| F036 | Error handling | (~1102 catch sites in `sm-*` src) | Medium | L | Dominant pattern is `catch (Exception e) { log.error(e); ... return "GENERICERROR"; }`. Loses exception type, swallows root cause from caller, sometimes wraps in `RuntimeException`. 86 `printStackTrace` calls leak to stderr. | Replace `printStackTrace` with `log.error(msg, e)`. Define a small set of domain exceptions; let infrastructure ones bubble. |
| F037 | Error handling | `sm-core/src/com/salesmanager/core/module/impl/application/logon/JAASSecurityCustomerLoginModule.java:129-132` | Medium | S | Login failure path `catch (Exception e) { e.printStackTrace(); } return false;` — failed login looks identical to a runtime exception. | Distinguish auth-failure from infrastructure-failure; log infra-failures at ERROR with stack. |
| F038 | TODO/FIXME debt | (`grep -n TODO --include="*.java"`) | Low | M | 160 TODO/FIXME markers, most `// TODO Auto-generated method stub` from Eclipse, indicating module impls left empty (e.g. `GenericCurrencyModule.java:139,161,182,196`, `CalculateItemPackingModule.java:42,105,111,118,124`, `SimpleCaptchaModule.java:70`, `JAASLogonImpl.java:35`). | Triage: delete unreachable stubs; finish the rest; lift a `// TODO Auto-generated method stub` ban into the build. |
| F039 | Hibernate / Mapping | `sm-core/conf/hibernate/*.hbm.xml` (82 files) | Medium | L | XML mappings for every entity duplicate Hibernate-Annotations metadata that the entities themselves carry (`@Entity`, `@Table`). Two sources of truth. | Pick one. JPA annotations on entities is the modern default. |
| F040 | Performance | `sm-core/src/com/salesmanager/core/service/order/impl/dao/OrderDao.java:170-176` (`findOrdersByMerchant`) | Medium | S | Returns *all* orders for a merchant, no pagination, no limit. On a healthy store this is millions of rows. | Add `setMaxResults` / pagination on the API. Same audit applies to `findInvoicesByCustomer` (line 180). |
| F041 | Performance / Lazy-load | `sm-core/src/com/salesmanager/core/entity/orders/Order.hbm.xml`, `OrderImpl.java` | Medium | M | Many associations are eager-fetched by default (Hibernate 3 default). Loading one `Order` pulls products, attributes, statuses, totals, downloads in one shot. N+1 expected on list views. | Audit fetch strategies; switch to `lazy="true"` on collections, use `JOIN FETCH` only where needed. |
| F042 | Security | `sm-central/WebContent/WEB-INF/web.xml:70-82` | Medium | S | DWR servlet exposed on `/dwr/*`. Even with `debug=false`, the DWR endpoints expose any registered remoted bean — DWR's own security model is opt-in per-method. | If DWR is in use, audit `dwr.xml` for what's remoted; if not in use, drop the servlet entirely. |
| F043 | Security | `sm-core/src/com/salesmanager/core/util/www/SalesManagerInterceptor.java:95-153` | Medium | M | Multi-tenant store selection driven by user-controlled `merchantId` parameter / cookie with no authorization check that the user belongs to that merchant. Public storefront context — but the same interceptor wires the central admin path too. | Verify the authenticated principal's merchant membership before switching `STORE`. |
| F044 | Build / OS | `sm-central/build.xml:148-151`, `sm-shop/build.xml:139-142` | Medium | S | `deploy` target requires `-Dcontainer.dir`/`-Dwebfolder.name` and copies the WAR into a Tomcat/JBoss tree directly. No environment separation, no rollback. | Ship a WAR; let the CD pipeline place it. |
| F045 | Documentation | `docs_bmad/architecture.md:99-145` vs reality | Medium | S | Doc claims `sm-core` and `sm-central` are the "two primary modules" — `sm-shop` and `media` aren't mentioned. `media` WAR is empty. SOAP unprotected-endpoints not flagged. | Re-derive doc from code (or remove the doc). The pre-existing `docs_bmad/architecture.md` Technical Debt section is also incomplete (misses every Critical above). |
| F046 | Logging hygiene | `sm-core/conf/properties/log4j.properties:55,59,79-92,97` | Medium | S | INTEGRATION appender writes to `sm-catalog.log` (typo, copy-paste from CATALOG). Pattern key `CINTEGRATIONlayout` is misspelled (line 59) — invalid log4j property silently ignored. AXISFILE referenced (line 97) but never declared. All `com.salesmanager.*` loggers default to `DEBUG` — combined with F007, debug-mode CVV exposure. | Fix typos; raise default to `INFO`; keep `DEBUG` overrides per-package, off by default. |
| F047 | Build hygiene | `sm-core/build.xml:62-69` | Low | S | `init` target deletes `${dist.dir}` and `${working.dir}` then mkdirs them — but the `<delete>` runs over a `<fileset>` of an *existing* dir. If those dirs don't exist on first build, the script behavior is "works by accident." | Use `<delete dir="${dist.dir}"/>` instead of `<delete><fileset .../></delete>`. |
| F048 | Build hygiene | `sm-central/build.xml:97-110`, `sm-shop/build.xml:90-103` | Low | M | Each WAR build copies *all* of sm-core's libs (axis, jax-ws, struts, hibernate, spring, misc) into its WEB-INF/lib. WARs end up with overlapping ~150 jars each. | Externalize the shared jars into the container, or use a Maven multi-module setup (F018). |
| F049 | Misnamed file | `sm-core/conf/properties/struts.properties` | Low | S | `sm-core` should not own struts config — Struts is the web tier. | Move to sm-central / sm-shop. |
| F050 | API / SOAP | `sm-central/WebContent/WEB-INF/web.xml:48-68` | Medium | S | `salesManagerInvoiceService` mapped, `salesManagerOrderService` commented out (line 58-63) — so the order WS is half-removed. `WSServletContextListener` still loads its WSDLs. | Decide: ship it or remove it. |
| F051 | Dead code | `sm-core/src/com/salesmanager/core/util/EncryptionUtil.java:62,77` | Low | S | Several "// NEED TO UNDERSTAND WHY PKCS5Padding DOES NOT WORK" comments reflect an unresolved investigation that was patched around. Decryption on `decryptFromExternal` still uses NoPadding while `decrypt` uses PKCS5Padding — splits the path. | Pick one padding; document; delete the other. |
| F052 | Trust boundary | `sm-shop/src/com/salesmanager/checkout/files/FilesAction.java:67-119` | Medium | S | `accessUrl` decides authz purely on `dt.before(today)` token expiry. Token format is built from `EncryptionUtil.encrypt(...)` (broken per F001/F002), so anyone who knows the static key can forge a fileId. | Sign tokens with HMAC-SHA-256 over a server-side secret + per-user nonce; check signature before any file access. |
| F053 | Architecture / SoC | `sm-core/src/com/salesmanager/core/service/order/OrderService.java` (imports `JasperFillManager`, `JasperPrint`, `JasperExportManager`) | Medium | M | Service layer directly drives Jasper PDF rendering. Reporting concerns leak into transactional code. | Extract `InvoiceRenderer` interface; OrderService passes data, renderer formats. |
| F054 | XSS exposure | `sm-shop/WebContent/checkout/components/confirmation.jsp:169`, `invoiceConfirmation.jsp:95`, `thankyou.jsp:127` (`<s:property value="#session.PAYMENTMETHOD.paymentMethodConfig['key']" escape="false"/>`); `dashboard/dashboard.jsp:43`, `editparentcategory.jsp:33` | Medium | S | 25 sites with `escape="false"`. Most are admin-controlled rich text (probably intentional), but `paymentMethodConfig['key']` and `productPrice` are not user-prose; rendering raw is risky if upstream stores user input. | Audit each of the 25; escape unless the field is an explicit rich-text column. |
| F055 | Trust boundary | `sm-central/WebContent/orders/orderdetails.jsp:96,124,145,179` | Low | S | Mixes Struts `<s:property>` (which auto-escapes) with raw JSP `<%= request.getAttribute("SCHEMEID") %>` etc. (e.g. `taxrates.jsp:288-289`). Inconsistency is the risk: someone adds a hidden field copying user input via JSP scriptlet later. | Pick one templating path. Forbid `<%= ... %>` in admin JSPs. |
| F056 | Error handling | `sm-core/src/com/salesmanager/core/util/www/integration/fb/FacebookIntegrationFactory.java:171-173`, `sm-core/src/com/salesmanager/core/service/reference/impl/dao/CoreModuleServiceDao.java:183` | Low | S | `catch (Exception e) { log.error(e); }` swallows entirely; subsequent code keeps running with `user` half-built. CoreModuleServiceDao has a literal `// TODO: handle exception`. | Fail fast; let the caller decide. |
| F057 | Build / Repro | `sm-core/lib/compile/README.TXT` | Low | S | "For compiling only, do not package with application" — guidance lives in a TXT file the build system never reads. The build instead runs `<exclude name="jsp-api.jar"/>` and `<exclude name="servlet-api.jar"/>` in the `copyconf` target — convention is encoded in the wrong place. | Move to a Maven scope (`provided`). |
| F058 | Naming drift | `sm-core/src/com/salesmanager/core/util/` (15+ "Util" classes 980-LOC down) | Low | M | "Util" suffix used for everything: stateless helpers (`HexUtil`), services with state (`RefCache`, `LabelUtil`), and full domain logic (`ProductUtil`, `CheckoutUtil`). | Rename the service-shaped ones to `*Service`; make the stateless ones final + private constructor. |
| F059 | Performance | `sm-core/src/com/salesmanager/core/util/EncryptionUtil.java:90-107` | Low | S | `bytesToHex` builds the hex string with `+` concatenation in a loop — quadratic. Not on a hot path, but still. | `String.format("%02x", b)` or `StringBuilder`. |
| F060 | Documentation drift | `README.md:6-7`, `docs_bmad/project-overview.md:82-89` | Low | S | README says "unofficial maintenance branch... security fixes and enhancement only" — but no security fixes are visible: every Critical above is unaddressed. | Either land the fixes or update the README to "archive — do not deploy". |

## Top 5 — if you fix nothing else, fix these

### 1. Replace password "encryption" with hashing — F001 / F002 / F003

You cannot ship this in any deployed environment. The fix is:

```java
// EncryptionUtil → no longer used for passwords
// Add PasswordHasher (BCrypt example, spring-security-crypto)

public class PasswordHasher {
  private static final BCryptPasswordEncoder enc = new BCryptPasswordEncoder(12);
  public static String hash(String plain)      { return enc.encode(plain); }
  public static boolean verify(String plain, String hash) { return enc.matches(plain, hash); }
}
```

Then change the customer login lookup (`CustomerDao.findCustomerbyUserNameAndPassword` plus `JAASSecurityCustomerLoginModule.isValidUser`) from "look up by `(name, encryptedPwd)`" to "look up by `name`, then `verify(submitted, stored)`". Add a `password_format` column or version-prefix scheme so the DB can carry both old and new hashes during cutover; on next successful old-style login, rehash and persist.

This is a few hundred lines, one DB migration, and one round of customer "password reset on next login" comms.

### 2. Lock down web.xml + auth filter — F008 / F009 / F020 / F042

Three concrete diffs:

```xml
<!-- sm-central/WebContent/WEB-INF/web.xml -->
<filter-mapping>
  <filter-name>auth</filter-name>
  <url-pattern>/*</url-pattern>     <!-- was *.action -->
</filter-mapping>
<!-- and uncomment the security-constraint with CONFIDENTIAL -->
```

```java
// CustomAuthFilter.bypassUrl — drop contains, use prefix on canonical path
String path = new java.net.URI(request.getRequestURI()).normalize().getPath();
return path.startsWith(request.getContextPath() + "/rest/")
    || path.startsWith(request.getContextPath() + "/anonymous/");
```

```xml
<!-- if DWR isn't actually used, delete the servlet block entirely -->
```

### 3. Strip CVV from logs everywhere, default payment logger to INFO — F007 / F046

In every `*TransactionImpl.java`, the `sblog` `StringBuffer` is built specifically to be logged. Either delete the `x_card_code` line in `sblog` (keep it in `sx`, the actual request body), or guard the whole `log.debug` with `log.isDebugEnabled()` AND a `cardLoggingEnabled` config flag that defaults false. Also update `log4j.properties:92` to `INFO` not `DEBUG`. Same edit in BeanStream/Psigate/Paypal. ~50 lines of edits across 4 files.

### 4. Patch the two arbitrary-action vulns — F005 / F006

```java
// BinUploadAction.deleteFile
File target = new File(deleteFilePath).getCanonicalFile();
File root   = futil.getMerchantBinRoot(merchantid).getCanonicalFile();
if (!target.toPath().startsWith(root.toPath())) throw new SecurityException("path");
futil.deleteFile(merchantid, target);
```

```java
// FilesAction.getFile
private static final Set<String> ALLOWED_MODS = Set.of("localFileModule", "s3FileModule");
if (!ALLOWED_MODS.contains(mod)) { return "GENERICERROR"; }
DownloadFileModule module = (DownloadFileModule) SpringUtil.getBean(mod);
```

Each is a one-file change.

### 5. Move to Maven, then bulk-bump deps — F018 / F019 / F004

Until there is a manifest, every CVE remediation is manual and unreproducible. Steps:

1. Generate `pom.xml` per module using `<dependency>` entries derived from the `.classpath` listing.
2. Resolve versions from the readme files under `lib/<group>/`.
3. `mvn dependency:tree` and reconcile (a few duplicates: e.g., `commons-collections` 2.1.1 in hibernate dir vs 3.2 in struts dir; `commons-lang-2.3` referenced from both `misc/` and `struts/`).
4. Bump in waves: log4j 1.2.16 → 2.x (or reload4j as a bridge), jackson 1.6 → 2.x, Struts 2.2.1.1 → 2.5.33+ (this is the painful one — DTD changes, OGNL allowlist, `paramsPrepareParamsStack` gone). iText 2.1.0 → OpenPDF 1.3 is near-drop-in. mysql-connector → 8.x.

Step 4 is L-effort but unblocks everything else. If it's deferred, the app has to live behind a strict WAF.

## Quick wins

- [ ] **F005** Path-traversal fix in `BinUploadAction.deleteFile` (10 lines).
- [ ] **F006** Allowlist `mod` in `FilesAction.getFile` (8 lines).
- [ ] **F010** Move FB HMAC secret to `MerchantConfiguration` (1 file edit, 1 row insert).
- [ ] **F021** Delete dead `if (session == null)` branch in `AuthFilter` (~30 lines deleted).
- [ ] **F022** Add `setSecure`/`setHttpOnly` on STORE cookie.
- [ ] **F023** Replace SQL-concat insert in `OrderDao.createRawOrder` with parameter binding.
- [ ] **F024** Fix `like %:cName%` HQL bug (this method is currently broken at runtime).
- [ ] **F025** Collapse the two identical if/else branches in `OrderDao.searchInvoice`/`searchOrder`.
- [ ] **F026** Flip `ge` to `le` for end-date guard in `OrderDao.searchOrder`.
- [ ] **F028** Swap c3p0 `max_size` / `timeout` properties in spring config.
- [ ] **F029** Externalize `hibernate.show_sql`.
- [ ] **F046** Fix log4j typos (`CINTEGRATIONlayout`, AXISFILE undeclared, INTEGRATION→sm-integration.log).
- [ ] **F049** Move `struts.properties` out of sm-core/conf.
- [ ] **F050** Decide on `salesManagerOrderService` (ship or delete commented mapping).

## Things that look bad but are actually fine

- **`<s:property value="invoiceUrl" escape="false"/>` in `invoicedetails.jsp:521`** — looks like XSS, but `invoiceUrl` is built server-side by `FileUtil.getInvoiceUrl` joining numeric IDs. Not user-influenced. *Required* category at F054 because of the inconsistency, but this specific site is OK.
- **`@SuppressWarnings("unchecked")` on every Hibernate-returning DAO method** — ugly, but unavoidable on Hibernate 3 + raw `Criteria.list()` returning `List`. The fix is generics in the DAO (F035), not removing the suppressions.
- **`FedexQuotesStubImpl.java:538` hardcoding `https://gatewaybeta.fedex.com:443/web-services`** — looked like a hardcoded prod URL but it's a sandbox endpoint for the test mode, switched per-merchant config elsewhere. Leave it.
- **`OrderDao.createRawOrder` writing `INSERT INTO orders(orders_id) VALUES (?)`** — looked like a layering violation (raw SQL in a Hibernate DAO), but the comment context implies this exists to seed an ID before Hibernate's identity-generator sees the entity. The pattern is questionable but intentional. Keep, just bind the parameter (F023).
- **`SecurityConstants.idConstant = "100"`** — looks like a magic number. It is, but its semantic role is "obfuscate the encryption key from the source code" — which doesn't work (F002). Calling it out as security debt, but not as "remove the constant" — the code that depends on it is what needs to go.
- **`CustomerLoginCallBackHandler` storing the cleartext password as a field** — looked dangerous, but JAAS callback handlers genuinely need this to satisfy the `PasswordCallback`. It's fine in scope; the real problem is what happens *with* the password downstream (F003).
- **`oscache-2.4.1.jar`** — old, but `OSCache` is the Hibernate L2 cache provider here and Hibernate 3.2 expects exactly this version. Don't bump in isolation; it has to move with Hibernate (F019/F041).
- **`Order.java` 1679 LOC** — flagged at F013, but I considered flagging the per-field setters/getters as "auto-generated bloat to delete" — and chose not to. They're Hibernate-required boilerplate; Lombok would help, but adding Lombok to a non-Maven build is more debt than it removes. Live with it until F018 lands.

## Open questions for the maintainer

1. **Is the `media` module abandoned?** It's an empty WAR shell with no Java sources, only `web.xml` + `build.xml`. Was it stubbed for a future media-server split that never happened?
2. **What is the threat model for `/dwr/*`?** If DWR isn't used by the JSPs (I see `<script src=".../dwr/...">` but didn't crawl all 268 JSPs), the servlet should be deleted. If it is used, the audit needs to grow a section on `dwr.xml` remoting policy.
3. **Is `salesManagerOrderService` removed on purpose?** The mapping in `sm-central/web.xml:58-63` is commented out, but `SalesManagerOrderWSImpl` and friends still compile. Half-removed feature?
4. **Why are there 0 jars in the lib tree but 80+ jar names in `.classpath`?** Was this repo lifted from somewhere else and only the manifests committed? Affects whether Maven migration (F018) is "translate manifests" or "redo from scratch."
5. **The README says "security fixes and enhancement only" but ships none — what is this branch's actual ship history?** No git, so I cannot tell. Has anything been deployed since 2010?
6. **Is the `Customer.customerPassword` column going to be migrated, or do you accept resetting all existing passwords?** F003 cutover strategy depends on this.
7. **Does any production deploy still exist?** If yes, F004 is acutely live (Struts 2.2.x is exploited by drive-by scanners). If no, audit priority shifts entirely toward F018 (Maven) as a prerequisite to anything else.
8. **`docs_bmad/`** — is this ours or someone else's pre-existing audit? If theirs, the gap analysis (Critical findings missed) is a finding about *that* document's reliability. If ours, this audit replaces it.
