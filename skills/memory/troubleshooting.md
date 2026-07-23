# Troubleshooting

## Can't Find Information

**Symptoms:**
- "I don't see that in memory"
- Information exists but not found

**Fixes:**

| Cause | Check | Fix |
|-------|-------|-----|
| Not indexed | `grep -l "topic" ~/Clawic/data/memory/*/INDEX.md` | Add to relevant INDEX |
| Wrong category | Check other categories and `inbox/` | Move to correct place |
| Vocabulary mismatch | Try 2-3 keyword variants (nickname, codename, abbreviation) | Add the missed term to the file's `Keywords:` line so the same search never misses twice |
| In built-in, not here | Check agent's MEMORY.md | Sync if needed |

**Quick search:**
```bash
# Find across all memory
grep -r "keyword" ~/Clawic/data/memory/

# Find in indices only (faster)
grep -r "keyword" ~/Clawic/data/memory/*/INDEX.md
```

---

## Memory Getting Slow

**Symptoms:**
- Takes long to find things
- Category indices are huge

**Fixes:**

1. **Check index sizes:**
```bash
wc -l ~/Clawic/data/memory/*/INDEX.md
# Line count ≈ entry count. Over 100 entries? Split (SKILL.md Rule 6).
```

2. **Split large categories:**
```
Before: projects/INDEX.md (150 entries)
After:  projects/active/INDEX.md (30 entries)
        projects/archived/INDEX.md (120 entries)
```

3. **Archive old content:**
```bash
# Move old items to archive
mv ~/Clawic/data/memory/projects/old.md ~/Clawic/data/memory/archive/
# Update both indices
```

---

## Conflicting with Built-In Memory

**Symptoms:**
- Agent confused about which memory to use
- Duplicate information

**Rule:** This system (`~/Clawic/data/memory/`) is SEPARATE from built-in.

**Fixes:**

1. **Never modify built-in memory** from this skill
2. **Check locations:**
   - Built-in: workspace `MEMORY.md`, workspace `memory/`
   - This skill: `~/Clawic/data/memory/` (home directory)
3. **If duplicates:** Keep detailed version here, summary in built-in

---

## Structure is Messy

**Symptoms:**
- Files everywhere
- Hard to know where things go

**Fixes:**

1. **Establish clear categories:**
```bash
ls ~/Clawic/data/memory/
# Should show clear category folders, not loose files
```

2. **No files in root:**
```
~/Clawic/data/memory/
├── config.md     # OK (system file)
├── INDEX.md      # OK (root index)
├── projects/     # OK (category)
├── random.md     # BAD - put in a category
```

3. **Use inbox for unsorted:**
```bash
mkdir ~/Clawic/data/memory/inbox
# Put unclear items there, sort weekly
```

---

## Sync Not Working

**Symptoms:**
- Sync folder empty or outdated
- Built-in changes not reflected

**Fixes:**

1. **Check sync is enabled:**
```bash
grep sync ~/Clawic/data/memory/config.md
```

2. **Manual sync:**
   - Read from agent's MEMORY.md
   - Extract relevant sections
   - Write to ~/Clawic/data/memory/sync/
   - Update ~/Clawic/data/memory/sync/INDEX.md with date

3. **Remember:** Sync is manual, not automatic. Re-sync periodically.

---

## Recall Returns Outdated Facts

**Symptoms:**
- Agent states an old fact as current ("Alice is at Acme" — she left last year)

**Fixes:**

1. **Date-stamp check:** undated entries can't be judged stale — add `YYYY-MM-DD` to every entry (SKILL.md Rule 4)
2. **Delete, don't archive:** wrong facts get deleted; only old-but-true content is archived — archived wrong facts resurface as truth
3. **Duplicate hunt:** if a fact was updated in one file but recalled from another, two copies exist. Keep one canonical home, convert the rest to links (SKILL.md Rule 5):
```bash
grep -ril "the-fact-keyword" ~/Clawic/data/memory/ | head -5
```

---

## Forgot What Categories Exist

**Quick check:**
```bash
# See all categories
cat ~/Clawic/data/memory/INDEX.md

# See folder structure
ls ~/Clawic/data/memory/
```

---

## INDEX.md Out of Date

**Symptoms:**
- Files exist but not in INDEX
- INDEX lists files that don't exist

**Fix:**

```bash
# Check for unlisted files
for f in ~/Clawic/data/memory/projects/*.md; do
  name=$(basename "$f")
  grep -q "$name" ~/Clawic/data/memory/projects/INDEX.md || echo "Not indexed: $name"
done

# Check for dead links (slugs may contain digits and hyphens)
grep -oE '[a-z0-9-]+\.md' ~/Clawic/data/memory/projects/INDEX.md | while read f; do
  [ ! -f ~/Clawic/data/memory/projects/"$f" ] && echo "Missing: $f"
done
```

**Rebuild INDEX if badly broken:**
```bash
# Generate new index from existing files
ls ~/Clawic/data/memory/projects/*.md | while read f; do
  name=$(basename "$f" .md)
  echo "| $name | ? | $(date +%Y-%m-%d) | $name.md |"
done
```

---

## Not Sure What Goes Where

**Decision tree:**

```
Is it about a specific project?
  → projects/

Is it about a person?
  → people/

Is it a decision with reasoning?
  → decisions/

Is it reference/learning material?
  → knowledge/

Is it a list of things you collect?
  → collections/

None of the above?
  → inbox/ (sort later)
```

---

## Quick Health Check

```bash
#!/bin/bash
echo "=== Memory Health Check ==="

# Check root
[ -d ~/memory ] && echo "✓ ~/memory exists" || echo "✗ ~/memory missing"
[ -f ~/Clawic/data/memory/INDEX.md ] && echo "✓ Root INDEX.md exists" || echo "✗ Root INDEX.md missing"
[ -f ~/Clawic/data/memory/config.md ] && echo "✓ config.md exists" || echo "✗ config.md missing"

# Check categories have indices
echo ""
echo "Category indices:"
for dir in ~/Clawic/data/memory/*/; do
  name=$(basename "$dir")
  [ -f "$dir/INDEX.md" ] && echo "  ✓ $name/INDEX.md" || echo "  ✗ $name/INDEX.md missing"
done

# Count total files
total=$(find ~/memory -name "*.md" | wc -l)
echo ""
echo "Total files: $total"

echo "=== Done ==="
```
