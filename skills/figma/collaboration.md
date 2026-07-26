# Collaboration — Plans, Permissions, and Review

Half of "Figma cannot do that" is a plan gate, and most of the rest is a permission. Resolve both before architecting anything around a mechanism.

## Plan Gates

Verified 2026-07. Tier names and caps move; check the current pricing and plan pages before committing an architecture to them.

| Capability | Typically requires | If unavailable |
|---|---|---|
| Modes per collection | Lowest tier is effectively single-mode; the mid tier allows a small handful; org and enterprise tiers allow dozens | Split theming axes across separate collections instead of stacking modes in one |
| Branching and merge review | Org tier and above | Duplicate the file as a working copy, name a version before and after, schedule library edits |
| Dev Mode inspect, annotations, Code Connect | A Dev or Full seat per person | Deliver a written spec page plus exported assets |
| Library analytics (insertions, detach rate) | Org tier and above | Reconstruct adoption from a REST-side component-instance audit |
| Variables read/write over the REST API | Enterprise tier | Export tokens through a plugin |
| Shared fonts across the org | Org tier and above | Everyone installs locally; list licensed families on the cover page |
| Link-sharing restrictions, SSO, SCIM, activity logs | Enterprise tier | Manual link hygiene and periodic access review |
| Version history retention | Short retention window on the free tier; extended on paid | Duplicate a snapshot file before any risky restructure |
| Plugin allowlisting and private org plugins | Org tier and above | A written policy, which is weaker than a control |

The consequence that catches teams: a theming matrix of brand × theme × density needs more modes than the mid tier allows, and the discovery usually arrives after the token structure is built. Check `figma_plan` before designing it.

## Structure: Drafts, Projects, Teams

- Drafts are personal. Work left in drafts is invisible to the team, excluded from team-wide search, and lost when the account is deprovisioned. Move anything real into a project on day one.
- Projects group files; teams group projects; the org tier adds cross-team governance. Name projects after the product surface, not after a quarter.
- The file cover — the first frame on the first page — is the thumbnail in recents and search. Files without one get duplicated because nobody can identify them.

## Permissions and Link Hygiene

- Edit vs view is per person or per link. A "can edit" link forwarded outside the team is an edit-access incident, not a view one.
- "Anyone with the link" on a file containing unreleased work is the most common Figma leak. Restrict to the team by default; open the link only for the specific review that needs it, and close it after.
- Prototype links are share links with the same exposure. A prototype sent to a client shares the whole prototype flow, and anyone with the link can pass it on.
- Embeds inherit the file's sharing setting. An embed in a public document with an open file link publishes the file.
- Periodically review guest access. Guests accumulate and rarely get removed.

## Comments and Review

- Comment mode (`C`) pins feedback to coordinates or to a node. Pin to a node when the layout will move; coordinate pins drift into nonsense after a restructure.
- Resolve is the review's state machine. An unresolved thread means an open decision; letting them pile up destroys the signal.
- Comments on a branch stay on the branch. A reviewer commenting on the main file while the work lives on a branch is reviewing an old version — say which surface you want feedback on.
- Async review works when the file carries its own context: a cover note stating what changed, what decision is needed, and by when. Without it, reviewers explore and comment on whatever catches their eye.
- Name a version before and after any review round. Named versions are how "what did it look like before we changed it" gets answered in ten seconds.

## Multiplayer Facilitation

- Spotlight pulls everyone's viewport to yours — the presenting tool for a walkthrough, not for silent work.
- Observation mode follows one person's cursor, which is how you watch a usability session without hovering.
- Cursor chat is for quick coordination in the moment; anything that needs to survive the session goes in a comment.
- Multiple people restructuring the same component set at the same time will produce a mess no merge exists to resolve. Announce structural edits.

## Symptom → Cause

| Symptom | Cause | Fix |
|---|---|---|
| Cannot add another mode | Tier cap on modes per collection | Split axes into separate collections, or upgrade |
| Engineers see measurements but not annotations | No Dev seat | Assign a Dev seat, or ship a written spec |
| Work disappeared when someone left | It lived in their drafts | Move everything real into a project |
| Unreleased design found outside the company | Open link sharing, or a forwarded prototype link | Restrict to team; audit link settings per file |
| Comments do not match the current layout | Coordinate-pinned on a restructured frame | Pin comments to nodes |
| Reviewer feedback refers to old work | Commented on main while work is on a branch | State the review surface; comment on the branch |
| Two designers broke the same component set | Simultaneous structural edits | Branch where available; announce edit windows |
| Cannot recover a file from last month | Short version-history retention on the tier | Duplicate snapshot files before risky work |
