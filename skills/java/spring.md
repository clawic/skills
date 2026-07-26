# Spring Boot — Proxies, Transactions, JPA, Configuration

Spring's surprises almost all come from one mechanism: **your bean is wrapped in a proxy, and the annotation only works when the call goes through it.** Learn that and half the mysteries disappear.

## Proxies: the Rule Behind the Mysteries

- `@Transactional`, `@Cacheable`, `@Async`, `@Retryable`, `@PreAuthorize` are implemented by a proxy around the bean.
- **Self-invocation bypasses it.** `this.doWork()` from another method of the same class calls the raw object: no transaction, no cache, no async, no error (SKILL.md Traps). Fixes, in order of preference: move the method to another bean; inject the bean into itself; or `AopContext.currentProxy()` with `exposeProxy = true` (last resort).
- `private`, `static`, and `final` methods cannot be advised. A `final` class cannot be CGLIB-proxied at all.
- A method called from a constructor or `@PostConstruct` runs before the proxy exists.
- Symptom checklist for "my annotation does nothing": is the call external? is the method public and non-final? is the feature enabled (`@EnableAsync`, `@EnableCaching`, `@EnableScheduling`)? is the class actually a bean?

## Transactions

- **Only unchecked exceptions roll back by default.** A checked exception thrown out of a `@Transactional` method COMMITS the transaction. Declare `@Transactional(rollbackFor = Exception.class)` when you throw checked exceptions.
- Catching an exception inside the transactional method means no rollback at all — the proxy never sees it.
- `UnexpectedRollbackException` on commit: an inner `@Transactional` method (propagation `REQUIRED`, so it joined your transaction) failed and marked it rollback-only; you caught the exception and carried on, and the outer commit then failed. Either let the exception propagate or make the inner method `REQUIRES_NEW`.
- `REQUIRES_NEW` uses a second database connection while the first is held. With a pool of 10, five nested calls deadlock the pool — size the pool for the nesting depth or avoid the nesting.
- `@Transactional(readOnly = true)` on queries lets Hibernate skip dirty checking and hints the driver/replica; it is not a security boundary.
- Transaction boundaries belong in the service layer, not the controller (too wide: HTTP time inside a DB transaction) and not the repository (too narrow: no atomicity across two writes).
- Anything non-transactional inside a transaction (an HTTP call, a message publish, a cache write) is not rolled back. Publish after commit (`TransactionSynchronization`/`@TransactionalEventListener(AFTER_COMMIT)`).

## JPA and Hibernate

- **Fetch defaults are asymmetric**: `@ManyToOne` and `@OneToOne` are EAGER by default; `@OneToMany` and `@ManyToMany` are LAZY. Set `fetch = LAZY` on every `@ManyToOne` — the default is why loading one entity pulls a graph.
- **N+1 queries**: one query for the list, one per element for the association. Detect by counting SQL statements (`spring.jpa.properties.hibernate.generate_statistics=true` or a datasource proxy in tests); fix with a `JOIN FETCH` query, an `@EntityGraph`, or a batch size (`hibernate.default_batch_fetch_size`).
- `LazyInitializationException` means the session closed before the association was touched. The fix is fetching what the caller needs in the query, not enabling `spring.jpa.open-in-view` — that holds a database connection for the entire request and converts a query bug into pool exhaustion (SKILL.md Traps).
- `save()` on an entity with a non-null id issues a SELECT and a MERGE, not an INSERT. For bulk inserts use `persist` semantics, batch inserts, and a flush/clear every N rows to keep the persistence context from becoming a memory leak (`memory.md`).
- Entity `equals`/`hashCode` on a generated id is broken before the insert (id is null) and breaks `Set` membership after it. Use a business key, or a UUID assigned in the constructor (`collections.md`).
- A dirty entity is flushed automatically at commit — an accidental setter call inside a transaction writes to the database with no `save()` in sight.
- `@Transactional` and lazy loading interact: outside a transaction, every lazy access is either an exception or a separate connection.

## Configuration and Profiles

- Precedence, highest first: command-line args → `SPRING_APPLICATION_JSON` → OS environment variables → `application-{profile}.yaml` (external, then classpath) → `application.yaml` → `@PropertySource` → defaults in code. When a value "is ignored", it is being overridden by something higher in this list.
- Relaxed binding maps `APP_DATA_SOURCE_URL` to `app.data-source.url`. Environment variables are therefore a first-class override in containers.
- `@Value("${a.b}")` fails fast on a missing property (good) but does no type-safe grouping; `@ConfigurationProperties` binds a whole tree, supports validation (`@Validated` + Bean Validation), and is the right default.
- Profile-specific files are ADDITIVE, not replacements: `application-prod.yaml` overrides keys from `application.yaml` and inherits the rest. A list property is replaced wholesale, not merged.
- Never put secrets in `application.yaml`. Environment or a secrets manager (`security.md`).
- `@ConditionalOnProperty`, `@ConditionalOnMissingBean` and friends are how starters back off; when a bean is unexpectedly missing, `--debug` prints the full auto-configuration report explaining every decision.

## Beans, Scopes, and Startup

- Component scanning starts at the `@SpringBootApplication` package and covers only that package and below. A bean in a sibling package is invisible — the most common "bean not found" (`debug.md`).
- Constructor injection over field injection: it makes dependencies explicit, allows `final` fields, and lets tests build the object without a container (`testing.md`).
- Circular dependencies are rejected by default since Boot 2.6. `spring.main.allow-circular-references=true` is a migration crutch; the real fix is extracting the shared logic.
- Injecting a prototype-scoped bean into a singleton gives you ONE instance forever. Use `ObjectProvider<T>` or a lookup method.
- Request/session-scoped beans injected into a singleton need a scoped proxy; outside a request thread (in `@Async` or a scheduler) they fail at access time.
- `@PostConstruct` runs before the proxy is in place and before the application is ready. Use `ApplicationReadyEvent` for work that must see a fully started context.
- Startup profiling: `spring.main.lazy-initialization=true` measures how much of startup is bean construction; the `ApplicationStartup` metrics (`BufferingApplicationStartup`) name the slow steps (`jvm.md`).

## Web and Testing Layer

- `@SpringBootTest` starts the whole context and caches it per unique configuration. Every distinct combination of properties, profiles, and mock beans creates ANOTHER context — the reason a suite takes minutes. `@DirtiesContext` evicts the cache and should be rare.
- Slice tests are the default choice: `@WebMvcTest` (controllers + MockMvc), `@DataJpaTest` (repositories with a transaction rolled back per test), `@JsonTest` (serialization contracts).
- `@MockitoBean` (Boot 3.4+; previously `@MockBean`) replaces a bean in the context — and every distinct set of mock beans forks the context cache.
- `RestTemplate` is in maintenance; `RestClient` (6.1+) is the synchronous successor and `WebClient` the reactive one. Whichever you use, set connect and read timeouts explicitly — the defaults are effectively infinite (`async.md`).
- Actuator endpoints go on a separate management port, and only `health` and `info` are exposed by default. Exposing `env`, `heapdump`, or `loggers` publicly is an information disclosure and, for `heapdump`, a secrets leak (`security.md`).
