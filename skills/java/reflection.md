# Reflection and Annotations — Metadata, Handles, and Their Cost

Reflection is the fallback, never the default: it hides call graphs from the compiler, from static analysis, from dead-code elimination, and from every packaging step that rewrites or prunes names. Reach for it only after the table below says nothing else fits.

## Pick the Weakest Tool That Works

| Need | Use | Not |
|---|---|---|
| Metadata known at compile time | An annotation processor that generates code (`build.md`) | Scanning the classpath at startup |
| Plug in implementations | `ServiceLoader` + `META-INF/services` | A package scan |
| Call one known method dynamically | A `static final MethodHandle` resolved once | `Method.invoke` looked up per call |
| Atomic access to a field you do not own | `VarHandle` (9+) | `sun.misc.Unsafe`, `AtomicReferenceFieldUpdater` |
| Intercept every call to an interface | `java.lang.reflect.Proxy` | Hand-rolled bytecode generation |
| Read a record's shape | `getRecordComponents()` (16+) | Guessing accessor names from fields |
| Reach a private member of your own code, in a test | A package-private accessor or a test in the same package | `setAccessible(true)` |

Cost order, worth re-measuring per JDK (`performance.md`): direct call < cached constant `MethodHandle` < cached `Method.invoke` << `getMethod` on every call. `Class.getMethods()` returns a defensive copy of the array each time it is called; caching the lookup is most of the win.

## Custom Annotations

```java
@Retention(RetentionPolicy.RUNTIME)          // CLASS is the default and is invisible to reflection
@Target({ElementType.TYPE, ElementType.METHOD})
@Inherited                                    // TYPE only, superclass chain only
public @interface Audited {
    String value() default "";
    Level level() default Level.INFO;
}
```

