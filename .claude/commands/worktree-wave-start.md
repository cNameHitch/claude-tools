---
description: Launch Terminal.app sessions for a specific wave from the saved worktree wave plan.
handoffs:
  - label: Merge This Wave
    agent: worktree-wave-merge
    prompt: "Merge wave {wave_id}"
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding. The argument is a **wave identifier** (e.g., `1`, `2`, `2b`, `3`). If no argument is provided, **STOP** and ask the user which wave to launch (list available waves from the plan).

## Outline

### 1. Load the wave plan

Read `.claude/memory/worktree-wave-plan.json` from the repo root. If the file does not exist, **STOP** and tell the user to run `/worktree-wave-plan` first to generate a plan.

Parse the JSON and extract:
- `specs_dir` — the specs directory path
- `worktree_base` — the base directory for worktrees
- `base_branch` — the branch to rebase onto (typically `main`)
- `prompt_template` — the prompt template with `{specs_dir}` and `{spec}` placeholders
- `waves` — the array of wave definitions

### 2. Find the requested wave

Match the user's wave ID argument against the `id` field of each wave in the plan. The match should be case-insensitive and flexible (e.g., `"2b"` matches `"2b"` or `"2B"`).

If no matching wave is found, **STOP** and show the user all available wave IDs with their names.

### 3. Display wave details

Show the user what will be launched:
- Wave name and ID
- Number of specs
- For each spec: name, branch, worktree directory, dependencies
- The prompt that will be used (with placeholders filled in)

### 4. Validate worktree directories

For each spec in the wave, verify the worktree directory exists at `{worktree_base}/{spec}`. Check by running `ls` on each directory.

If any worktree is missing:
- List the missing worktrees
- Tell the user to create the worktrees manually using `git worktree add`
- **STOP** — do not proceed with partial launches

### 5. Rebase worktrees onto latest base branch

For each spec in the wave, rebase the worktree's branch onto the latest base branch. Run these commands for each worktree:

```bash
cd {worktree_base}/{spec} && git fetch origin {base_branch} && git rebase origin/{base_branch}
```

If any rebase fails (conflicts), report the failure and **STOP** — do not launch sessions with unresolved conflicts. Tell the user to resolve conflicts manually in the worktree directory.

### 6. Launch Terminal.app sessions

For each spec in the wave, open a new Terminal.app window using AppleScript. The window should:

1. `cd` to the worktree directory
2. Run `claude --dangerously-skip-permissions '<prompt>'` where `<prompt>` is the template with `{specs_dir}` replaced by the plan's `specs_dir` value and `{spec}` replaced by the spec name.

Use this AppleScript pattern for each spec (run via `osascript -e`):

```applescript
tell application "Terminal"
    activate
    do script "cd \"{worktree_dir}\" && claude --dangerously-skip-permissions '{prompt}'"
    set custom title of front window to "{spec}"
end tell
```

**Important escaping rules:**
- The prompt should be wrapped in single quotes in the shell command. Ensure the prompt does not contain single quotes. If it does, escape them as `'\''`.
- The worktree directory path should be wrapped in double quotes in case it contains spaces.
- Backslashes and double quotes in the AppleScript string must be escaped.

Launch all specs in the wave — they run concurrently.

### 7. Report results

After launching all sessions, display:
- How many Terminal windows were opened
- The wave name and specs launched
- A reminder to monitor the Terminal windows for completion
- A reminder that when all sessions in this wave complete, they can merge the wave with `/worktree-wave-merge {wave_id}`
- A reminder that after the merge PR is reviewed and merged to `main`, they can launch the next wave with `/worktree-wave-start {next_wave_id}`

If this is not the last wave, indicate which wave comes next.
