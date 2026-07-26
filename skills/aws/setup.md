# Setup — AWS

Read this on first use to load stored preferences. Do not interview the user.

## Your Attitude

AWS has hundreds of services; this user needs three of them and a bill that surprises nobody. Cut through the catalog, name the monthly number, and say what the blast radius is. Reach for the cheapest thing that meets the requirement, and say when a cheaper thing would not.

## How To Load Preferences

1. Read `~/Clawic/data/aws/config.yaml` if it exists and apply its values.
2. For anything absent, use the defaults in the Configuration table of `SKILL.md` — do not ask. Defaults: `default_region: none`, `cli_profile: none`, `iac_tool: terraform`, `monthly_budget_usd: 100`, `account_model: single`, `compliance_regime: none`.
3. Fall back to `~/Clawic/profile.yaml` for shared universals (currency, locale) when the skill's own config is silent. Precedence: `config.yaml` → `profile.yaml` → table default.
4. Read `~/Clawic/data/aws/memory.md` for prior context, plus `~/Clawic/data/servers/servers.md` (shared host inventory, any provider) and `resources.md` / `spend-log.md` if they exist. Absence is fine; proceed without commenting on it.

Work from defaults immediately. Never open with questions about their account, their budget, or how proactive to be.

The one exception to silence is `default_region`: while it is unset, state which region you are assuming before running or recommending anything region-scoped (SKILL.md Rule 7). That is a statement, not a question.

## Recording Preferences (only when the user declares one)

Write to config or memory **only** when a preference surfaces in the course of the work — never as a preflight questionnaire.

- User names a region, profile, IaC tool, budget, account model, or compliance regime → update the matching key in `~/Clawic/data/aws/config.yaml`.
- User expresses a stance or habit — tagging conventions, standardized instance families, how aggressively to surface cost warnings, whether destructive commands may be shown at all — record it under the relevant preference area (tooling, conventions, platform, safety posture, cost reporting, service preferences) in `~/Clawic/data/aws/memory.md`.
- User corrects earlier guidance → update the stored value so the correction is not needed twice.

If the user has said nothing, store nothing. A declared preference in `config.yaml` outranks anything observed in `memory.md`; never overwrite the former with the latter without asking.

## What The Working Files Hold

- `memory.md` — account context, experience level, pain points, how they like answers shaped. Only what they actually revealed.
- `resources.md` — the infrastructure inventory built up by Rule 1 discovery, so the next session does not rediscover the account.
- `spend-log.md` — monthly totals, alerts configured, and optimization wins, so a bill delta can be compared against something.

Update `resources.md` whenever an inventory pass runs, and `spend-log.md` whenever a cost review happens. Both are cheap to maintain and expensive to reconstruct.
