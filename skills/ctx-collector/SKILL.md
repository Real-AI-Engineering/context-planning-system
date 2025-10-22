---
name: ctx-collector
description: Scan and aggregate Speckit task checkboxes from projects/**/specs/tasks.md into a unified backlog.yaml file for project planning and tracking across multiple repositories.
---

# ctx-collector

## Purpose

Aggregate task checkboxes from Speckit-format task files across multiple project repositories into a unified backlog for planning and tracking. This skill operates in "no-clone mode"—it only reads from existing local directories under `projects/`, making it safe and fast for continuous scanning.

## When to Use This Skill

Use this skill when:
- The user requests to scan tasks across projects (e.g., `/ctx.scan`)
- Building or refreshing the project backlog before planning
- Collecting task status updates from multiple repositories
- Needing a unified view of all open and completed tasks

Do not use this skill when:
- Working with a single project's tasks directly
- The user wants to modify tasks (this skill only reads)
- Projects are not structured under `context/projects/`

## How It Works

### Task Collection Process

1. **Discover task files** recursively in `projects/` directory:
   - Primary: `tasks.md` files (at any depth)
   - Optional: `checklists/**/*.md` files (if configured)
   - Works with any project structure you prefer

2. **Parse checkboxes** from discovered files:
   - Open tasks: `- [ ] Task description`
   - Completed tasks: `- [X] Task description`

3. **Extract metadata** from task lines and headings:
   - Task ID: Pattern `T\d+` (e.g., T101, T250)
   - Priority: Pattern `P1|P2|P3` (from line or nearest heading)
   - Scope: Last two heading levels as context
   - File location and line number for traceability

4. **Generate unified backlog** at `state/backlog.yaml` with structure:
   ```yaml
   generated_at: "ISO-8601 timestamp"
   items:
     - uid: "my-project#T101"
       project: "my-project"
       file: "projects/my-project/specs/tasks.md"
       line: 42
       id: "T101"
       title: "Task description"
       priority: "P1"
       status: "open"
       scope:
         section: "Feature Name"
         subsection: "Phase 1"
   ```

**Note:** Project names are derived from the directory structure automatically.

### Using the Bundled Script

The skill includes `scripts/scan_tasks.py` for deterministic, fast scanning:

```bash
# Run from context directory
python3 skills/ctx-collector/scripts/scan_tasks.py
```

The script automatically:
- Scans all projects in `projects/` directory
- Handles missing YAML library (falls back to JSON)
- Generates unique IDs for tasks without explicit IDs
- Creates `state/` directory if needed
- Overwrites `state/backlog.yaml` with fresh scan

### Task ID and UID Generation

- **Explicit IDs**: Tasks with `T\d+` pattern use that as ID
- **Implicit UIDs**: Tasks without IDs get hash-based UID from `project#path:line:title`
- **Project prefix**: All UIDs include `project#` for cross-project uniqueness

### Priority Detection

Priority is determined in order:
1. Priority marker in task line (e.g., `- [ ] T101 P1 Fix critical bug`)
2. Priority marker in nearest parent heading
3. Default to `P2` if not specified

## Workflow Example

When user requests `/ctx.scan`:

1. Verify `projects/` contains your project directories
2. Execute `scripts/scan_tasks.py` or implement inline scanning
3. Report summary: "Scanned X files, found Y tasks (Z open, W done)"
4. Confirm `state/backlog.yaml` has been updated

## Constraints

- **Read-only operation**: Never modifies source task files
- **Local files only**: No network access or git operations
- **No cloning**: Only processes existing local directories
- **Stable output format**: Other skills depend on backlog.yaml structure

## Related Skills

- `ctx-planning`: Consumes `state/backlog.yaml` to generate daily/weekly plans
- Use `/ctx.update` to modify planning parameters in `BaseContext.yaml`

## Troubleshooting

If scanning produces unexpected results, check:
- Task files use proper checkbox syntax: `- [ ]` or `- [X]`
- Task IDs follow `T\d+` pattern (e.g., T1, T100, not task-1)
- Priority markers use `P1`, `P2`, or `P3` format
- Files are UTF-8 encoded
- `state/` directory is writable
