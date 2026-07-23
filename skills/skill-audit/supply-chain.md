# Supply Chain — Publishers, Names, Dependencies, Registries

The install moment is the hijack moment: every name a skill asks you (or the agent) to install gets verified before it runs — after is too late.

## Package-Name Verification (any install a skill requests)

1. Exact string against the OFFICIAL registry (npmjs.com, pypi.org, crates.io, brew formulae, the skill registry itself) — never a search-engine result, never a link inside the skill under audit.
2. Char-by-char: homoglyphs pass reading and fail comparison (`hidden-content.md`). Copy-paste the name into the registry search; retyping "corrects" what your eyes missed.
3. Lookalike families: hyphenation (`python-dateutil` vs `python-dateutils`), doubled letters, singular/plural (`request` vs `requests` is the canonical PyPI typosquat), prefix squats (`node-` / `py-` + famous name).
4. Age + adoption: a package bearing a famous name, registered weeks ago, with negligible downloads = typosquat until proven otherwise.
5. Dependency confusion: an internal-sounding name (`acme-corp-utils`) requested from a public registry — the attack class Birsan demonstrated against major vendors. Anything that smells internal gets source verification, not installation.

## Skill-Registry Specifics

- Scoped slugs: verify the OWNER, not the slug — slugs are not unique across owners, and `@someone/docker` says nothing about `@ivangdavila/docker`.
- The registry card shows the entry file; the audit covers the folder as installed on disk (Rule 6). Audit what landed, not what the card showed.
- Homepage/repo mismatch: a skill claiming a repo its publisher doesn't own, or a homepage on an unrelated domain = flag.
- Stars and downloads are reputation, and reputation is not evidence (Traps): accounts get compromised and sold. The artifact is the unit of trust.

## Dependencies a Skill Declares

- Required bins in metadata: each one is an install decision — verify names exactly like packages above.
- Pins: an unpinned install command inherits whatever the registry serves at run time; the audited artifact must pin what it asks for.
- Checksums verify only when they arrive by a DIFFERENT channel than the download (registry page vs CDN file). A checksum served next to the file proves transport integrity, not authorship.

## In-Skill Updaters

"Check <url> for the newest version of this skill" bypasses both the registry's scanning and your diff-audit — the two controls `update-audit.md` exists to apply. Flag; reject if the fetched content is instructions or code.

## When Verification Fails

Name absent from the official registry, owner mismatch, or homoglyph found → REJECT and report to the registry (`incident.md` has the reporting outline). Never "install the probably-right one" on the user's behalf: name resolution is the user's call once shown the evidence.
