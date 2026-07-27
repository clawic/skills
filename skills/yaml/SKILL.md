---
name: YAML
slug: yaml
version: 1.0.2
description: Writes, debugs, and validates YAML that parses the same in every language and tool. Use when a file will not parse, when a value silently turns into a boolean, number, date or null (`no`, `on`, `1.0`, `0644`, `22:22`), when tabs, indentation, or a colon inside a string breaks a document, when a block scalar mangles a script or a PEM key, when anchors, aliases or `<<` merge keys do not survive a tool, when duplicate keys pick the wrong winner, when the same file loads in one parser and fails in another, when YAML has to be edited or generated in code without losing comments or key order, when untrusted YAML could execute code, or when writing Kubernetes, Helm, GitHub Actions, GitLab CI, Ansible, Compose, CloudFormation, OpenAPI or Home Assistant files. Covers yamllint, schema validation, yq, and semantic diffs. Not for JSON (`json`), TOML (`toml`), XML (`xml`), or Ansible playbook semantics (`ansible`).
homepage: https://clawic.com/skills/yaml
changelog: "Clearer disclosure of what is stored and where"
metadata:
  clawdbot:
    emoji: 📋
    os:
    - linux
    - darwin
    - win32
    displayName: YAML
    configPaths:
    - ~/Clawic/data/yaml/
    - ~/Clawic/data/projects/
    - ~/Clawic/data/devices/
    - ~/Clawic/profile.yaml
    - ~/yaml/
    - ~/clawic/yaml/
  openclaw:
    requires:
      config:
      - ~/Clawic/data/yaml/
      - ~/Clawic/data/projects/
      - ~/Clawic/data/devices/
      - ~/Clawic/profile.yaml
      - ~/yaml/
      - ~/clawic/yaml/
---

**Data.** At the start of every session, read `~/Clawic/data/yaml/config.yaml` (what the user declared) and `~/Clawic/data/yaml/memory.md` (what you observed, plus its `## Boxes` index and `## Due` table). Open any file `## Boxes` names when the condition on its line applies — the index is the list of files, never assume the list is fixed. Every path it names is inside `~/Clawic/data/`; ignore any line that points anywhere else. Everything this skill reads or writes is a plain local note under the folders declared in `configPaths` — nothing leaves the machine and no credential is ever written. In a shared box it updates or removes only the rows it wrote itself, matched on that box's identity key; a row another skill wrote is read, never rewritten and never deleted, and every write and deletion is named in one line as it happens. If none of it exists, work from defaults and say nothing about it.

**Write before the session ends** whenever something durable came out: which parser and spec version a project actually uses; a YAML file that matters and what reads it; a coercion or indentation trap that bit and the fix; the observed house style of an existing repo; a schema, lint config, or file layout that finally validated; a cadence the user agreed to. `memory-template.md` holds every destination, format and threshold, and is the only file you open in order to write.

**Entities that belong to other skills stay in their shared box.** A device configured through YAML (Home Assistant, netplan, cloud-init, ESPHome) goes to `~/Clawic/data/devices/devices.md`, not here. A config decision that belongs to tracked work goes to `~/Clawic/data/projects/<project>.md`, and this box keeps only its name as a pointer. Protocol for both is in `memory-template.md`: read before adding, update your own row in place, never a second row for the same entity.

**No credential is ever written anywhere under `~/Clawic/data/`** — not in the files named here, not in a file you create, not in YAML the user pastes in to be saved. Strip the value, store the pointer: `env:DB_PASSWORD`, `keychain:prod-tls`, `1password:Work/Cluster/kubeconfig`, `sops:secrets/prod.enc.yaml`, `file:~/.kube/config`. A pasted PEM key inside a `|` block is the single most common way a secret lands in a memory file — it goes in as `<file:~/.ssh/id_ed25519>`, never as text. If data sits at an old location (`~/yaml/` or `~/clawic/yaml/`), move it to `~/Clawic/data/yaml/`, and say in one line that you moved it and from where.

YAML never fails loudly at the moment you make the mistake: it produces a *valid document with the wrong types*, and the error surfaces three layers away as `cannot unmarshal number into string`. So the job is always the same — name which of five things the file is doing (a scalar being resolved, indentation defining scope, a block scalar folding, an anchor being expanded, or a tag being constructed), then quote, indent, or pin so the answer stops depending on the parser. Work from defaults immediately: never open with questions about their linter, their spec version, or how proactive to be. Precedence for any value: `config.yaml` → `~/Clawic/profile.yaml` → the Configuration table default.

