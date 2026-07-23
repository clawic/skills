# Auth Playbook — Accounts, Scopes, and Service Accounts

## Recommended Auth Paths

- Local desktop: `gws auth setup` then `gws auth login` (`gcloud` recommended for setup)
- Existing project control: manual OAuth client + `gws auth login`
- CI/headless: inject a dedicated credentials file from a secure secret manager at runtime — never bake tokens into images or repos; preflight checks for unattended runs in `automation.md`
- Server-to-server: service account credentials file with domain-wide delegation and impersonation

## OAuth Client Lifecycle (the failure nobody diagnoses)

- An OAuth client in **Testing** publishing status issues refresh tokens that expire after 7 days, with 100 test users max (documented Google OAuth policy).
- Symptom signature: logins that die weekly with `invalid_grant`. The fix is not "log in again" — move the client to Production status (the unverified-app warning is cosmetic for your own internal use) or accept weekly re-auth deliberately.
- Restricted scopes (full Gmail, full Drive) on a **published** client trigger Google's verification/security-assessment pipeline. For personal/internal tooling, staying unverified is normal; for distributed apps, budget for review before choosing restricted scopes.

## Multi-Account Operations

```bash
gws auth login --account work@corp.com
gws auth login --account personal@gmail.com
gws auth list
gws auth default work@corp.com
```

One-off override pattern:

```bash
gws --account personal@gmail.com drive files list --params '{"pageSize":5}'
```

## Precedence Rules

1. explicit access-token override
2. explicit credentials-file override
3. encrypted account credentials (`gws auth login --account`)

## Scope Strategy

Scopes are tiered; pick the narrowest tier that can actually complete the task:

| Need | Scope | Note |
|------|-------|------|
| Read/write only files this tool creates or opens | `drive.file` | Sees nothing else in the Drive — the "why is my Drive empty" scope |
| Read entire Drive | `drive.readonly` | Sensitive tier |
| Full Drive read/write | `drive` | Restricted tier — verification burden if published |
| Send mail only | `gmail.send` | Cannot read the mailbox |
| Read/label/trash mail | `gmail.modify` | Cannot permanently delete |
| Permanent delete (`messages.delete`, `batchDelete`) | `https://mail.google.com/` | Full scope is the documented requirement here, not a smell — `gmail.modify` cannot do it |
| Read-only audit work (Directory, Reports) | `.readonly` admin scope variants | Token physically cannot mutate — the right shape for investigations (`admin.md`) |

- Expand with explicit `--scopes` only when a method requires it; record the expansion in `memory.md` scope profiles.
- Contrarian but defensible: when the workflow genuinely needs permanent deletion, requesting full Gmail scope up front beats a mid-task re-consent loop — narrow-by-default is a heuristic, not dogma.

## Service Accounts

- A bare service account only sees its own (empty) Drive and has no mailbox. The classic confusion — `files.list` returns nothing — is not an error; it is the wrong identity.
- To act as a user: enable domain-wide delegation, have a super admin authorize the client ID with the exact scope list in Admin Console, then impersonate via subject/impersonation settings. Scope list mismatch between code and Admin Console grant = `unauthorized_client`.

## Non-Negotiables

- Never request secrets in chat text.
- Never mix unrelated tenants under one default account.
- Never run mutation commands when account ownership is unclear.
- Never store unencrypted credentials in shared workspaces.
