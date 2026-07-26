# Ownership — Moves, Borrows, and Interior Mutability

Mental model: every value has exactly one owner; a borrow is a temporary, compiler-tracked promise that the owner outlives it. Almost every "the borrow checker is wrong" is one of three things: a borrow that lives longer than you think, a method that borrows more than you think, or two owners you have not admitted to yet.

## What Actually Moves

- Assignment, passing by value, returning, and pushing into a collection all move — unless the type is `Copy` (integers, floats, `bool`, `char`, shared references, and tuples/arrays of those).
- `for x in v` moves `v`; `for x in &v` borrows; `for x in &mut v` borrows mutably. `.iter()`/`.iter_mut()`/`.into_iter()` say the same thing explicitly, and `IntoIterator` for `&Vec<T>` is what makes the second form work.
- Closures capture per-variable and per-use: read-only use captures `&`, mutation captures `&mut`, a move-requiring use captures by value. `move ||` forces by-value for all of them — required for `thread::spawn` and any `'static` future.
- Partial move: moving one non-`Copy` field out of a struct makes the whole struct unusable as a whole, though the remaining fields stay individually usable. Destructuring in one statement (`let Foo { a, b } = foo;`) is the clean version.
- Deref does not move: `*boxed` in a value position moves out of the `Box` (allowed, `Box` is special), but `*rc` never can — `Rc`/`Arc` only hand out shared references.

## Borrow Rules, Precisely

Either one `&mut` or any number of `&`, never both, and no reference may outlive its referent. Two refinements that explain most surprises:

- **NLL**: a borrow ends at its last use, not at end of scope (`rust >=1.31` in edition 2018, and for all editions since 1.63). Reordering statements is therefore a real fix.
- **Field granularity**: direct field access borrows one field; a method call borrows all of `self`. `self.items.push(self.count)` compiles; `self.push_item(self.count())` does not. Extract the value first, or make the helper an associated function taking the fields it needs.

## Reborrowing

- Passing a `&mut T` to a function does not move it — the compiler inserts an implicit reborrow, and the original is usable again after the call returns.
- The reborrow is not inserted through generics: `fn takes<T>(t: T)` called with a `&mut U` moves the reference. Write `&mut *r` explicitly when a generic function eats your mutable reference.
- Returning a reborrow ties the output lifetime to the input, which is exactly what elision assumes for `&self` methods.

## Interior Mutability — What Each One Buys

| Type | Checked | Thread-safe | Panics / blocks | Use when |
|---|---|---|---|---|
| `Cell<T>` | Nothing to check (no references handed out) | No | Never | `T: Copy` or you only get/set whole values |
| `RefCell<T>` | At runtime, dynamically | No | Panics on overlap | Single-threaded graph, callback registry, cached field |
| `OnceCell<T>` / `OnceLock<T>` | Once, then immutable | `OnceLock` yes | Never | Lazy initialization of an expensive value |
| `Mutex<T>` | At runtime, by blocking | Yes | Blocks; poisons on panic | Shared mutable state across threads |
| `RwLock<T>` | At runtime, by blocking | Yes | Blocks; deadlocks on re-entrant read | Read-dominated shared state, measured |
| `Atomic*` | Hardware | Yes | Never | Single scalar: counter, flag, generation number |

- `RefCell` panics carry no location by default. `try_borrow_mut()` turns the panic into an `Err` you can attribute; in debug builds the panic message names the file and line of the *conflicting* borrow, which is the one you actually need.
- The safe pattern for "read a bit of shared state, then do work": clone the small piece out inside a block, drop the guard, then work. Holding a guard across a long computation is how a `RefCell` panic or a lock convoy is born.

## The Cases That Look Impossible

| Shape | Why it is refused | Standard answer |
|---|---|---|
| Doubly-linked list, tree with parent pointers | Every node would need two owners | Arena: `Vec<Node>` + `u32` indices; or `Rc` down + `Weak` up |
| Self-referential struct (a field borrowing another field) | Moving the struct invalidates the internal reference | Store an index or a range instead of the reference; `ouroboros`-style crates only when the API demands it |
| Callback holding a reference to its owner | The owner is borrowed for the callback's whole life | Pass the data to the callback as an argument, or store an id the callback resolves |
| Mutating a collection while iterating it | The iterator holds a borrow of the whole collection | Collect indices, then mutate; or `retain`, `drain`, `iter_mut` which are designed for it |
| Two `&mut` into one `Vec` | Aliasing cannot be proven by index | `split_at_mut`, `iter_mut`, or `[T]::get_disjoint_mut` on recent std |
| Builder that consumes `self` in a loop | Each call moves the builder | Take `&mut self` and return `&mut Self`, or reassign: `b = b.with(x);` |

## Drop Order (matters more than it looks)

- Local variables drop in **reverse** declaration order; struct fields drop in **declaration** order. A guard declared before the thing it protects therefore outlives it.
- `drop(x)` moves and drops immediately — the standard way to release a lock early without an extra block.
- A value assigned to `_` drops immediately (SKILL.md rule 7); a value bound to `_name` lives to end of scope.
- Temporaries in a `let` live to the end of the statement; temporaries in the tail expression of a block live to the end of the enclosing statement. Edition 2024 shortened the `if let` case: the scrutinee's temporaries now drop before the `else` block runs, which silently fixes a class of `if let Some(x) = m.lock().unwrap().get(k)` deadlocks — and changes behavior when you migrate.
- `Drop` cannot be called manually, cannot be async, and does not run on `std::process::exit`, on abort, or on a leaked value (`Box::leak`, `mem::forget`, an `Rc` cycle). Never make correctness depend on a destructor running.

## Reading a Borrow Error Fast

1. Which line creates the first borrow, which line creates the second, which line is the last use of the first? Those three spans are always in the message.
2. Can the last use move earlier? (rung 1)
3. Is one side a whole-`self` method call that could be a field access? (rung 2)
4. Does the code want two owners? Then choose the container deliberately (SKILL.md Pointer And Container Choice), do not discover it by trial.
