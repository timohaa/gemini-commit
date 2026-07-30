# gemini-commit

Smart `git commit` from the command line, powered by the [Antigravity CLI](https://antigravity.google/docs/cli/install) (the successor to the Gemini CLI).

Run `commit` in any git repository and Antigravity inspects your working tree, writes commit messages matching your repo's existing style, and commits — splitting large changesets into logical separate commits when appropriate.

## Requirements

- [Antigravity CLI](https://antigravity.google/docs/cli/install) installed and authenticated (`agy` must be on your `PATH`)
- `zsh` (default shell on macOS)
- `git`

## Installation

Clone the repo and symlink the script somewhere on your `PATH`:

```sh
git clone https://github.com/timohaa/gemini-commit.git
ln -s "$(pwd)/gemini-commit/commit" ~/.local/bin/commit
```

Make sure `~/.local/bin` is on your `PATH`.

## Usage

```
Usage: commit [options] [custom instructions...]

Options:
  -n, --no-yolo     Disable YOLO mode (ask for confirmation before executing actions)
      --safe        Alias for --no-yolo
  -p, --push        Automatically push after successful commit(s)
  -q, --quick       Use a fast, low-effort model (gemini-3.6-flash-low)
  -d, --deep        Use a slower, high-effort model (gemini-3.1-pro-high)
      --model NAME  Use a specific model (see 'agy models')
      --effort STR  Reasoning effort: low, medium or high
      --show-prompt Print the generated prompt and exit without running agy
  -h, --help        Show this help message
  -v, --version     Show version
  --                Treat everything after as custom instructions (even if it starts with -)
```

### Examples

Commit everything, fully automatic:

```sh
commit
```

Guide the commit with natural-language instructions:

```sh
commit "only index.js changes"
commit "split the refactor and the bugfix into separate commits"
```

Commit and push:

```sh
commit -p "fix bug in parser"
```

Review each action before it runs:

```sh
commit --no-yolo
```

## Model selection

By default no model is passed and Antigravity uses whatever it is configured to use. Two presets cover the common cases:

```sh
commit --quick    # gemini-3.6-flash-low  — small, obvious changesets
commit --deep     # gemini-3.1-pro-high   — large or subtle refactors
```

Anything else is reachable directly:

```sh
commit --model gemini-3.5-flash
commit --effort high
```

Run `agy models` to see what's currently available. If the requested model isn't in that list, the script warns and falls back to Antigravity's default rather than failing — so the presets keep working if Google renames a model.

## How it works

After validating that you're in a git repository with uncommitted changes, the script gathers a little context itself — the current branch, the last 20 commit subjects, `git status --porcelain`, and a diffstat against `HEAD` — and inlines it into the prompt. That saves the agent from spending its first few turns re-deriving the same information on every run.

The prompt then asks it to analyze the diffs before deciding anything, group the changes into one logical change per commit, order those commits so dependencies land first and the history reads chronologically, match the message style of the recent subjects, and stage each commit with explicit paths.

Antigravity is launched in interactive mode (`agy -i`), so you keep the session after the commits land. By default it runs in YOLO mode (`--dangerously-skip-permissions`), letting it execute git commands without per-command confirmation; use `--no-yolo` if you want to approve each step. To see exactly what's being sent without running anything, use `--show-prompt`.

## Versioning

Releases follow [semantic versioning](https://semver.org) and are tagged on GitHub (`v<major>.<minor>.<patch>`). v1.x used the Gemini CLI; v2.0.0 switched to the Antigravity CLI; v2.1.0 added model selection and precomputed repository context in the prompt.

## License

[MIT](LICENSE)
