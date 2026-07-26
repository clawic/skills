# REST API — Automation Outside the Editor

The REST API reads a file's structure, renders images from it, and (on the highest tier) reads and writes variables. It is the right layer for audits, token exports, and pipelines that run in CI. It is not the layer for editing the document — document mutation happens in plugins.

## Authentication

- Personal access tokens carry scopes; issue the narrowest scope the job needs and nothing more. A read-only file-content scope cannot be turned into a write by a leaked script.
- OAuth is for anything multi-user: a token belonging to one designer becomes a single point of failure and an offboarding incident the day they leave.
- Never put a token in a plugin bundle (installable and readable), in a client-side app, or in a repo. Repo history retains it after deletion — rotate rather than delete.
- Store tokens in the CI secret store, and prefer a service account over a personal one for pipelines.

## The Endpoints That Matter

| Endpoint | Returns | Use it for |
|---|---|---|
| `GET /v1/files/:key` | The whole document tree | Rarely — it is enormous. Use `depth` to cap it |
| `GET /v1/files/:key?depth=1` | Top-level pages plus `lastModified` | Cheap change detection before a full pull |
| `GET /v1/files/:key/nodes?ids=` | Only the nodes requested | The default fetch. Targeted, small, fast |
| `GET /v1/images/:key?ids=&format=&scale=` | Rendered image URLs | Asset pipelines, visual regression baselines |
| `GET /v1/files/:key/components` | Published components in the file | Inventory, adoption reports |
| `GET /v1/files/:key/styles` | Published styles | Style-to-variable migration audits |
| Variables endpoints | Local variables and collections | Token export without a plugin — highest tier only |
| `GET /v1/files/:key/comments` | Comments | Routing review feedback into a tracker |
| Webhooks | Push events | Triggering pipelines on publish |

## Practical Mechanics

- **File key** comes from the URL: `figma.com/design/<key>/<name>`.
- **Node IDs** appear in the URL as `node-id=1-23` but the API expects `1:23`. Converting the dash to a colon is the single most common first-hour bug.
- **`depth`** limits how far the tree is walked. Fetching a page at `depth=1` to list frames, then fetching only the frames you care about, is orders of magnitude cheaper than one whole-file pull.
- **`geometry=paths`** adds vector path data. Omit it unless you are actually rendering geometry — it inflates the payload dramatically.
- **Image renders are asynchronous behind the scenes**: the endpoint returns temporary URLs that expire. Download immediately; do not store the URL.
- **Rate limits are per user and vary by endpoint cost.** Cache aggressively: store `lastModified` from a `depth=1` call and skip the pull when it has not changed. Batch node IDs into one request rather than looping one ID at a time.

## Webhooks

- Build pipelines on library-publish events, not on file-update events. File update fires on essentially every edit; library publish fires when something consumers should react to actually changed.
- Verify the payload signature and treat the webhook body as untrusted input.
- Make handlers idempotent — retries are normal and a token export that runs twice must produce the same result.

## What the API Cannot Do

- It does not edit the document. No creating frames, no renaming layers, no restyling. That is a plugin's job, and the two are complementary: API to detect, plugin to fix.
- Variables read and write access is tier-gated, which is the fork in every token pipeline: on the highest tier the API is the clean route; below it, a plugin export is the route that works.
- It does not give you prototype interaction data or analytics events.

## Pipelines Worth Building

- **Token export**: on library publish, pull variables (API where available, plugin export otherwise), transform into platform formats, open a pull request. Never let the pipeline commit straight to the main branch — a bad token rename should be reviewable.
- **Component inventory**: walk every product file's component instances, join against the library's published components, and produce adoption and detach numbers on plans without native analytics.
- **Asset pipeline**: render every icon component to SVG on publish and commit the diff, so engineers never export by hand.
- **Visual baseline**: render key frames to PNG on publish and diff against the previous render to catch unintended library changes.
- **Design-code drift check**: compare the file's token values against the values in the codebase and fail the build on divergence.

## Symptom → Cause

| Symptom | Cause | Fix |
|---|---|---|
| 404 on a node that exists | Dash-form node ID from the URL | Convert `1-23` to `1:23` |
| Response is hundreds of megabytes | Whole-file pull with geometry | Use `nodes?ids=` and drop `geometry=paths` |
| Rate limited in CI | Polling whole files on a schedule | Change-detect with `depth=1` and `lastModified`; move to webhooks |
| Image URLs return nothing later | Render URLs expire | Download in the same run |
| Variables endpoint returns forbidden | Tier gate | Export through a plugin instead |
| Pipeline fires constantly | Subscribed to file-update | Subscribe to library-publish |
| Token leaked in a repo | Committed and later deleted | Rotate; history still holds the old value |
| Script works for one person only | Personal access token | Move to OAuth or a service account |
