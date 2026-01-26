---
name: git-integration
description: Set up and use Git integration for semantic diff and merge of event model files. Use when working with version control, resolving merge conflicts, or comparing model versions.
---

# Git Integration for Event Models

Event Modeler provides semantic Git integration that understands the structure of `.eventmodel` files. Instead of text-based diffs and merges, it compares and merges based on the actual model semantics.

## Why Semantic Git Integration?

Event model files (`.eventmodel`) are stored as newline-delimited JSON events. Text-based Git tools would:
- Show noisy diffs for insignificant changes (positions, sizes)
- Create false conflicts when changes don't actually overlap
- Fail to merge compatible changes intelligently

The semantic integration:
- **Ignores layout properties** - Position/size changes don't create conflicts
- **Understands entity relationships** - Knows when changes are compatible
- **Auto-resolves safe conflicts** - Identical changes, status upgrades, array unions
- **Provides meaningful diffs** - Shows what actually changed semantically

## Commands Overview

```
GIT INTEGRATION:
  git setup                      Configure git to use semantic diff/merge for .eventmodel files
  git setup --global             Configure globally (for all repos)
  git status                     Show git integration status

DIFF:
  diff <file1> <file2>           Compare two event model files semantically
  diff <file> [--ref <gitref>]   Compare working copy against git ref (default: HEAD)
  diff ... --format text         Human-readable output (default)
  diff ... --format xml|json     Structured output

MERGE:
  merge --base <file> --ours <file> --theirs <file> --output <file>
                                 Three-way merge of event model files
  merge ... --strategy ours|theirs
                                 Auto-resolve conflicts with given strategy
  merge ... --dry-run            Show conflicts without writing output
```

## Setting Up Git Integration

### Initial Setup

Run this once per repository (or globally with `--global`):

```bash
# In a git repository
eventmodeler git setup

# Or globally for all repositories
eventmodeler git setup --global
```

This configures:
1. **Merge driver** - Git uses `eventmodeler merge` for `.eventmodel` files
2. **Diff driver** - Git uses `eventmodeler diff` for `.eventmodel` files
3. **`.gitattributes`** - Associates `.eventmodel` files with these drivers

### Verify Setup

```bash
eventmodeler git status
```

Example output:
```
Git Integration Status
======================
In git repository: Yes
Merge driver configured: Yes
  Driver: eventmodeler merge --base %O --ours %A --theirs %B --output %A
Diff driver configured: Yes
  Command: eventmodeler diff
.gitattributes configured: Yes
```

## Comparing Event Models (Diff)

### Compare Against Git History

```bash
# Compare working copy against HEAD (most common)
eventmodeler diff model.eventmodel

# Compare against a specific commit/branch
eventmodeler diff model.eventmodel --ref HEAD~3
eventmodeler diff model.eventmodel --ref origin/main
eventmodeler diff model.eventmodel --ref feature-branch
```

### Compare Two Files

```bash
# Compare two files directly
eventmodeler diff old-model.eventmodel new-model.eventmodel

# With named arguments
eventmodeler diff --base old-model.eventmodel --compare new-model.eventmodel
```

### Output Formats

**Text format** (default) - human-readable:
```bash
eventmodeler diff model.eventmodel --format text
```

Example output:
```
5 changes: 3 added, 1 removed, 1 modified

Added:
  + Command "PlaceOrder"
  + Event "OrderPlaced"
  + Flow PlaceOrder → OrderPlaced

Removed:
  - Screen "Legacy Checkout"

Modified:
  ~ ReadModel "OrderSummary"
    + field "totalAmount" (Decimal)
    ~ field "status": renamed from "orderStatus"
```

**JSON format** - for scripts/CI:
```bash
eventmodeler diff model.eventmodel --format json
```

**XML format** - structured output:
```bash
eventmodeler diff model.eventmodel --format xml
```

## Merging Event Models

### How Three-Way Merge Works

When Git encounters a merge conflict on an `.eventmodel` file, it invokes:

```
eventmodeler merge --base <ancestor> --ours <our-branch> --theirs <their-branch> --output <result>
```

The merge algorithm:
1. Computes semantic diffs from the common ancestor to each branch
2. Identifies which changes can be merged automatically
3. Detects true conflicts that require human intervention
4. Merges event streams based on timestamps

### Manual Merge

You can also invoke merge directly:

```bash
# Perform a three-way merge
eventmodeler merge \
  --base ancestor.eventmodel \
  --ours my-changes.eventmodel \
  --theirs their-changes.eventmodel \
  --output merged.eventmodel
```

### Dry Run (Preview)

See what would happen without writing output:

```bash
eventmodeler merge \
  --base base.eventmodel \
  --ours ours.eventmodel \
  --theirs theirs.eventmodel \
  --dry-run
```

### Auto-Resolve Conflicts

When you want to automatically resolve conflicts:

```bash
# Accept our changes on conflicts
eventmodeler merge ... --strategy ours

# Accept their changes on conflicts
eventmodeler merge ... --strategy theirs
```

## What Gets Auto-Resolved

The merge driver automatically resolves these cases:

| Situation | Resolution |
|-----------|------------|
| Both changed layout (position, size) | Accept theirs (last-write-wins) |
| Both changed slice status | Accept higher priority (done > in-progress > blocked > created) |
| Both added same item to array | Union of both additions |
| Both removed same item from array | Item is removed |
| Both made identical changes | Accept the change |
| Both added identical fields | Accept the field |

## What Creates Conflicts

These situations require human intervention:

- **Name conflicts** - Both renamed an entity to different names
- **Type conflicts** - Both changed a field to different types
- **Delete vs modify** - One branch deleted, the other modified
- **Scenario changes** - Conflicting changes to given/when/then
- **Field mapping conflicts** - Different mappings on the same flow
- **Both added with same ID** - Duplicate entity creation

## Merge Output

### Success (Exit Code 0)

```xml
<merge-result status="success">
  <summary merged-events="127" conflicts="0" auto-resolved="3"/>
  <auto-resolved>
    <resolution entity="slice" name="Place Order" action="both-modified-status-identically"/>
    <resolution entity="command" name="PlaceOrder" action="layout-change"/>
    <resolution entity="readModel" name="OrderSummary" action="both-added-field-identically"/>
  </auto-resolved>
</merge-result>
```

### Conflicts (Exit Code 1)

```xml
<merge-result status="conflicts">
  <summary merged-events="124" conflicts="2" auto-resolved="1"/>
  <conflicts>
    <conflict entity="event" name="OrderPlaced" id="abc-123">
      <reason>Both modified field "status" to different types</reason>
      <ours type="String"/>
      <theirs type="Enum"/>
    </conflict>
    <conflict entity="readModel" name="OrderSummary" id="def-456">
      <reason>Ours deleted, theirs modified</reason>
    </conflict>
  </conflicts>
</merge-result>
```

## Workflow Examples

### Standard Git Workflow

After setup, Git operations work normally:

```bash
# Make changes to your event model
# ... edit in Event Modeler UI ...

# Commit changes
git add model.eventmodel
git commit -m "Add order placement slice"

# Pull with potential merge
git pull origin main
# → Semantic merge happens automatically if there are conflicts

# Push your changes
git push
```

### Reviewing Changes Before Commit

```bash
# See what you changed semantically
eventmodeler diff model.eventmodel

# If the diff looks good, commit
git add model.eventmodel
git commit -m "Refactor payment processing slice"
```

### Comparing Branches

```bash
# Compare your branch against main
eventmodeler diff model.eventmodel --ref origin/main
```

### Resolving a Merge Conflict

If Git reports a conflict:

```bash
# See what conflicted
eventmodeler merge \
  --base model.eventmodel.BASE \
  --ours model.eventmodel.LOCAL \
  --theirs model.eventmodel.REMOTE \
  --dry-run

# Option 1: Accept theirs for all conflicts
eventmodeler merge \
  --base model.eventmodel.BASE \
  --ours model.eventmodel.LOCAL \
  --theirs model.eventmodel.REMOTE \
  --output model.eventmodel \
  --strategy theirs

# Option 2: Manually resolve in Event Modeler UI
# Open the conflicted file, make decisions, save

# Mark as resolved
git add model.eventmodel
git commit
```

### CI/CD Integration

```bash
# Get JSON output for parsing in CI
eventmodeler diff model.eventmodel --ref origin/main --format json

# Check for conflicts before merge
eventmodeler merge \
  --base base.eventmodel \
  --ours ours.eventmodel \
  --theirs theirs.eventmodel \
  --dry-run \
  --format json
# Exit code 1 if conflicts exist
```

## Best Practices

1. **Set up integration early** - Run `eventmodeler git setup` when starting a project
2. **Commit frequently** - Smaller changes = easier merges
3. **Review diffs before commit** - Use `eventmodeler diff` to see semantic changes
4. **Use dry-run for complex merges** - Preview before committing to a strategy
5. **Coordinate on structural changes** - Major refactors are harder to merge
6. **Use branches** - Feature branches isolate changes until ready

## Troubleshooting

### "Not in a git repository"

Run the command from within a git repository, or use `--global` for system-wide setup.

### Merge driver not triggering

Check that `.gitattributes` contains:
```
*.eventmodel merge=eventmodel
*.eventmodel diff=eventmodel
```

Verify with `eventmodeler git status`.

### Conflicts on every merge

If you're getting conflicts on layout changes, ensure Git is using the semantic merge driver. Text-based merges will conflict on position changes that the semantic driver auto-resolves.

### Diff shows no changes but file is modified

The semantic diff ignores layout properties. If only positions changed, the semantic diff is empty (which is correct - no semantic changes occurred).