- **Forgetting `@Retention(RUNTIME)` is the number one cause of "my annotation is not found":** `getAnnotation` returns null with no error, because `CLASS` retention keeps it in the bytecode and drops it at load time.
- Element types are restricted to primitives, `String`, `Class`, enums, other annotations, and single-dimension arrays of those. There is no null default — use `""`, a sentinel enum constant, or an empty array.
- `@Inherited` applies only to class-level annotations and only up the superclass chain; interfaces never propagate. Framework "meta-annotation" search (Spring's `@AliasFor`, composed annotations) is library behavior, not JDK behavior — do not assume it outside that framework (`spring.md`).
- `@Repeatable(Audits.class)` requires you to declare the container annotation, and repeats are read with `getAnnotationsByType`, never `getAnnotation`.
- `ElementType.TYPE_USE` (8+) is what lets `@Nullable` sit on `List<@Nullable String>`; it is the target nullability checkers expect (`nulls.md`).
- Annotating parameters is useful only if you also compile with `-parameters` when you need their names (`build.md`).

## Reading Metadata at Runtime

- `getDeclaredMethods()` = declared on this class, any visibility. `getMethods()` = public, including inherited. Mixing them up is the usual "my private method does not exist".
- The order of `getDeclaredFields()`/`getDeclaredMethods()` is **unspecified** and has changed between JDK builds. Sort explicitly if output depends on it — otherwise a serializer's field order flips on a JDK upgrade and a golden-file test fails for no visible reason (`testing.md`).
- Bridge and synthetic methods show up in the results: a generic override appears twice unless you filter `isBridge()` and `isSynthetic()`.
- Declarations keep their generic signature (`getGenericReturnType`, `getGenericType`); values never do (`generics.md`).
- Names: for `a.b.Outer.Inner`, `getName()` is `a.b.Outer$Inner`, `getSimpleName()` is `Inner`, `getCanonicalName()` is `a.b.Outer.Inner`, and arrays render as `[Ljava.lang.String;`. `Class.forName` accepts only the binary form with `$`.
- `getSimpleName()` is empty for an anonymous class and misleading for lambdas — never key a registry on it.

## setAccessible and Strong Encapsulation

- Since 16/17, `setAccessible(true)` into JDK internals throws `InaccessibleObjectException`; the escape hatch and where to put the flags are in `migration.md`.
- Classpath code (the unnamed module) is still fully open to itself. The wall is `java.*` and modularized libraries, not your own packages.
- Do not use it to defeat `final`: `Field.set` on a `static final` throws `IllegalAccessException` even after `setAccessible`, and any value the JIT already constant-folded would not change if it did not. Record components reject `setAccessible` outright.
- Reflection in tests is a refactor bomb: a rename compiles green and fails at runtime. Widen the member to package-private and put the test in the same package instead.

## MethodHandles and VarHandle

```java
private static final MethodHandle LENGTH = MethodHandles.lookup()
        .findVirtual(String.class, "length", MethodType.methodType(int.class));
```

- Resolve once into a `static final` field. A constant handle is inlinable by the JIT nearly down to a direct call; a handle stored in an instance field or resolved per call is not.
- `invokeExact` demands that the call-site signature match exactly, boxing included; `invoke` inserts conversions. A `WrongMethodTypeException` from `invokeExact` is almost always an unexpected box or an `Object` return type.
- `VarHandle` (9+) is the supported replacement for `Unsafe` and the atomic field updaters, with explicit memory semantics per access — `getVolatile`, `setRelease`, `compareAndSet`, `getAcquire` (`concurrency.md`).
- `MethodHandles.privateLookupIn(Target.class, lookup())` is how a framework legally obtains private access; it still requires the target module to `open` that package.

## Dynamic Proxies

- `Proxy.newProxyInstance(loader, new Class<?>[]{Api.class}, handler)` implements **interfaces only**. Proxying a class needs a bytecode library (ByteBuddy, CGLIB) — which is exactly what Spring falls back to when a bean has no interface (`spring.md`).
- Everything routes through `invoke(proxy, method, args)`, including `equals`, `hashCode`, and `toString`; a handler that forwards them blindly to the target breaks proxy identity. Default methods must be dispatched explicitly with `InvocationHandler.invokeDefault` (16+).
- Unwrap `InvocationTargetException.getCause()` before logging, or the trace shows the reflection plumbing and not the failure (`debug.md`). An exception the interface does not declare surfaces to the caller as `UndeclaredThrowableException`.
- Generated proxy classes and their classloaders are a standard metaspace leak in redeploy-heavy servers (`memory.md`).

## Annotation Processing (compile time)

- A processor runs inside `javac` over `javax.lang.model` and may only ADD files through the `Filer` — never modify an existing one. Lombok edits the compiler's AST through internal APIs, which is precisely why it breaks on JDK upgrades (SKILL.md Where Experts Disagree).
- Declare the processor path explicitly: Maven `<annotationProcessorPaths>`, Gradle's `annotationProcessor` configuration. A processor sitting only on the compile classpath is not picked up by Gradle, and `javac` warns about implicit discovery from 21 and stops running implicitly discovered processors from 23 — pass `-proc:full` or declare the path (`build.md`).
- Processing runs in rounds: files generated in one round are themselves processed in the next, which is how generated code can carry annotations.
- Generated sources land in `target/generated-sources/annotations` or `build/generated/sources/annotationProcessor`. Register them with the IDE or you get red editors over a green build (`debug.md`).
- `-proc:none` on a module that needs no processing measurably shortens a large multi-module build.

## Making Reflection Survive Packaging

- Shading relocates package names; every class referenced by a string literal breaks, and so does every resource path inside the relocated library (`build.md`).
- `META-INF/services` entries must be merged by the shade plugin (`ServicesResourceTransformer`) or the last jar wins and `ServiceLoader` silently finds nothing — this is a top cause of `ClassNotFoundException` for a driver or provider that exists in the fat jar (SKILL.md Exception Triage).
- `minimizeJar` and obfuscators delete classes reachable only by reflection. Keep an explicit retain list.
- GraalVM native images need every reflective target declared in `reflect-config.json`; generate it with the tracing agent from a real run rather than by hand (`jvm.md`).
