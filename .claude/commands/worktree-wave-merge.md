---
description: Merge a wave's spec branches into an aggregation branch, resolve conflicts, run quality gates, push, and create a PR.
handoffs:
  - label: Launch Next Wave
    agent: worktree-wave-start
    prompt: "Launch wave {next_wave_id}"
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding. The argument is a **wave identifier** (e.g., `1`, `2`, `2b`, `3`). If no argument is provided, **STOP** and ask the user which wave to merge (list available waves from the plan).

## Outline

### 1. Load the wave plan

Read `.claude/memory/worktree-wave-plan.json` from the repo root. If the file does not exist, **STOP** and tell the user to run `/worktree-wave-plan` first to generate a plan.

Parse the JSON and extract:
- `base_branch` — the branch to base the aggregation branch on (typically `main`)
- `waves` — the array of wave definitions
- `dependency_graph` — the dependency graph for ordering

### 2. Find the requested wave

Match the user's wave ID argument against the `id` field of each wave in the plan. The match should be case-insensitive and flexible (e.g., `"2b"` matches `"2b"` or `"2B"`).

If no matching wave is found, **STOP** and show the user all available wave IDs with their names.

### 3. Determine merge order

Sort the wave's specs in dependency order — specs with **fewer in-wave dependencies first**. This ensures that when merging spec B that depends on spec A, spec A's changes are already in the aggregation branch.

To determine order:
1. For each spec in the wave, count how many of its `depends_on` entries are also specs **within this same wave**.
2. Sort ascending by this count (fewest in-wave dependencies first).
3. If tied, sort by spec name (lexicographic by the numeric prefix).

Display the planned merge order to the user before proceeding.

### 4. Create or checkout the aggregation branch

The aggregation branch is named `wave-{id}` (e.g., `wave-2`, `wave-2b`).

```bash
git fetch origin
```

Check if the branch already exists locally or on the remote:
- If it exists locally, check it out: `git checkout wave-{id}`
- If it exists only on the remote, check it out tracking the remote: `git checkout -b wave-{id} origin/wave-{id}`
- If it does not exist at all, create it from the base branch: `git checkout -b wave-{id} origin/{base_branch}`

### 5. Merge each spec branch sequentially

For each spec branch in the determined merge order:

#### 5a. Fetch the latest spec branch

```bash
git fetch origin {branch}
```

#### 5b. Attempt the merge

```bash
git merge --no-ff origin/{branch}
```

#### 5c. Handle merge success (no conflicts)

If the merge succeeds cleanly, the default merge commit message is fine. Move on to the next spec.

#### 5d. Handle merge conflicts

If the merge reports conflicts:

1. **Identify conflicting files**: Run `git diff --name-only --diff-filter=U` to list files with unresolved conflicts.
2. **For each conflicting file**:
   - Read the file contents (which will contain `<<<<<<<`, `=======`, `>>>>>>>` conflict markers).
   - Analyze both sides of each conflict:
     - The **ours** side (aggregation branch) contains changes from previously merged specs.
     - The **theirs** side (incoming spec branch) contains the new spec's changes.
   - **Resolution strategy**:
     - For `mod.rs` files (module declarations): Almost always keep **both** sides — both the existing `pub mod` declarations and the new ones. Combine them, maintaining sorted order.
     - For shared type files (e.g., `params.rs`, `poly.rs`): Merge both additions. If both sides add different items, keep all of them. If both sides modify the same item differently, prefer the version from the spec that owns that item (based on the spec name).
     - For test files: Keep all tests from both sides.
     - For `lib.rs` / `main.rs` re-exports: Combine all re-exports from both sides.
     - General principle: **Additive merge** — both branches are adding new code for different specs, so typically both sides should be kept.
   - Write the resolved file using the Edit tool (replace the entire conflicted region).
   - Stage the resolved file: `git add {file}`
3. **Complete the merge** after all conflicts are resolved:
   ```bash
   git commit -m "merge: resolve conflicts merging {spec} into wave-{id}"
   ```
4. **Report** what conflicts were found and how they were resolved.

### 6. Run quality gates

After all spec branches are merged, run the quality gates in order. The specific commands depend on the project's build system (check CLAUDE.md or project configuration for the correct commands).

Common quality gate pattern:

#### 6a. Format check

Run the project's formatter check. If issues are found, run the formatter to fix them, then stage and commit:
```bash
git add -A
git commit -m "style: apply formatting after wave-{id} merge"
```

#### 6b. Lint check

Run the project's linter. If errors are found, **investigate and fix them**:
- Read the error messages carefully
- The most common issues after merges are: duplicate imports, unused imports, type mismatches from conflicting type definitions, missing trait implementations
- Fix each issue in the appropriate source file
- Stage and commit fixes:
  ```bash
  git add -A
  git commit -m "fix: resolve lint warnings after wave-{id} merge"
  ```

#### 6c. Test suite

Run the project's test suite. If tests fail, **investigate and fix**:
- Read the test failure output
- Common issues: test functions referencing moved/renamed items, conflicting test helper definitions, import path changes
- Fix each issue
- Stage and commit:
  ```bash
  git add -A
  git commit -m "fix: resolve test failures after wave-{id} merge"
  ```

### 7. Iterative fix loop

If any quality gate failed in step 6, repeat the full quality gate sequence after applying fixes. Continue this loop up to **3 attempts**.

If after 3 attempts the quality gates still fail, **STOP** and report:
- Which gate is failing
- The exact error messages
- What fixes were attempted
- Ask the user for guidance

### 8. Push the aggregation branch

Once all quality gates pass:

```bash
git push -u origin wave-{id}
```

If the push fails because the remote branch has diverged, report this to the user and **STOP** — do not force push. Ask the user how to proceed.

### 9. Create a pull request

Create a PR against the base branch using `gh`:

```bash
gh pr create --base {base_branch} --head wave-{id} --title "Wave {id} — {wave_name}" --body "$(cat <<'EOF'
## Summary

Automated merge of Wave {id} ({wave_name}) spec branches into `{base_branch}`.

### Specs merged (in order):
{for each spec: - `{spec}` from branch `{branch}`}

### Merge details:
- Total specs merged: {count}
- Conflicts resolved: {conflict_count} (automated)
- Quality gates: format check, lint check, test suite

## Test plan
- [ ] All tests pass
- [ ] Lint check clean
- [ ] Format check clean
- [ ] Review merge conflict resolutions for correctness

Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

Replace all `{placeholders}` with actual values. The spec list should be a markdown bullet list.

If a PR already exists for this branch, report the existing PR URL instead of creating a duplicate.

### 10. Report results

Display a summary:
- Aggregation branch name
- Number of specs merged
- Number of conflicts resolved
- Quality gate results (pass/fail for each)
- PR URL
- Next steps: If this is not the last wave, suggest launching the next wave with `/worktree-wave-start {next_wave_id}` or merging the next wave with `/worktree-wave-merge {next_wave_id}` after the PR is reviewed and merged.
