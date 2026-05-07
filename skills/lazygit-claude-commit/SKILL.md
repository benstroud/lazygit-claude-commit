---
name: lazygit-claude-commit
description: Set up a lazygit custom command that generates Git commit messages by piping the staged diff to the Claude CLI. Use this skill whenever the user wants AI-generated commit messages in lazygit, wants to wire up Claude (or `claude -p`) into their git workflow, asks about lazygit `customCommands`, mentions generating commit messages from diffs, or wants a key binding in lazygit that drafts commit messages for them — even if they don't name this skill explicitly.
---

# lazygit-claude-commit

Adds a `customCommands` entry to lazygit's user config (path resolved via `lazygit --print-config-dir` — `~/Library/Application Support/lazygit/config.yml` on macOS, `~/.config/lazygit/config.yml` on Linux) that, when triggered from the Files panel, pipes the staged diff to `claude -p --bare` and opens the resulting message in `$EDITOR` for review before the commit lands.

## Why this design

A few choices are load-bearing — preserve them unless the user asks otherwise:

- **`git commit --edit -F <tmp>`** instead of auto-commit. The model drafts; the human approves. This keeps you in the loop, lets you fix bad output without rerunning the model, and means pre-commit hooks still run normally.
- **`claude -p --bare`** rather than plain `claude -p`. `--bare` skips hooks, CLAUDE.md auto-discovery, auto-memory, plugin sync, and keychain reads — all of which are wasted overhead for a one-shot diff-to-message generation, and any of which can introduce surprising context. Fastest startup, most predictable output.
- **`subprocess: true`** in the lazygit entry. Required so the running command can hand the terminal to `$EDITOR` for the commit message review.
- **Empty-staged-diff guard**. Without it, the command happily calls Claude on an empty diff and then opens an empty commit editor — confusing failure mode.

## Workflow

### Step 1: Verify prerequisites

Check that both tools are installed before changing anything:

```bash
command -v lazygit && command -v claude
```

If either is missing, stop and tell the user what to install (`brew install lazygit` and the Claude CLI from Anthropic's docs). Don't proceed with a partially working setup.

### Step 2: Find the config path

**Don't assume the path.** lazygit's config location is platform-dependent and respects `XDG_CONFIG_HOME`. On macOS the default is `~/Library/Application Support/lazygit/config.yml`, on Linux it's typically `~/.config/lazygit/config.yml`, and either platform can be overridden by env vars. Hardcoding the wrong path silently produces a config file lazygit will never read — the most common failure mode for this skill.

Always discover the path at runtime:

```bash
lazygit --print-config-dir
```

The config file is `<that-dir>/config.yml`. Use that path for every read and write below. If the directory doesn't exist yet, `mkdir -p` it.

> Common gotcha on macOS: a stale `~/.config/lazygit/config.yml` from earlier dotfile setups or cross-platform copying. lazygit will still ignore it. If the user reports "I edited the file but nothing changed," check whether they edited the XDG path while lazygit reads the Application Support path. Either move the file or `ln -s` it, but resolve the divergence — two config files drift.

### Step 3: Inspect the existing config

Read the discovered config file before writing — there are three cases to handle differently:

1. **File missing or empty** → write the full config block fresh.
2. **File exists, no `customCommands` key** → append a top-level `customCommands:` list with this entry.
3. **File exists, already has `customCommands`** → append this entry to the existing list. Do not duplicate: if there's already an entry whose `command` references `claude` (any flag combination), ask the user whether to replace it or skip.

Also check the chosen key binding (`<c-a>` by default) against any existing `customCommands` keys in the same `context`. If there's a collision, propose an alternative (`<c-g>` for "Generate", `M` for "Message") and let the user choose before writing.

### Step 3: Write the entry

The canonical block to install (preserve formatting; the multi-line shell script inside the YAML scalar is intentional):

```yaml
customCommands:
  - key: '<c-a>'
    context: 'files'
    description: 'Generate commit message via Claude'
    loadingText: 'Asking Claude…'
    subprocess: true
    command: >
      bash -c '
        set -e;
        diff=$(git diff --cached);
        if [ -z "$diff" ]; then
          echo "No staged changes — stage files first."; exit 1;
        fi;
        tmp=$(mktemp);
        trap "rm -f $tmp" EXIT;
        printf "%s" "$diff" | claude -p --bare "Write a git commit message for the staged diff on stdin. Format: a single conventional-commit subject line (type: summary, under 72 chars, imperative mood), then a blank line and a short body only if the change is non-trivial. Output only the message — no preamble, no code fences, no trailing commentary." > "$tmp";
        git commit --edit -F "$tmp"
      '
```

Use the `Edit` tool when the file already has unrelated content — preserve every byte you didn't deliberately change. Use `Write` only when the file is empty or absent.

### Step 4: Verify

Tell the user how to test:

1. Open lazygit in any repo.
2. Stage a file (`s` on the file in the Files panel).
3. Press `Ctrl+A`. Expect to see `Asking Claude…`, then `$EDITOR` opens with a generated message.
4. Save to commit, or quit with an empty buffer to abort.

Edge cases to mention:
- With nothing staged, the command prints `No staged changes — stage files first.` and does not open an editor.
- Pre-commit hooks still run on the resulting `git commit`. If a hook fails, the commit is rejected just like any manual commit.
- `claude -p` runs in the current working directory (the repo). It does not load CLAUDE.md (because of `--bare`), so the model only sees the diff itself.

## Customisation the user might ask for

- **Different key binding**: change `key:`. Any value lazygit accepts works (`<c-g>`, `M`, `<f5>`, etc.).
- **Different prompt style** (e.g., your team uses a specific commit-message convention): edit the long prompt string inside the `claude -p --bare "..."` call. Keep "Output only the message" — without it, the model is liable to add a friendly preamble that ends up in your commit.
- **Auto-commit without editor review**: replace `git commit --edit -F "$tmp"` with `git commit -F "$tmp"`. Mention that this removes the safety net — the user has explicitly opted out of reviewing.
- **Different model / effort**: add `--model` or `--effort` flags to the `claude -p --bare` call.

## Don't

- Don't drop `--bare`. Without it, `claude -p` reads CLAUDE.md from the cwd, runs hooks, and may pull in irrelevant context that drifts the commit message away from the actual diff.
- Don't replace `git commit --edit -F` with a here-string or `-m "$(...)"` form. The temp-file path lets pre-commit hooks reformat the message; an inline `-m` string skips that pipeline.
- Don't `git add -A` inside the command. The user staged what they wanted; respect that.