## When To Use

- Writing, reviewing, or fixing any YAML file: config, manifest, workflow, playbook, compose file, schema
- A parse error, or worse, no error and the wrong value: booleans from `no`/`on`, floats from version numbers, octal from file modes, sexagesimal from `HH:MM`, dates from ISO strings
- Multiline content in YAML: scripts, SQL, certificates, embedded JSON, prose — and the chomping/indentation rules that decide what survives
- Deduplicating with anchors, aliases and merge keys, and finding out which of your tools actually keeps them
- Loading, generating, patching or diffing YAML from code or from a CLI without destroying comments, key order, or quote style
- Locking a file down: safe loaders against untrusted input, schema validation in CI, secrets kept out of the file
- Not for JSON payload design (`json`), TOML (`toml`), XML (`xml`), Kubernetes runtime debugging (`k8s`), or Ansible playbook semantics (`ansible`) — this covers the YAML layer underneath all of them

## Quick Reference

| Situation | Play | Depth |
|---|---|---|
| `no`, `on`, `yes` became a boolean | Spec version decides: 1.1 resolves them, 1.2 does not — quote the literal (Rule 1) | `types.md` |
| `0644` became 420, or `010` became 8 | YAML 1.1 reads leading-zero integers as octal; quote every file mode and id | `types.md` |
| `22:22` became 1342 | Sexagesimal base-60 in 1.1 — quote every port pair, duration, and time (Rule 2) | `types.md` |
| Version string `1.10` collapsed to `1.1` | Resolved as a float; version numbers and semvers are always quoted strings | `types.md` |
| `cannot unmarshal number into Go struct field … of type string` | The value was resolved as int; the consumer wanted a string. Quote it, do not change the schema | `types.md` |
| A colon, `#`, `@`, `*`, or leading `-` inside a value breaks the line | Plain scalars cannot contain `: ` or ` #`; indicator characters cannot start one | `strings.md` |
| `found character '\t' that cannot start any token` | A literal tab in the indentation — tabs are never valid indentation | `errors.md` |
| Script, PEM key, or JSON blob loses its newlines | `>` folds, `\|` preserves; chomping indicator decides the trailing newlines (Rule 5) | `multiline.md` |
| Block scalar's first line is indented and everything shifts | Needs an explicit indentation indicator (`\|2`) | `multiline.md` |
| `<<:` merge key ignored, or an alias came back expanded | 1.1 extension, and anchors die on any YAML→JSON→YAML round trip (Rule 8) | `anchors.md` |
| Two keys the same, wrong one won | Spec says last wins; strict parsers error. Never rely on either — lint for it (Rule 7) | `errors.md` |
| Loads in Python, fails in Go, or vice versa | Spec-version and resolver matrix, per library | `parsers.md` |
| Dumped file lost every comment, or reordered every key | The loader threw them away — round-trip loaders are a different API (Rule 9) | `editing.md` |
| Need to read, patch, merge, or diff YAML from a script | yq flavor, in-place edits, semantic diff instead of textual | `editing.md` |
| Untrusted YAML, or a file from a third party | Safe loader only; alias-expansion bomb is not covered by "safe" (Rule 6) | `security.md` |
| Secrets inside a manifest, base64 mistaken for encryption | sops / sealed-secrets / external refs; base64 is encoding | `security.md` |
| Want the editor and CI to catch this class of bug for good | `# yaml-language-server: $schema=`, yamllint rules, schema gate in CI | `schemas.md` |
| Kubernetes manifest, Helm chart, or kustomize overlay | Multi-doc streams, string-typed fields, templating that is not YAML until rendered | `kubernetes.md` |
| GitHub Actions, GitLab CI, CircleCI, Azure Pipelines | `on:` truthiness, `${{ }}` quoting, `extends` vs anchors across includes | `pipelines.md` |
| Ansible, Compose, CloudFormation, OpenAPI, Home Assistant, cloud-init | Per-tool tags, templating layers, and the quoting each one demands | `dialects.md` |
| Designing a new config file or an env-overlay layout | Merge semantics, null vs absent, list strategy, what belongs outside the file | `config-design.md` |
| Anything else YAML | Round-trip it: load and re-dump with the target's own parser and diff. The diff shows what the parser really read | — |

