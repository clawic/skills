# Class Design — Records, Sealed Types, Inheritance, Object Lifecycle

Default shape for new types: **a record if it is data, an enum if the set is closed, a final class with an interface if it has behavior**. Reach for an extensible class hierarchy only when you can name the subclasses someone else will write.

## Records (16+)

```java
public record Money(BigDecimal amount, Currency currency) {
    public Money {                                     // compact constructor: validate and normalize
        Objects.requireNonNull(currency);
        amount = amount.setScale(currency.getDefaultFractionDigits(), RoundingMode.HALF_UP);
    }
    public Money plus(Money other) { ... }             // behavior is allowed and encouraged
}
```

- Generated: a canonical constructor, accessors named `amount()` (not `getAmount()`), `equals`, `hashCode`, `toString`. Implicitly final, cannot extend another class, can implement interfaces.
- The compact constructor CAN reassign the parameter (normalization, as above) — the assignment to the field happens after your block. It cannot assign `this.amount` directly.
- Records have no instance fields beyond their components; static fields and methods are fine.
- A record holding a mutable component (`List`, array, `Date`) is not immutable. Defensive-copy in the compact constructor AND in the accessor, or accept that callers can mutate your value object.
- Local records inside a method are the cheapest way to name an intermediate tuple in a stream pipeline.
- Serialization of records ignores `writeObject`/`readObject` and always goes through the canonical constructor — validation cannot be bypassed, which fixes a long-standing Java serialization hazard (`serialization.md`).

## Sealed Types and Pattern Matching (17+/21+)

```java
public sealed interface Shape permits Circle, Rect {}
public record Circle(double r) implements Shape {}
public record Rect(double w, double h) implements Shape {}

double area(Shape s) {
    return switch (s) {                                 // exhaustive: no default needed
        case Circle c            -> Math.PI * c.r() * c.r();
        case Rect(double w, double h) -> w * h;         // record deconstruction (21+)
    };
}
```

- Permitted subclasses must be in the same module (same package if unnamed module) and must each declare `final`, `sealed`, or `non-sealed`. `non-sealed` reopens that branch to anyone.
- Exhaustive switches over a sealed type are the point: adding a subclass turns every unhandled switch into a **compile error** instead of a runtime surprise. Do not add a `default` branch — it destroys that guarantee.
- Guards use `when`: `case Rect r when r.w() == r.h() -> ...`. Order matters, and the compiler rejects a case dominated by an earlier one.
- `case null` is allowed in a pattern switch (21+); without it, a null selector throws NPE as it always did.
- Sealed hierarchy + records is Java's algebraic data type. Use it for parse results, state machines, and command types — the places where an enum plus a payload used to be the ugly workaround.

## Inheritance, When You Actually Use It

- Design for extension or forbid it. A class that is neither `final` nor documented for subclassing is a maintenance trap: any overridable method called from the constructor runs against an uninitialized subclass.
- Never call an overridable method from a constructor, `clone()`, or `readObject()` — the subclass's fields are not yet assigned, so the override sees nulls and zeros.
- `static` methods hide rather than override: dispatch is by the reference type, not the object. Fields behave the same way — polymorphism applies only to instance methods.
- `private` methods are never overridden; a same-named method in a subclass is simply a different method.
- Always write `@Override`. It is the only compile-time check that your "override" matches a real signature (the classic miss: `equals(MyType)` instead of `equals(Object)`).
- Prefer composition + an interface. "Extends" ties you to the parent's invariants forever, and a parent's internal refactor breaks subclasses that did nothing wrong.
- Interface `default` methods are for evolving an interface without breaking implementors — not for sharing implementation. When two defaults conflict, the implementor must override and can pick with `Interface.super.method()`.

## Immutability Checklist

- All fields `private final`; no setters.
- The class is `final` (or all its constructors are private with static factories).
- Defensive-copy every mutable input in the constructor and every mutable output in a getter.
- Do not let `this` escape the constructor (`concurrency.md`).
- Result: safely publishable across threads with no synchronization, cacheable, and usable as a map key.

## equals, hashCode, toString, compareTo

- Covered by the contract in `collections.md`. Additional design rules:
- Compute both from the SAME set of final fields; a mismatch is a silent bug.
- `compareTo` should be consistent with `equals` — when it is not (`BigDecimal`), document it, because sorted collections behave differently from hash collections.
- `toString` is a debugging interface: include the identifying fields, never a secret, and never something that can throw (a `toString` NPE inside a log statement hides the original error).
- A record gives you all of these correctly; that is a reason to use one.

## Object Lifecycle and References

- `finalize()` is deprecated for removal and unreliable — it can run late, never, or on a starved thread while blocking GC. Never write one; `--finalization=disabled` (18+) is how you prove nothing depends on it.
- Deterministic cleanup is `AutoCloseable` + try-with-resources (SKILL.md rule 4). Everything else is a fallback.
- `java.lang.ref.Cleaner` is the safety net for native resources: register an action that does NOT capture the object being cleaned (a lambda referencing it keeps it alive forever, which is the mistake everyone makes on the first attempt).
- Reference strengths: strong → **soft** (cleared only under memory pressure; a badly-behaved cache substitute) → **weak** (cleared at the next GC; correct for canonicalizing maps) → **phantom** (never returns the referent; for post-mortem cleanup with a `ReferenceQueue`).
- `WeakHashMap` keys are weak, values are strong — a value referencing its own key pins the entry permanently.
- A non-static inner class holds a reference to its enclosing instance. That is how a small listener keeps an entire object graph alive (`memory.md`); use a `static` nested class or a lambda that captures only what it needs (capture and allocation rules: `lambdas.md`).

## Enums Do More Than You Think

- Constant-specific bodies give each constant its own implementation, which replaces a switch that must be updated in two places.
- Enums are singletons enforced by the JVM, including across serialization — the safest singleton in Java.
- `EnumMap`/`EnumSet` are array-backed and beat hash structures for enum keys (`collections.md`).
- `values()` returns a fresh clone on every call: hoist it out of loops and hot paths.
- Never persist `ordinal()`. Reordering constants silently changes the meaning of stored data; persist `name()` or an explicit code field.
