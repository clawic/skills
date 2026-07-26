# Performance and File Health

Figma runs in a browser tab with finite memory. A slow file is not bad luck; it is a ranked, diagnosable list of causes.

## Causes, Ranked

Work down this list — the order reflects how often each is the dominant cost in a real slow file.

1. **Raster images at original resolution.** Figma retains the source bitmap regardless of the layer's displayed size. Twenty 4000 px photos cropped to thumbnails cost the same as twenty full-size photos.
2. **Vector node count.** Auto-traced logos, imported maps, and illustration with thousands of points. A single traced logo can carry more nodes than an entire UI page.
3. **Effects.** Layer blur is the most expensive, followed by background blur, then stacked drop shadows. Three shadows on a component multiplied across a hundred instances is a measurable cost.
4. **Instance count and nesting depth.** Deeply nested components (an instance inside an instance inside an instance) re-resolve on every edit to any ancestor.
5. **Layer count on one page.** Many pages are cheap because pages load on demand; one page with tens of thousands of layers is not.
6. **Linked libraries.** Each linked library loads with the file. A file linking six libraries to use two of them pays for six.
7. **Fonts.** Each family and weight loads separately. A file that accumulated eleven families through copy-paste pays for all of them.

## Diagnosing

Figma exposes no profiler, so diagnosis is by isolation and by symptom shape:

| Symptom shape | Points at |
|---|---|
| Slow to open, fine once open | Images, linked libraries, fonts |
| Fine to open, laggy on every edit | Instance depth, layer count, effects |
| Laggy only on one page | That page's contents — bisect it |
| Laggy only when editing one component | That set is past its variant ceiling |
| Laggy only for some teammates | Their machine or browser, not the file |
| Slow scroll, fast edit | Effects and images in the viewport |

Bisect method: duplicate the file, delete half the pages, reopen. Repeat on the slow half. Three rounds usually names the page; a fourth names the frame.

## The Cleanup Pass

Run in this order; each step makes the next cheaper.

1. Downscale or flatten raster that will never be re-cropped, keeping editable originals on a hidden `_source` page.
2. Find and rebuild or rasterize high-node vectors. Auto-traced artwork almost always redraws smaller by hand.
3. Remove stacked effects that do not survive a side-by-side comparison at 100% zoom.
4. Unlink libraries the file does not actually consume.
5. Remove unused fonts by finding and restyling stray text nodes.
6. Move dead explorations to an `Archive` page, or out to an archive file if the page itself is huge.
7. Split by concern: foundations library, component library, product file, marketing file. A file that opens slowly is a file nobody opens.

## Preventing Regrowth

- A file has a working size. When one page exceeds what fits comfortably, split it by feature rather than letting it grow.
- Paste-special as plain content rather than pasting whole nested frames from other files, which drags their component and style dependencies along.
- Do not archive by hiding: hidden layers still load. Move them to an `Archive` page or delete them with a named version as the safety net.
- Component sets past their variant ceiling are a performance problem as well as a usability one; split them when editing one variant becomes noticeably slow.
- Prefer the desktop app for large files: it gets its own process and memory budget instead of competing with forty browser tabs.

## Symptom → Cause

| Symptom | Cause | Fix |
|---|---|---|
| File takes a minute to open | Original-resolution images, many linked libraries | Downscale raster; unlink unused libraries |
| Editing anything lags | Deep instance nesting, huge single page | Flatten nesting; split the page |
| Scrolling stutters, editing is fine | Blurs and stacked shadows in view | Remove effects that fail a 100%-zoom comparison |
| One page is the whole problem | Traced vectors or a giant frame | Bisect the page; rebuild or rasterize the artwork |
| Browser tab crashes | Memory ceiling | Desktop app; split the file |
| Only one component set lags | Combinatorial size | Split the set; move axes to properties |
| Hidden pages still slow the file | Hidden is not unloaded | Archive out of the file |
| File grew 3x after a paste | Pasted frames dragged dependencies in | Paste as plain content; audit new styles and components |