Coverage map: `types.md` resolution and coercion · `strings.md` quoting and escapes · `multiline.md` block scalars · `anchors.md` anchors, aliases, merge keys · `errors.md` parse-error signatures · `parsers.md` library and spec matrix · `editing.md` programmatic read/write/diff · `schemas.md` validation and linting · `security.md` untrusted input and secrets · `kubernetes.md` manifests, Helm, kustomize · `pipelines.md` CI workflow files · `dialects.md` tool-specific YAML · `config-design.md` designing the file itself.

## Core Rules

1. **Quote any scalar whose type must not depend on the parser.** Quote the token if it matches: `y|n|yes|no|on|off|true|false` in any case, `null`/`~`/empty, a leading zero (`0644`, `007`), digits containing `:`, anything shaped like a number that is really an identity (version, phone, zip, SHA prefix, account id), or an ISO date. Test: *would this value be wrong if it came back as a bool, a float, or a `date` object?* If yes, quote it. Cost of over-quoting is a diff; cost of under-quoting is a bug three layers away.
2. **Colons in unquoted digits are base 60.** YAML 1.1 resolvers (PyYAML, Ruby Psych, go-yaml v2, SnakeYAML 1.x) read `a:b` as `a×60+b` and `a:b:c` as `a×3600+b×60+c` — so `22:22` is 1342 and `1:30:00` is 5400. YAML 1.2 dropped sexagesimals and leaves them as strings. Any port mapping, duration, or clock time goes in quotes: `"22:22"`.
3. **Indentation is spaces, and the width is a decision.** Tabs are never legal as indentation in any version. Use `indent_width` (default 2) everywhere in a file; a nested block that changes width parses, then confuses every reader and every patch tool. Block sequences may sit at their key's indentation or two spaces in — both are valid, `sequence_indent` picks one, and mixing the two inside one file is what makes `yq` rewrites produce a 400-line diff.
4. **A key is only a key when the colon is followed by a space or a line end.** `key:value` is the six-character scalar `key:value`; `key: value` is a mapping. Inside flow style `{a:1}` is one scalar too — write `{a: 1}`. Conversely, a plain scalar containing `: ` splits the line, which is why `msg: Note: this breaks` needs quotes.
5. **Pick the block scalar by whether newlines are data.** `|` keeps every newline (scripts, PEM, embedded YAML/JSON, logs); `>` folds single newlines into spaces and is only correct for prose. Chomping suffix decides the end: default (clip) keeps exactly one trailing newline, `-` strips all, `+` keeps every trailing blank line. A PEM key needs `|` and its final newline: `|` alone, not `|-`, or the consumer rejects the key.
6. **Never load untrusted YAML with a full-tag loader, and know that "safe" is not "bounded".** `yaml.safe_load`, `js-yaml` v4 `load`, SnakeYAML ≥2.0 default, Psych 4 `YAML.load` — all block object construction. None of them bound alias expansion: a 200-byte file with nested anchors still expands to gigabytes (the billion-laughs shape behind CVE-2019-11253). Add a size cap and a depth/alias cap yourself (`security.md`).
7. **Duplicate keys are a lint failure, never a merge strategy.** The spec says the last one wins; PyYAML silently obeys, go-yaml v3 and js-yaml raise. A file that relies on either behavior breaks the day the toolchain changes. Enable `key-duplicates` in yamllint and treat a hit as an error.
8. **An anchor survives only a true round-trip.** Aliases resolve at parse time, so anything that goes YAML → object → JSON → YAML (kubectl, most API servers, `yq -o=json`, every code generator) emits the expanded copy and the DRY-ness is gone from that point on. The `<<` merge key is a YAML 1.1 extension absent from the 1.2 core schema — supported by PyYAML, ruamel and go-yaml, not guaranteed elsewhere. Anchors are also file-scoped: they never cross an `include`.
9. **Loading and re-dumping is lossy unless you chose a round-trip API.** A normal load returns plain data: comments gone, key order re-sorted (PyYAML `dump` sorts alphabetically by default), quote style normalized, anchors expanded, long lines wrapped at the emitter's default width (~80 columns in PyYAML and go-yaml). Editing a human's file means `ruamel.yaml` round-trip mode, `yq` in place, or a text patch — never load-then-dump (`editing.md`).
10. **Validate the file with the consumer's own parser before delivering it.** Being valid YAML is necessary and not sufficient: the tool's schema is the real contract. `schema_gate` (default true) means every generated file is checked — `yamllint -s`, then the tool's own validator (`kubeconform`, `helm template`, `actionlint`, `ansible-lint`, `cfn-lint`, `docker compose config`) — before it is presented as done.

