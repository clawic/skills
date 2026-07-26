# Files, Paths, and Streams

`java.nio.file` (7+) replaces `java.io.File` for everything: it reports real errors instead of returning `false`, handles symlinks and attributes, and works with `Path`.

## Reading and Writing

```java
String s        = Files.readString(path, UTF_8);            // 11+, whole file
List<String> ls = Files.readAllLines(path, UTF_8);          // whole file
try (var lines  = Files.lines(path, UTF_8)) { ... }         // streamed, MUST be closed
Files.writeString(path, s, UTF_8, CREATE, TRUNCATE_EXISTING);
try (var w = Files.newBufferedWriter(path, UTF_8)) { ... }
```

- Pick by size: `readAllLines`/`readString` hold the whole file in memory (SKILL.md rule 5's budget applies); `Files.lines` and `newBufferedReader` hold one line.
- `Files.lines`, `Files.walk`, `Files.list`, and `Files.find` all return streams backed by an open handle. Outside try-with-resources they leak descriptors and fail hours later with "Too many open files" (SKILL.md Traps).
- Every read/write API takes a charset — pass the configured `default_charset` (SKILL.md Configuration; UTF-8 unless the user set another). The no-charset overloads use the platform default, which changed in JDK 18 (`text.md`).
- Buffer explicitly for streams you construct: an unbuffered `FileOutputStream` writes syscall-per-byte. `Files.newBufferedWriter`/`newInputStream` + `BufferedInputStream` handles it.
- `Files.copy(in, path, REPLACE_EXISTING)` and `in.transferTo(out)` (9+) beat a hand-rolled buffer loop.

## Path Semantics

- `Path.of("a", "b")` / `path.resolve("b")` — never string concatenation with `/`, which breaks on Windows and on trailing separators.
- **`resolve` with an absolute argument returns that absolute path**, discarding the base. This is the mechanism behind most path-traversal bugs (`security.md`).
- `normalize()` collapses `.` and `..` textually; `toRealPath()` resolves symlinks and requires the file to exist. Validate containment with `base.resolve(user).normalize().startsWith(base)` — after `normalize`, never before.
- `path.getFileName()`, `getParent()`, and `getFileName().toString()` beat manual index arithmetic on the string.
- Case sensitivity differs by filesystem: macOS is usually case-insensitive, Linux is not. Two files differing only by case is a bug that only reproduces in production (`debug.md`).
- `File.delete()` returns a boolean nobody checks; `Files.delete(path)` throws `NoSuchFileException`/`DirectoryNotEmptyException` with the reason. `Files.deleteIfExists` when absence is fine.
- `Files.move(a, b, ATOMIC_MOVE)` is atomic only within the same filesystem; across mounts it throws, so a fallback copy+delete is not atomic.
- Write-then-rename is the durable-update pattern: write to a temp file in the SAME directory, `force()`/close, then `ATOMIC_MOVE` over the target.

## Directories

- `Files.createDirectories(p)` is idempotent; `createDirectory` throws if the parent is missing.
- Walking: `Files.walk` is depth-first and follows no symlinks unless asked; `Files.walkFileTree` with a `FileVisitor` is the only way to handle errors per entry (a permission error mid-walk aborts a `Files.walk` stream).
- `Files.newDirectoryStream(dir, "*.json")` is the cheap listing with a glob; it is also closeable.
- Deleting a tree: walk in reverse order (`Files.walk(p).sorted(Comparator.reverseOrder())`) so children go before parents.
- Watching: `WatchService` is polling-based on macOS with multi-second latency, native on Linux/Windows. Do not build tight-latency features on it.

## Temp Files and Cleanup

- `Files.createTempFile(dir, prefix, suffix)` creates the file with owner-only permissions atomically. `File.createTempFile` in a shared `/tmp` plus a later write is a classic symlink race (`security.md`).
- `deleteOnExit()` leaks: entries accumulate for the JVM's lifetime and never run on `kill -9`. Use try-with-resources and delete in `finally`.
- For a scratch directory, create it under a path you control, and delete the tree explicitly — the OS cleaner runs on its own schedule.

## Classpath Resources (the "works in the IDE" trap)

```java
try (InputStream in = MyClass.class.getResourceAsStream("/config/default.yaml")) { ... }
```

- Leading `/` = absolute from the classpath root; no slash = relative to the class's package. Getting this wrong returns `null`, not an exception — always null-check with a message naming the resource.
- **A resource inside a jar is not a file.** `getResource(...).getFile()` or `Paths.get(url.toURI())` works from `target/classes` and throws once packaged (`debug.md`). Always read the stream.
- Duplicate resource names across jars: `getResource` returns the first on the classpath. Use `getResources()` (plural) when you expect several, as SPI does.
- `ClassLoader.getResourceAsStream` never takes a leading slash; `Class.getResourceAsStream` does. Mixing them is why the same path works in one class and not another.
- Reading a resource into a string: `new String(in.readAllBytes(), UTF_8)` (9+).

## Streams, Readers, and Closing

- Byte streams (`InputStream`/`OutputStream`) for binary, character streams (`Reader`/`Writer`) for text — bridged by `InputStreamReader`/`OutputStreamWriter`, which is where the charset belongs.
- Closing the outermost wrapper closes the chain and flushes buffers. Data missing from the tail of a file is almost always an unflushed writer that was never closed (`exceptions.md`).
- `System.out`/`err` are `PrintStream`s that swallow `IOException` and auto-flush on newline — convenient for a CLI, wrong for a data path.
- Never close a stream you did not open (a servlet's output, a caller-supplied stream): closing someone else's resource breaks their code.
- `transferTo` does not close either side. Wrap both in try-with-resources.

## Memory-Mapped and Large Files

- `FileChannel.map` gives near-memory speed for large random access, but the mapping is not released deterministically — it is freed when the buffer is collected, which is why "the file cannot be deleted on Windows" after processing.
- `MappedByteBuffer` counts against native memory, not heap; it is invisible to a heap dump (`memory.md`).
- For sequential bulk processing, a buffered stream is simpler and nearly as fast. Map only for random access to files larger than the heap.
- Files above 2 GB need `long` offsets throughout: any `int` position or size is an overflow waiting for a big customer.
