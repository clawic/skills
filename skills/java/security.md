# Security — The Java-Specific Attack Surface

Language-agnostic rules (validate input, least privilege, patch fast) apply here too. This file is the part that is specific to the JVM and its libraries.

## Deserialization

- Java serialization on untrusted input is remote code execution: `readObject` executes code from classes already on your classpath, and published gadget chains chain them into a shell. No amount of input validation fixes it, because the payload is the classes, not the data (`serialization.md`).
- If you cannot remove it: `ObjectInputFilter` with an **allowlist** (9+, backported to 8u121), set per stream or globally via `-Djdk.serialFilter=`. Denylists are always one gadget behind.
- The same class of hole in JSON: Jackson's `activateDefaultTyping` (and any `@JsonTypeInfo` accepting an arbitrary class name) lets the payload choose which class to instantiate. Restrict to a named subtype list.
- YAML: SnakeYAML's default constructor instantiates arbitrary classes. Use `new Yaml(new SafeConstructor())` or a schema-bound loader.
- XML external entities (XXE) read local files and reach internal hosts. Harden every parser you create:
  ```java
  var f = DocumentBuilderFactory.newInstance();
  f.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
  f.setFeature(XMLConstants.FEATURE_SECURE_PROCESSING, true);
  f.setXIncludeAware(false); f.setExpandEntityReferences(false);
  ```
  The same applies to `SAXParserFactory`, `XMLInputFactory` (`SUPPORT_DTD` false), and `TransformerFactory` (`ACCESS_EXTERNAL_DTD`/`STYLESHEET` empty).

## Injection

- SQL: `PreparedStatement` with `?` placeholders. String concatenation is injection even when the value "looks numeric". Table and column names cannot be parameterized — validate them against an allowlist, never interpolate input (`sql` skill for the query side).
- JPA/HQL: `createQuery("... where name = :n").setParameter("n", v)`. Native queries built by concatenation are exactly as dangerous as raw JDBC.
- Command execution: `ProcessBuilder` with a **list** of arguments never involves a shell, so there is nothing to escape. `Runtime.exec(String)` splits on whitespace with surprising rules; passing user input into `sh -c` reintroduces shell injection completely.
- Expression languages: SpEL, OGNL, MVEL, and JEXL evaluate arbitrary code. Never build an expression from user input — this is the shape of several high-profile framework CVEs.
- Log injection: user input with newlines forges log lines. And never log-and-format untrusted strings through a formatter that resolves lookups — a template engine or a logging pattern that interpolates input is a code-execution vector, as Log4Shell demonstrated.
- LDAP filters and JNDI names: an attacker-controlled JNDI lookup fetches and instantiates a remote class. Treat any `InitialContext.lookup` with dynamic input as remote code execution.

## Files and Paths

- Path traversal: `base.resolve(userInput)` returns an absolute path if the input is absolute, and `..` walks out. Validate as `Path p = base.resolve(input).normalize(); if (!p.startsWith(base)) reject;` — normalize BEFORE the check (`io.md`).
- Zip Slip: an archive entry named `../../etc/cron.d/x` writes outside the extraction directory. Apply the same containment check to every `ZipEntry.getName()`, and cap the entry count and decompressed size (a zip bomb is a denial of service).
- Temp files: `Files.createTempFile` creates with owner-only permissions atomically; `File.createTempFile` followed by a separate write in a world-writable `/tmp` is a symlink race.
- Never make an uploaded file executable, and never derive a filesystem path from a client-supplied filename — store under a generated id and keep the original name as metadata.

## Crypto Choices

| Need | Use | Not |
|---|---|---|
| Password storage | Argon2id or bcrypt (via a maintained library); PBKDF2-HMAC-SHA256 with a high iteration count when a JDK-only solution is required | `MessageDigest` SHA-256, salted or not — it is orders of magnitude too fast |
| Random tokens, ids, salts | `SecureRandom` | `Random`, `Math.random`, `UUID.randomUUID()` for security tokens (it does use a CSPRNG, but caps you at 122 bits in a fixed hex format — no control over length or alphabet) |
| Symmetric encryption | `AES/GCM/NoPadding` with a unique 12-byte IV per message | `AES/ECB` (patterns survive), `AES/CBC` without an authentication tag |
| Signatures/hashes | SHA-256 or better | MD5, SHA-1 |
| TLS | Default JSSE trust store and hostname verification | Any `TrustManager` that accepts everything |

- Never reuse a GCM IV with the same key — it destroys confidentiality AND allows forgery. Generate the IV with `SecureRandom` and transmit it alongside the ciphertext.
- `SecureRandom.getInstanceStrong()` maps to a blocking source on some Linux configurations and can stall startup in a fresh container with little entropy. `new SecureRandom()` is the non-blocking, still-cryptographic default.
- Disabling certificate or hostname verification "temporarily for the test environment" is how it reaches production. Import the internal CA into a trust store instead (`-Djavax.net.ssl.trustStore=`); debug handshakes with `-Djavax.net.debug=ssl:handshake`.
- Do not implement crypto primitives, key derivation, or token formats yourself. Use a library and a standard (JWT libraries: verify the algorithm explicitly, reject `none` and confusion between HMAC and RSA).

## Secrets in a JVM Process

- Never in source, never in a container image layer, never in a URL. Environment variables and mounted files are the normal answers; a secrets manager is better.
- A secret in a `String` stays in the heap until GC and lands in every heap dump. `char[]`/`byte[]` you can zero after use is marginally better and the honest reason is dump hygiene, not memory forensics.
- `-D` system properties appear in the process list and in `jcmd VM.system_properties` — anyone who can run `jcmd` as that user reads them.
- Exceptions and logs leak: never log a full request body, connection string, or token. Redact at the logger, not at each call site (`exceptions.md`).
- A heap dump is a secrets dump. Treat `HeapDumpPath` output as sensitive and restrict who can read it (`memory.md`).

## Exposed Interfaces

- JDWP (`-agentlib:jdwp`) is unauthenticated remote code execution. Bind to `127.0.0.1` and tunnel; never expose it (`debug.md`).
- JMX/RMI without authentication and TLS is the same class of hole. Require credentials, or bind to localhost only.
- Actuator-style management endpoints belong on a separate port that is not routed publicly (`spring.md`).
- The Attach API works for any process of the same user — a shared-user host means every JVM is debuggable by every other tenant.

## Supply Chain

- Scan dependencies in CI and rebuild periodically: a jar pinned for six months carries six months of published CVEs (`build.md`).
- Pin versions and prefer digests where the registry supports them; a mutable tag or a version range can be repointed after you vetted it.
- Verify the checksum or signature of anything downloaded during a build (`RUN curl | sh` in a Dockerfile has no place in a JVM build either).
- Generate an SBOM (CycloneDX plugins exist for both Maven and Gradle) so that "are we affected?" is a query, not an investigation.
