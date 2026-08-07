# gemini-commit

Smart `git commit` from the command line, powered by the [Antigravity CLI](https://antigravity.google/docs/cli/install) (the successor to the Gemini CLI).

Run `commit` in any git repository and Antigravity inspects your working tree, writes commit messages matching your repo's existing style, and commits — splitting large changesets into logical separate commits when appropriate.

## Requirements

- [Antigravity CLI](https://antigravity.google/docs/cli/install) installed and authenticated (`agy` must be on your `PATH`)
- Any POSIX shell — the script is plain `sh`, so it runs under `sh`, `bash`, `zsh`, `dash`, `ash` and `ksh` on macOS and Linux, and under Git Bash or WSL on Windows
- `git`

## Installation

Clone the repo and symlink the script somewhere on your `PATH`:

```sh
git clone https://github.com/timohaa/gemini-commit.git
ln -s "$(pwd)/gemini-commit/commit" ~/.local/bin/commit
```

Make sure `~/.local/bin` is on your `PATH`.

## Usage

```text
Usage: commit [options] [custom instructions...]

Options:
  -n, --no-yolo     Disable YOLO mode (ask for confirmation before executing actions)
      --safe        Alias for --no-yolo
  -p, --push        Automatically push after successful commit(s)
      --amend       Amend the last commit instead of adding one (unpushed only)
      --tag NAME    Tag the result: a tag name, or major, minor or patch
  -q, --quick       Use the fastest low-effort model listed by 'agy models'
  -d, --deep        Use the strongest high-effort model listed by 'agy models'
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

Fold the working tree into the commit you just made:

```sh
commit --amend
commit --amend "the subject should say parser, not lexer"
```

Cut a release:

```sh
commit --tag v2.3.0
commit --tag minor -p
```

## Amending

`--amend` folds the working-tree changes into `HEAD` instead of adding commits, and on a clean
working tree it rewords `HEAD`'s message and changes nothing else. Exactly one commit exists
either way when the run finishes.

It only applies to commits that are still private. Before anything runs, the script refuses if
`HEAD` is reachable from the branch's upstream or from any remote-tracking ref, so there is no
force-push mode to reach for — an amended commit can always be pushed with a plain `git push`.
Two smaller refusals guard the same intent: an amend is rejected outright while a merge,
cherry-pick or revert is in progress, and in a repository with no commits at all.

Those checks read local refs and never fetch, which is the one hole worth knowing about: a
commit pushed from another machine since your last fetch still looks private. The checks are
therefore biased toward refusing. You also get a warning, not a refusal, when the branch has no
upstream to check against, when `HEAD` is a merge commit, and when a tag points at `HEAD` (that
tag stays behind on the pre-amend commit).

## Tagging

`--tag` adds an annotated tag on the commit the run produces. The value is mandatory, and is
either a literal tag name or one of `major`, `minor` or `patch`:

```sh
commit --tag v2.3.0        # exactly this name
commit --tag patch         # v2.2.0 -> v2.2.1
commit --tag=minor         # v2.2.0 -> v2.3.0
```

A bump is resolved by the script, not the agent, from the nearest tag reachable from `HEAD`
(`git describe --tags --abbrev=0`), keeping whatever prefix that tag uses — `release-1.4.7`
patches to `release-1.4.8`. Tags that are not `<prefix><major>.<minor>.<patch>`, such as a
pre-release `v3.0.0-rc2`, are refused rather than guessed at; pass an explicit name for those.
Resolving up front means an invalid name, an unbumpable base or a tag that already exists all
fail before `agy` starts, and that `--show-prompt` shows the name that will really be used.

The value is mandatory because bare words are custom instructions: with an optional argument,
`commit --tag "also bump the readme"` would name a tag after the instruction.

The tag stays local unless you also pass `-p`, which pushes it by name (`git push origin <tag>`,
never `--tags`, which would publish every local tag you happen to have). Tagging does not edit
version strings in the source tree — ask for that in the instructions if you want it, e.g.
`commit --tag minor "bump VERSION to 2.3.0"`. With a clean working tree, `--tag` tags `HEAD` as
it stands and commits nothing.

## Model selection

By default no model flag is passed to Antigravity, letting it use whatever model you've configured as default. Two presets cover the cases where you want to lean one way:

```sh
commit --quick    # small, obvious changesets
commit --deep     # large or subtle refactors
```

Neither preset names a model literally. Both read `agy models` at runtime and pick the first entry matching a short list of patterns, in order of preference:

| Preset    | Patterns tried                                  |
|-----------|-------------------------------------------------|
| `--quick` | `*flash*-low`, `*-low`, `*flash*`, first listed |
| `--deep`  | `*pro*-high`, `*pro*`, `*-high`, first listed   |

`agy models` lists newest first, so the first match is also the most recent model of that kind — as of writing, `gemini-3.6-flash-low` and `gemini-3.1-pro-high`. When Google ships a new generation the presets follow it with no edit here. If `agy models` can't be read at all (offline, older CLI), the presets fall back to those two names.

Anything else is reachable directly, and the last model flag on the command line wins:

```sh
commit --model gemini-3.5-flash
commit --effort high
```

If the requested model isn't in the `agy models` list, the script warns and falls back to Antigravity's default rather than failing.

## How it works

After validating that you're in a git repository with uncommitted changes (or a clean one, if `--amend` or `--tag` still gives it something to do), the script gathers a little context itself — the current branch, the last 5 commit messages in full, the last 20 commit subjects, `git status --porcelain`, and a diffstat against `HEAD` — and inlines it into the prompt. That saves the agent from spending its first few turns re-deriving the same information on every run. `--amend` adds `HEAD`'s message verbatim and `git show --stat HEAD`, since an amended commit is the sum of what `HEAD` already had and what's in the working tree. `--tag` adds the existing tags and the commit subjects since the last one.

The two log views do different jobs. The full messages show what a subject line alone can't: whether bodies are used at all, how they're formatted and wrapped, and which trailers the repo carries. The wider subject sample shows the prevailing subject convention. Both are capped, so the prompt stays a couple of kilobytes regardless of repository size.

`Co-authored-by:` trailers are filtered out of the full messages before they're sent. Shown as part of the sample they read as house style, and get copied onto commits the named co-author had no hand in. Other trailers — `Signed-off-by:`, `Refs:` — are left intact. One consequence worth knowing: the agent won't reproduce co-author trailers even in a repo that uses them deliberately, because it never sees them. The one exception is `--amend`, where `HEAD`'s own message is repeated verbatim with its trailers, since that message is the one being rewritten and stripping it would quietly drop a co-author who genuinely worked on the commit.

Actual diffs are deliberately left out. The agent reads those itself and can be selective per file, which keeps the prompt size independent of how large the changeset is.

The prompt then asks it to analyze the diffs before deciding anything, group the changes into one logical change per commit, order those commits so dependencies land first and the history reads chronologically, match the message style of the recent commits, and stage each commit with explicit paths.

`--amend` and a clean-tree `--tag` replace that instruction list rather than adding to it — telling the agent to group the changes into commits and then to produce exactly one amended commit would just be a prompt arguing with itself.

Antigravity is launched in interactive mode (`agy -i`), so you keep the session after the commits land. By default it runs in YOLO mode (`--dangerously-skip-permissions`), letting it execute git commands without per-command confirmation; use `--no-yolo` if you want to approve each step. To see exactly what's being sent without running anything, use `--show-prompt`.

## Versioning

Releases follow [semantic versioning](https://semver.org) and are tagged on GitHub (`v<major>.<minor>.<patch>`), newest first:

- v2.3.0 — added `--amend`, which refuses any commit already reachable from a remote, and `--tag` for cutting annotated release tags; it also fixes `--quick`/`--deep` model resolution when `zsh` is used as the interpreter directly (`zsh commit`), where zsh's native splitting and pattern-matching rules made both presets fall back to their hard-coded default.
- v2.2.0 — rewrote the script in POSIX `sh` (dropping the `zsh` requirement), made `--quick` and `--deep` resolve against `agy models` instead of hard-coded model names, and added recent commit messages in full to the prompt so body and trailer conventions are visible alongside subject style; it also joins unquoted instruction words into a single instruction, rejects bare repositories and `.git` directories properly, and lets `--show-prompt` run without the Antigravity CLI installed.
- v2.1.0 — added model selection and precomputed repository context in the prompt.
- v2.0.0 — switched to the Antigravity CLI.
- v1.x — used the Gemini CLI.

## License

[MIT](LICENSE)
