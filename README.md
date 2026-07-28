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
  -p, --push        Automatically push after successful commit(s)
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

## How it works

The script is a thin wrapper: after validating that you're in a git repository with uncommitted changes, it launches `agy` in interactive mode with a prompt instructing it to commit the changes, follow the repository's existing commit-message style, and optionally push. By default it runs in YOLO mode (`--dangerously-skip-permissions`), letting Antigravity execute git commands without per-command confirmation; use `--no-yolo` if you want to approve each step.

## Versioning

Releases follow [semantic versioning](https://semver.org) and are tagged on GitHub (`v<major>.<minor>.<patch>`). v1.x used the Gemini CLI; v2.0.0 switched to the Antigravity CLI.

## License

[MIT](LICENSE)
