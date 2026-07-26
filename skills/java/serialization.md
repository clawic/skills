# Serialization — JSON, Records, and Why Java Serialization Is a Liability

Default choice for anything crossing a process boundary: **JSON (or a schema format like Protobuf/Avro) over Java serialization**, always. Java's built-in serialization is a remote code execution surface and a versioning trap.

## Jackson: the Nine Things That Break

1. **`ObjectMapper` is thread-safe and expensive to build.** Create ONE (`static final` or an injected bean) and reuse it. Constructing one per request is a measurable throughput bug (`performance.md`).
2. **Unknown properties fail by default.** A new field added by the producer breaks the consumer at runtime. Decide deliberately: `mapper.configure(FAIL_ON_UNKNOWN_PROPERTIES, false)` for tolerant consumers, or keep it strict and version the contract.
3. **Java 8 dates need the JSR-310 module**: register `JavaTimeModule` and disable `WRITE_DATES_AS_TIMESTAMPS`, or an `Instant` serializes as `1690000000.000000000` instead of an ISO string (`datetime.md`).
4. **Records work natively from Jackson 2.12+**; regular classes with a multi-argument constructor need `@JsonCreator`/`@JsonProperty` or the `-parameters` compiler flag, otherwise deserialization fails with "no suitable constructor".
5. **Generic types need a type reference.** `readValue(json, List.class)` gives you `List<LinkedHashMap>`; `readValue(json, new TypeReference<List<Foo>>() {})` gives you what you meant (`generics.md`).
6. **`@JsonIgnore` on the field also hides it from the getter** unless you annotate the accessor; the asymmetric case (`@JsonProperty(access = READ_ONLY)`) is what you usually want for passwords and ids.
7. **Getter naming drives the JSON name.** `getURL()` becomes `url` in some versions and `URL` in others; `isActive()` becomes `active`. Pin the wire name with `@JsonProperty("...")` on anything a client depends on.
8. **Polymorphic types**: `@JsonTypeInfo`/`@JsonSubTypes` with an explicit name list is safe; `enableDefaultTyping()`/`activateDefaultTyping` with a permissive validator is the Jackson equivalent of the Java-serialization gadget hole — it lets the payload choose the class to instantiate (`security.md`).
9. **Null vs absent**: `@JsonInclude(NON_NULL)` drops nulls, which makes "clear this field" indistinguishable from "leave it alone" in a PATCH. Model the tri-state explicitly (a wrapper or `JsonNullable`) rather than guessing.

## Designing a Wire Format That Survives

- Field names are a contract. Renaming one is a breaking change even if the Java field is private.
- Add fields as optional with a default; never repurpose an existing name for a new meaning.
- Enums on the wire: serialize `name()`, never `ordinal()`, and configure `READ_UNKNOWN_ENUM_VALUES_AS_NULL` (or a `@JsonCreator` fallback) so a new producer value does not break old consumers (`classes.md`).
- Numbers: send ids and money as strings when a JavaScript client is in the path — its numbers lose integer precision above 2^53.
- Dates as ISO-8601 strings with an offset. Epoch numbers require a documented unit and always get it wrong once (`datetime.md`).
- Round-trip test every DTO: serialize → deserialize → assert equality. It catches missing constructors, mismatched names, and lost precision in one assertion (`testing.md`).

## Java Serialization (`Serializable`)

Use only where a framework forces it (some caches, RMI, legacy session replication). Everything about it is sharp:

- **Deserializing untrusted data is remote code execution.** `readObject` runs code from classes on your classpath; gadget chains in common libraries turn a byte array into a shell. There is no safe way to deserialize an untrusted stream without a filter.
- Mitigation when you cannot remove it: an `ObjectInputFilter` (9+, backported to 8u121) with an allowlist — `jdk.serialFilter` as a system property for a global default, or per-stream via `setObjectInputFilter`. Allowlist classes; a denylist is always incomplete (`security.md`).
- `serialVersionUID`: declare it explicitly as `private static final long serialVersionUID = 1L`. Without it, the JVM computes one from the class shape, and adding a method changes it — old data then fails with `InvalidClassException`.
- `transient` fields are not written and come back as `null`/`0`. Anything derived must be recomputed in `readObject`.
- Custom `writeObject`/`readObject` must be `private void` with the exact signature; a typo makes the JVM silently ignore them.
- Deserialization bypasses constructors, so class invariants are not enforced — validate in `readObject`, or use `readResolve` to return a canonical instance (the singleton fix).
- A parent class that is not `Serializable` must have an accessible no-arg constructor, and its fields are reinitialized rather than restored.
- Records serialize through their canonical constructor, which restores validation and makes them the least-bad `Serializable` option (`classes.md`).

## Choosing a Format

| Format | Choose when | Cost |
|---|---|---|
| JSON (Jackson) | Public APIs, config, anything humans debug | Verbose; no schema unless you add one |
| Protobuf / Avro | High volume, cross-language, schema evolution matters | Build step, generated code, less greppable |
| CBOR / MessagePack | JSON model with smaller payloads | Not human-readable |
| Java serialization | A framework demands it | Security filter mandatory, version-fragile |
| Plain text / CSV | Bulk import-export with external tools | Quoting and charset edge cases (`text.md`) |

## Performance Notes

- Reuse `ObjectMapper`, `ObjectReader`, and `ObjectWriter` — the reader/writer variants are pre-resolved for a type and faster than the generic mapper.
- Stream large payloads: `mapper.readValue(InputStream)` and the `JsonParser` API avoid materializing the whole document; `readTree` on a 200 MB response is how a service OOMs (`memory.md`).
- Jackson's `afterburner`/`blackbird` modules remove reflection overhead; measure before adopting, since they interact with the module system (`migration.md`).
- Serialization usually shows up in profiles as reflection plus `String` allocation — the fix is fewer fields on the wire, not a faster library (`performance.md`).