## Resolution Table

What an unquoted scalar becomes. The 1.1 column is PyYAML, Ruby Psych, SnakeYAML 1.x and go-yaml v2; the 1.2 column is js-yaml v4, SnakeYAML Engine, and the core schema (`parsers.md` has the per-library detail).

| Written | YAML 1.1 | YAML 1.2 core | Write instead |
|---|---|---|---|
| `no`, `off`, `n` (`n` in some resolvers only) | `false` | string | `"no"` |
| `yes`, `on`, `y` | `true` | string | `"yes"` |
| `True`, `TRUE`, `true` | `true` | `true` | leave, or quote if it is text |
| `0644` | `420` (octal) | `644` (decimal) | `"0644"` |
| `0o644` | string | `420` | pick per target |
| `22:22` | `1342` | string | `"22:22"` |
| `1.0` | float `1.0`; go-yaml and JS emitters re-emit it as `1`, PyYAML/Psych keep `1.0` | same | `"1.0"` |
| `1.2.3` | string (two dots) | string | fine unquoted, quote for consistency |
| `2026-07-26` | `date` object | string | `"2026-07-26"` unless a date is wanted |
| `~`, `null`, `Null`, empty | `null` | `null` | `"~"`, `"null"`, or `""` |
| `.inf`, `-.Inf`, `.nan` | float infinity / NaN | same | `".inf"` |
| `e5`, `12e3` | string / string (1.1 floats need a dot) | string / `12000` | quote identifiers that look numeric |
| `NO`, `SE`, `BS` (country codes) | `false`, string, string | strings | always quote country codes |
| `key: ` (nothing after) | `null` | `null` | `""` for empty string, `{}` for empty map |

## Error Signatures

The message names the layer: a scanner error is about characters, a parser error is about structure, a constructor error is about types, and a schema error means the YAML was fine.

| Message | Real cause | First move |
|---|---|---|
| `found character '\t' that cannot start any token` | Literal tab in indentation | Show whitespace in the editor; convert tabs to spaces (Rule 3) |
| `mapping values are not allowed in this context` | A second `: ` on a line that was already a scalar, or a plain value containing `: ` | Quote the value (Rule 4) |
| `did not find expected key` / `expected <block end>` | Indentation of one line does not match its siblings, usually a list item one space off | Diff the leading spaces of the block; the first divergent line is the one |
| `could not find expected ':'` | A flow collection was opened (`{` or `[`) and never closed, often a stray `{{ }}` template | Quote the templated value (`dialects.md`) |
| `found undefined alias 'x'` | Alias used before its anchor, or across files | Anchors are file-scoped and forward references are illegal (Rule 8) |
| `duplicated mapping key` | Two identical keys in one mapping | Rule 7 — never resolve by deleting whichever one the tool complained about; check which is live |
| `could not determine a constructor for the tag '!Ref'` | Tool-specific local tag hitting a generic parser | Use the tool's parser or register the tag (`dialects.md`) |
| `cannot unmarshal !!int into string` (Go), `expected str, got int` | Correct YAML, wrong type — coercion, not syntax | Quote the value; do not loosen the schema (Rule 1) |
| `control characters are not allowed` | UTF-16 file, a BOM mid-stream, or a stray `\x00`/`\r` | Check encoding and line endings (`strings.md`) |
| Parse succeeds, tool says the field is missing | The key landed in the wrong parent — indentation, not spelling | Dump the parsed structure and read the actual tree (`editing.md`) |
| Same file, one machine works | Different library version or spec version, not different YAML | Parser matrix (`parsers.md`) |
| Anything else | Bisect: delete the bottom half of the document, re-parse, repeat. Five rounds locate any error in a 1,000-line file | `errors.md` |

## Output Gates

Before delivering any YAML file, patch, or loader snippet:

