# lazygit-claude-commit

A [Claude Code](https://claude.ai/code) skill that adds a [lazygit](https://github.com/jesseduffield/lazygit) custom command for generating Git commit messages from the staged diff via the Claude CLI.

## What it does

Drops a `customCommands` entry into your lazygit user config (whatever `lazygit --print-config-dir` reports — `~/Library/Application Support/lazygit/config.yml` on macOS, `~/.config/lazygit/config.yml` on Linux) so that, from the lazygit Files panel, you can press `Ctrl+A` to:

1. Pipe instructions + the staged diff (`git diff --cached`) to `claude -p --tools ""`.
2. Open the generated message in your `$EDITOR`, pre-filled.
3. Commit (or quit empty to abort) with pre-commit hooks running normally.

Why these specific choices:

- **Plain `claude -p`, not `claude -p --bare`** — `--bare` is tempting (skips CLAUDE.md, hooks, auto-memory) but also disables OAuth/keychain auth, demanding `ANTHROPIC_API_KEY` instead. Most Claude Code users authenticate via OAuth, so `--bare` returns "Not logged in" and the command exits before the editor opens.
- **`--tools ""`** — disables all tools so the model can only generate text. Without it the model may try to read related files or run bash to "understand context," turning a one-shot text generation into a multi-turn agent loop.
- **Instructions before diff in stdin** — when `claude -p` receives a diff first and instructions second (or instructions only via prompt-arg with diff via stdin), the model often replies "What would you like me to do with this diff?" instead of following the instructions. Putting instructions first then diff via stdin gets reliable single-message output.
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
