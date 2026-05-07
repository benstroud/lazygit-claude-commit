# lazygit-claude-commit

A [Claude Code](https://claude.ai/code) skill that adds a [lazygit](https://github.com/jesseduffield/lazygit) custom command for generating Git commit messages from the staged diff via the Claude CLI.

## What it does

Drops a `customCommands` entry into your lazygit user config (whatever `lazygit --print-config-dir` reports — `~/Library/Application Support/lazygit/config.yml` on macOS, `~/.config/lazygit/config.yml` on Linux) so that, from the lazygit Files panel, you can press `Ctrl+A` to:

1. Pipe the staged diff (`git diff --cached`) to `claude -p --bare`.
2. Open the generated message in your `$EDITOR`, pre-filled.
3. Commit (or quit empty to abort) with pre-commit hooks running normally.

Why these specific choices:

- **`claude -p --bare`** — `--bare` skips hooks, CLAUDE.md auto-discovery, auto-memory, plugin sync, and keychain reads. Fastest startup, no surprising context, the model only sees the diff.
- **`git commit --edit -F <tmp>`** — the model drafts; you approve. Lets you fix bad output without rerunning, and pre-commit hooks still run.
- **Empty-staged-diff guard** — refuses to call Claude (or open an empty editor) when there's nothing staged.

## Prerequisites

- [lazygit](https://github.com/jesseduffield/lazygit) (`brew install lazygit`)
- [Claude Code CLI](https://docs.claude.com/en/docs/claude-code) (`claude` on `$PATH`)
- [Claude Code](https://claude.ai/code) for invoking the skill

## Installation

Drop the skill into your user skills directory:

```bash
git clone https://github.com/benstroud/lazygit-claude-commit.git
cp -r lazygit-claude-commit/skills/lazygit-claude-commit ~/.claude/skills/
```

Then in any Claude Code session, ask Claude to set up Claude commit messages in lazygit and the skill will trigger.

## Usage

Inside lazygit:

1. Stage files (`s` on each, or `a` for all).
2. From the Files panel, press `Ctrl+A`.
3. Wait for `Asking Claude…`, then review/edit the message in your `$EDITOR` and save to commit.

With nothing staged, the command prints `No staged changes — stage files first.` and does nothing.

## Customisation

The skill itself documents the common knobs — different key binding, different prompt template, auto-commit without editor review, different model/effort. Re-run the skill and tell Claude what you want.

## License

MIT — see [LICENSE](./LICENSE).