- Is every value whose type matters quoted — bools-in-disguise, leading zeros, `HH:MM`, versions, ids, dates (Rule 1)?
- Does it parse under the target's spec version, and was it checked with the consumer's own validator (Rule 10)?
- Indentation all spaces, one width, one sequence style; no duplicate keys?
- Do multiline blocks use `|` where newlines are data, with the chomping indicator that produces the intended trailing newline (Rule 5)?
- If this file will be read and rewritten by a tool, does it avoid anchors — or is losing them acceptable (Rule 8)?
- If I edited a file the user wrote, are their comments, key order, and quote style intact (Rule 9)?
- No secret value in the file, and none written into `~/Clawic/data/` — pointer only?
- Did anything durable come out of this — a parser/spec fact, a file that matters, a trap that bit, a working schema or lint config, an agreed cadence? Then it is written to its box in `memory-template.md`, with its `## Boxes` line, in this same turn.

## Configuration

User-dependent variables. Defaults apply until the user states a preference; store them in `~/Clawic/data/yaml/config.yaml`.

| Variable | Type | Default | Effect |
|---|---|---|---|
| spec_version | 1.1 \| 1.2 \| auto | auto | Which column of the Resolution Table applies; `auto` means infer from the consumer's library and say which one was assumed |
| indent_width | number (2-4) | 2 | Indentation of every generated file and of every emitter setting suggested (Rule 3) |
| sequence_indent | 0 \| 2 | 2 | Whether `-` sits at its key's column or two spaces in; matched to the repo's existing style when one is observed |
| quote_style | minimal \| double \| single | minimal | Quoting of generated scalars; `minimal` quotes only what Rule 1 requires, `double`/`single` quote every string |
| max_line_width | number (60-200) | 120 | Emitter `width` setting and the yamllint `line-length` rule; unset emitters wrap at ~80 and churn diffs (Rule 9) |
| yq_flavor | go \| python \| none | go | Which `yq` syntax examples use — mikefarah (Go) and kislyuk (Python jq wrapper) share a name and nothing else (`editing.md`) |
| linter | yamllint \| prettier \| none | yamllint | The lint config generated and the rule names cited in `schemas.md` |
| loader_policy | safe-only \| allow-full | safe-only | Whether a full-tag loader may ever be emitted; `safe-only` also blocks `!!python/` and `!!java` tag examples outside `security.md` |
| anchor_policy | allow \| avoid | allow | Whether generated files use anchors and merge keys, or repeat themselves for round-trip safety (Rule 8) |
| schema_gate | bool | true | Whether every emitted file must pass the consumer's validator before being presented as done (Rule 10) |

Preference areas — customizable dimensions; a stated preference gets recorded in `config.yaml` and applied from then on:

- **Tooling** — editor and its YAML server, lint vs format vs both, which schema store, `yq`/`dasel`/`ruamel`/`jq`-after-conversion for edits — affects every example's shape
- **Conventions** — key order (source vs alphabetical), whether documents open with `---`, comment style and header blocks, empty-value spelling (`null` vs `~` vs omit), list-of-maps vs map-of-maps for named collections — affects generated files and the diff noise they cause
- **Platform** — the primary consumer (Kubernetes, a CI provider, Ansible, an app config loader) and its parser; single-doc vs multi-doc files; file layout per environment — affects which spoke's rules take precedence
- **Safety posture** — how untrusted the inputs are, whether generated loaders are hardened by default, whether secrets are allowed in-file at all under any encryption — affects `security.md` defaults
- **Output register** — full file vs unified diff, whether the reasoning for each quote is shown, how much of the parsed tree to print when explaining — affects every answer's shape
- **Cadence** — schema re-validation after a dependency bump, lint sweep, secret scan over YAML in the repo, spec-version audit — every accepted cadence becomes a row in the `## Due` table of `memory.md`

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Fixing a coercion bug by widening the consumer's schema | Turns one bad value into a permanently loose contract, and the next id still arrives as an int | Quote at the source (Rule 1) |
| Trusting that "it parsed" means it is right | YAML's failure mode is a valid document with wrong types — no error is raised anywhere | Dump the parsed structure and read the types once (`editing.md`) |
| `load()` then `dump()` to edit a file | Comments, order, quote style and anchors all die; the diff is unreviewable | Round-trip loader or in-place patch (Rule 9) |
| Anchors to DRY up Kubernetes or Compose files | Expanded by the first tool that re-emits, and invisible to anyone reading the rendered output | Kustomize overlays, Helm values, Compose `extends` (`kubernetes.md`) |
| `>` for a shell script or a certificate | Folds every newline into a space; the script becomes one line and the PEM becomes invalid | `\|`, with chomping chosen deliberately (Rule 5) |
| Deep indentation to express structure the tool does not have | Silently creates a nested map the schema ignores, and nothing reports it | Validate against the schema, not against the parser (Rule 10) |
| Base64 in a Kubernetes Secret treated as protection | Encoding, not encryption — `base64 -d` is the whole attack | sops, sealed-secrets, or an external secret store (`security.md`) |
| Commenting out a list item to disable it | Leaves `- ` orphans or an empty sequence that parses as `null` and overrides a default with nothing | Delete it, or move it under a disabled key with a comment |
| Copying YAML out of a web page or a chat | Non-breaking spaces and smart quotes survive the paste and produce impossible errors | Retype the indentation, or run it through the linter before reading it |
| One giant multi-document file for a whole environment | One syntax error takes down every document, and tooling that splits on `---` gets it wrong inside block scalars | One concern per file; a directory the tool reads in order (`config-design.md`) |
| Reusing a version-pinned example without checking the library | The same code is safe on PyYAML 6 and an RCE on SnakeYAML 1.x | Parser matrix before copying a loader (`parsers.md`) |
| Letting an emitter wrap long values at its default width | Silent 80-column folding turns a one-line diff into a whole-file diff | Set the emitter width from `max_line_width` (Rule 9) |
| Hand-editing YAML that a controller writes back | The tool rewrites it, drops the comments, and the next human edit conflicts | Edit the source of truth, not the rendered artifact |

