# Commands — Memory Store Toolkit

Operator one-liners. Two shell variables carry through every block: `M` is the store, `SPLIT` is the user's `index_split_at` from `config.yaml` (default 100, declared range 25-250). Read the real value before running anything that judges a size — a store configured at 50 or 200 makes every hardcoded 100 a lie.

```bash
M=~/Clawic/data/memory
SPLIT=$(grep -m1 '^index_split_at:' $M/config.yaml 2>/dev/null | awk '{print $2}'); SPLIT=${SPLIT:-100}
```

Substitute the full paths if the shell is not persistent between commands.

Contents: Orientation · Search · Integrity · Health Snapshot · Sync Hygiene · Bulk Operations · Index Regeneration.

## Orientation

```bash
cat $M/INDEX.md                          # what categories exist
ls -d $M/*/                              # the same, from the filesystem — differences are index drift
find $M -name "*.md" | wc -l             # total entries
wc -l $M/*/INDEX.md | sort -n            # index sizes; anything over $SPLIT needs a split (Rule 6)
ls -lt $M/*/*.md | head                  # most recently written — the "what happened last session" view
```

## Search

```bash
grep -i "term" $M/*/INDEX.md             # rung 2: indices only, the routine lookup
grep -ril "term" $M                      # rung 4: full sweep, returns paths
grep -ri "term" $M --include="*.md" -l | head -5
grep -rn "negotiat" $M                   # stem search, catches all inflections
grep -h "^\*\*Keywords:\*\*" $M/people/*.md   # every alias in a category, for vocabulary gaps
```

A body hit with no index hit means the index is stale, not that the search worked.

## Integrity

```bash
# Files with no index row
for f in $M/projects/*.md; do
  n=$(basename "$f"); [ "$n" = "INDEX.md" ] && continue
  grep -q "$n" $M/projects/INDEX.md || echo "Unindexed: $n"
done

# Index rows with no file
grep -oE '[a-z0-9-]+\.md' $M/projects/INDEX.md | sort -u | while read f; do
  [ -f "$M/projects/$f" ] || echo "Dead row: $f"
done

# Orphan links across the whole store
grep -rhoE '→ [a-z0-9/-]+\.md' $M | awk '{print $2}' | sort -u | while read p; do
  [ -f "$M/$p" ] || echo "Broken link: $p"
done
```

## Health Snapshot

```bash
echo "entries:      $(find $M -name '*.md' | wc -l)"
echo "categories:   $(ls -d $M/*/ 2>/dev/null | wc -l)"
echo "inbox:        $(ls $M/inbox 2>/dev/null | wc -l)"
echo "loose in root:$(ls $M/*.md 2>/dev/null | grep -v 'INDEX.md' | wc -l)"
echo "oversized (>$SPLIT):"
wc -l $M/*/INDEX.md 2>/dev/null | awk -v s="$SPLIT" '$1>s && $2!="total" {print "  "$2" ("$1")"}'
echo "stale >1y:";  find $M/people $M/projects -name '*.md' -mtime +365 2>/dev/null | head
```

Expected clean state: loose-in-root 0, inbox emptied at the last cadence, no index over `$SPLIT`.

## Sync Hygiene

```bash
find $M \( -name "*conflicted copy*" -o -name "* 2.md" -o -name "*.sync-conflict*" \) 
git -C $M status --short                 # if the store is versioned
git -C $M log --oneline -- people/alice-smith.md   # when a fact was learned
```

Conflicted index copies resolve as a union of rows, never by picking one: each copy holds a row the other lost.

## Bulk Operations

```bash
# Every mention of a subject, before a deletion request — inbox/ and sync/ included
grep -ril "northwind" $M

# Rename an entity: move, then repair inbound links
mv $M/people/old-slug.md $M/people/new-slug.md
grep -rl "old-slug.md" $M | xargs sed -i '' 's/old-slug\.md/new-slug.md/g'   # GNU sed: drop the ''
```

`sed -i` in bulk is the one command here that can damage the store without comment. Run the `grep -rl` alone first and read the file list.

## Index Regeneration (last resort)

```bash
# Skeleton rows from the files that exist; status and keyword columns must be re-filled by hand
ls $M/projects/*.md | while read f; do
  n=$(basename "$f" .md); [ "$n" = "INDEX" ] && continue
  d=$(grep -m1 -oE '[0-9]{4}-[0-9]{2}-[0-9]{2}' "$f" || date +%F)
  echo "| $n | ? | $d | $n.md |"
done
```

Regeneration recovers the rows and loses the columns that made the index worth reading. Repair row-by-row whenever the damage is small.

## Back To

SKILL.md — Finding Things (the ladder these commands implement), Maintenance (the cadence that runs them), Rules 6 and 9, Configuration (`index_split_at`, read into `$SPLIT` above).
