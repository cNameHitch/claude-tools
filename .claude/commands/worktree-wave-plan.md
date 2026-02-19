---
description: Analyze spec dependencies, build a wave execution plan, and save to memory for worktree-wave-start.
handoffs:
  - label: Launch a Wave
    agent: worktree-wave-start
    prompt: "Launch wave 1"
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty). The argument should be a specs directory path (e.g., `specs/phase_2`). If empty, **STOP** and ask the user which specs directory to plan for.

## Outline

### 1. Resolve the specs directory

- If user input is non-empty, use it as the specs directory path (relative to repo root).
- Verify the directory exists using `ls`. If not found, **STOP** and report an error.
- Record the absolute path as `SPECS_DIR`.

### 2. Discover all specs

- List all subdirectories of `SPECS_DIR`. Each subdirectory name is a spec identifier (e.g., `010-keccak-f1600`).
- Sort them by their numeric prefix.
- For each spec, verify that `spec.md` exists in the subdirectory.

### 3. Extract dependencies from each spec

For each spec, read `SPECS_DIR/{spec}/spec.md` and extract the **Dependencies** line from the header metadata. The format is:

```
**Dependencies**: `011-sha3` (description), `012-shake` (description), ...
```

Parse out the backtick-quoted spec identifiers (e.g., `011-sha3`, `012-shake`). These are the **direct dependencies** of this spec.

**Important filtering rules:**
- Only include dependencies that are **within the same specs directory** (i.e., specs that are part of this phase). Dependencies on earlier phases are already satisfied and should be excluded from the dependency graph.
- Build a complete dependency graph mapping each spec to its list of in-phase dependencies.

### 4. Build waves via topological sort

Group specs into concurrent execution waves based on dependency depth:

1. **Wave 1**: All specs with zero in-phase dependencies (they can start immediately).
2. **Wave 2**: All specs whose dependencies are entirely within Wave 1.
3. **Wave 2b, 3, 4, ...**: Continue grouping by dependency depth until all specs are assigned.

For each wave, determine:
- `id`: A string identifier (`"1"`, `"2"`, `"2b"`, `"3"`, etc.)
- `name`: A descriptive name (e.g., `"Wave 1 — Foundations"`)
- `concurrent`: Whether specs in this wave can run in parallel (`true` if >1 spec)
- `specs`: List of spec objects with `spec`, `branch`, and `depends_on` fields

Branch naming convention: `{phase_prefix}/{spec}` (e.g., `phase2/010-keccak-f1600`). Derive the phase prefix from the directory name by removing underscores (e.g., `phase_2` -> `phase2`, `phase_3` -> `phase3`).

**Wave ID assignment guidance:**
- Use numeric IDs for main waves: `"1"`, `"2"`, `"3"`, etc.
- If a wave has a single spec that bridges between two larger waves, consider using a sub-ID like `"2b"`.
- Follow the pattern: 1, 2, 2b, 3, 4, 5 unless the dependency analysis yields different groupings.

### 5. Configure the prompt template

Ask the user what prompt template they want for the Claude sessions. Present the default:

```
/speckit.implement for {specs_dir}/{spec}. This is a continuation of the implementation. It is important that everything is tested. If you have any questions please ask me for clarity, do not make assumptions. Security is the upmost importance. Please research best practices.
```

The template uses `{specs_dir}` and `{spec}` as placeholders that will be filled at launch time.

If the user wants to customize, accept their template. Otherwise use the default.

### 6. Determine worktree base directory

The worktree base directory is the **sibling** of the main repo directory, derived from the repo name:

```
REPO_ROOT = <detected from git rev-parse --show-toplevel>
REPO_NAME = <basename of REPO_ROOT>
WORKTREE_BASE = <parent of REPO_ROOT>/.{REPO_NAME}-worktrees
```

For example, if the repo is at `/Users/dev/projects/myapp`, then:
- `REPO_NAME = myapp`
- `WORKTREE_BASE = /Users/dev/projects/.myapp-worktrees`

### 7. Save the wave plan to memory

Write the complete plan to `.claude/memory/worktree-wave-plan.json` with this schema:

```json
{
  "specs_dir": "specs/phase_2",
  "worktree_base": "/path/to/.reponame-worktrees",
  "base_branch": "main",
  "prompt_template": "/speckit.implement for {specs_dir}/{spec}. ...",
  "waves": [
    {
      "id": "1",
      "name": "Wave 1 — Foundations",
      "concurrent": true,
      "specs": [
        {
          "spec": "010-example-spec",
          "branch": "phase2/010-example-spec",
          "depends_on": []
        }
      ]
    }
  ],
  "created_at": "2026-02-18T12:00:00Z",
  "dependency_graph": {
    "010-example-spec": [],
    "011-dependent-spec": ["010-example-spec"]
  }
}
```

Use the current ISO 8601 timestamp for `created_at`.

### 8. Display the wave plan summary

Print a formatted summary:

1. **Dependency graph table**: Show each spec and its dependencies.
2. **Wave diagram**: An ASCII art diagram showing the wave structure and how specs flow from one wave to the next, with arrows indicating dependencies.
3. **Wave summary table**: For each wave, list the wave ID, name, spec count, and spec names.
4. **Saved file path**: Confirm the plan was saved to `.claude/memory/worktree-wave-plan.json`.

### 9. Offer next steps

Inform the user they can now run `/worktree-wave-start {wave_id}` to launch any wave. The handoff button will also be available.