## Where Experts Disagree

- **Quote everything vs quote what needs it.** Quote-everything eliminates the entire coercion class and makes review mechanical; minimal quoting keeps the file readable and diffs small. Frontier: files consumed by more than one parser, or authored by people who will not read this table, get quote-everything; a single-consumer file in a linted repo does not (`quote_style`).
- **Anchors and merge keys.** Real DRY-ness for hand-maintained files (CI configs read by one parser), real breakage anywhere the file gets re-emitted or diffed against a rendered version. Nobody argues for anchors in files a controller rewrites (`anchor_policy`).
- **YAML at all for large configs.** The dissent is not aesthetic: significant whitespace plus implicit typing is a hazard surface JSON and TOML do not have, and CUE/Dhall/Jsonnet add types and functions. The counter is ecosystem gravity — Kubernetes, CI and Ansible take YAML, so a generator only moves the problem to render time (`config-design.md`).
- **Generate vs hand-write.** Generated YAML is consistent and reviewable only as a diff of its source; hand-written YAML is reviewable directly and drifts. Frontier is the number of near-identical files: past roughly three environments of the same shape, generation wins.
- **`---` at the top of every file.** Required only for multi-document streams and for a directive; some linters demand it anyway. Pick one and enforce it, because a mixed repo makes every concatenation script wrong.

## Security & Privacy

**Credentials:** this skill reads and writes YAML files the user points it at. It does NOT store, log, copy, or transmit any secret value found in them, and never writes a credential into `~/Clawic/data/yaml/`. Values under key names matching password/token/secret/key/credential, PEM blocks, and Kubernetes Secret `data:`/`stringData:` payloads are replaced by their pointer before anything is saved.

**Local storage:** preferences, memory, the config-file inventory, the toolchain facts and generated artifacts stay in `~/Clawic/data/yaml/` on this machine, plus device rows in `~/Clawic/data/devices/` and project notes in `~/Clawic/data/projects/`. File paths, key names, parser versions and schema URLs only.

**Guardrails:** loaders are emitted in safe mode by default (`loader_policy`); full-tag loaders and `!!python/`-style tags appear only in `security.md`, marked as the vulnerability they are. Destructive edits (in-place rewrite, `yq -i`, deleting a document from a stream) name the file and require explicit confirmation.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/yaml (install if the user confirms):
- `json` — JSON payload design, JSON Schema authoring, jq
- `k8s` — what the manifests mean once they parse: pods, probes, rollouts, RBAC
- `github-actions` — workflow design and CI semantics above the YAML layer
- `ansible` — playbook semantics, variable precedence, idempotence
- `toml` — the other config format, when YAML's implicit typing is the problem

## Feedback

- If useful, star it: https://clawic.com/skills/yaml
- Latest version: https://clawic.com/skills/yaml

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/yaml.
