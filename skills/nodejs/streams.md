# Stream Traps

- highWaterMark defaults: 16 KiB (generic streams), 64 KiB (`fs.createReadStream`), 16 items (objectMode). Raising it trades memory for fewer syscalls — a bulk file copy at 1 MiB does ~16x fewer reads than at 64 KiB; memory cost = highWaterMark × concurrent streams.
- `write()` returning false means the internal buffer passed highWaterMark. Ignoring it works — until sustained load buffers the entire source in RAM. Pattern: `if (!w.write(chunk)) await once(w, 'drain')`.
- `.pipe()` neither forwards errors nor cleans up on failure — `pipeline(src, ...transforms, dst)` does both, and accepts async generators as transforms (the modern way to write one-off Transforms).
- Any stream outside a pipeline needs its own `'error'` handler — an unhandled stream `'error'` is a synchronous process crash, not a rejected promise.
- `for await (const chunk of readable)` respects backpressure automatically; but `break`/`throw` inside the loop destroys the stream — you cannot resume it later.
- Don't mix consumption modes: attaching `on('data')` switches to flowing mode and races against pipe/async-iterator consumers of the same stream.
- `end()` vs `destroy()`: end flushes buffered data then closes; destroy drops it. Error paths want destroy (pipeline calls it for you) — calling end on error can flush half-written garbage downstream.
- HTTP responses are streams too: a client disconnect mid-response errors the destination — another reason request handlers use pipeline, not pipe.
