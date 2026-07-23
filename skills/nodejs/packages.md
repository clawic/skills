# Package Traps

- `^1.2.3` floats minor versions, but `^0.2.3` floats only patches — pre-1.0, semver treats minor as breaking. Apps: exact versions + committed lockfile. Libraries: ranges — exact deps in a library force duplicate copies into every consumer's tree.
- `npm install` mutates the lockfile whenever package.json allows it; `npm ci` deletes node_modules, installs the lockfile exactly, and fails on any mismatch — the only correct install for CI and production images.
- Production images: `npm ci --omit=dev` — devDependencies inflate image size and audit surface for code that never runs.
- `peerDependencies` auto-install since npm 7 — and auto-conflict; when two majors of one package coexist, `npm ls <pkg>` shows which dependency paths pulled each.
- `overrides` (npm >=8.3) pins a transitive dependency — the surgical fix for a vulnerability three levels deep. `npm audit fix --force` is the sledgehammer: major bumps, silent breakage.
- Audit triage by path, not count: an advisory in a devDependency that never executes in production is usually noise — check reachability with `npm ls <vuln-pkg>` before scheduling work.
- Know what ships before it ships: `npm pack --dry-run` lists the tarball contents. Prefer the `files` whitelist over `.npmignore` — deny-lists forget files added later.
- `prepublishOnly` runs only before `npm publish`; `prepare` also runs when someone installs your package from git — put the build in `prepare` if git installs should work.
- Docker: node_modules copied from the host carries native modules built for the wrong OS/arch — install inside the image; multi-stage to drop compilers from the final layer.
