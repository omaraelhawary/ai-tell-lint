# ai-tell-lint

A linter for prose that sounds machine-written. It runs as a CLI, as a Claude Code plugin, and as a GitHub Action. Python 3.9 or newer, standard library only, no network calls.

## What it catches

Regex rules, grouped by category. Block categories fail the check. Warn categories only report.

- Block: AI vocabulary, filler connectors, negative parallelism, compulsive summaries, puffery, editorializing, vague attribution, tailing -ing clauses.
- Warn: figurative "navigate", false ranges, rule of three, hedging stacks, curly quotes and ellipsis characters, emoji in headings.
- Em dashes are counted per 150 words (default limit 1).

Code blocks and inline code are ignored. Before and after:

```
Before: It's not just a tool, it's a game-changer. Moreover, it plays a pivotal role in
streamlining your workflow, highlighting its seamless integration.
```

After: This tool cuts the setup steps from six to two.

## Install as a Claude Code plugin

```
/plugin marketplace add omaraelhawary/ai-tell-lint
/plugin install ai-tell-lint@ai-tell-lint
```

The plugin hooks run before Bash and file-write calls. They deny commit messages, `gh` issue, PR and release text, and writes to prose files (`.md .mdx .markdown .txt .rst`) that trip a block rule. The hook fails open: any error exits 0 and prints nothing. The plugin and the Action both call `python3` (3.9 or newer), so it must be on `PATH`.

## Install the CLI

```
pipx install git+https://github.com/omaraelhawary/ai-tell-lint
ai-tell-lint README.md
```

Or without installing, from a checkout: `python3 -m ai_tell_lint README.md`.

Flags: `--config PATH`, `--format text|github` (default `text`), and `-` or no file to read stdin. Exit codes: 0 clean, 1 block hit, 2 usage or config error.

## GitHub Action

The action lints the lines a pull request adds in prose files, the PR description, and commit messages. It needs full history to find the base commit, so set `fetch-depth: 0`. A shallow clone exits 2.

```yaml
name: prose
on: pull_request

permissions:
  contents: read

jobs:
  ai-tell-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        with:
          fetch-depth: 0
      - uses: omaraelhawary/ai-tell-lint@v1  # works once the v1 tag is published
        with:
          commits: warn
```

Inputs:

- `commits`: `warn` (default) or `block`. Level for commit message hits.
- `config`: path to a config file. A repo-relative path is read from the base commit, so a PR cannot loosen its own rules. Default is `.ai-tell-lint.json` as it exists on the base branch.

Use `pull_request`, not `pull_request_target`. The latter runs with secrets and may not have the head commit.

## Config

Lookup order for the CLI and the hook: `--config`, then `.ai-tell-lint.json` at the repo root, then `~/.config/ai-tell-lint/config.json`. The first file found is used. To check a config the hook would silently ignore, run `ai-tell-lint < /dev/null` in the repo: exit 2 means the config is bad. Your patterns are added to the bundled rules.

Keys:

```json
{
  "block": { "House ban": ["\\bsynergy\\b"] },
  "warn": { "Soft ban": ["\\bleverage\\b"] },
  "disable": ["Rule of three"],
  "warn_only": ["Puffery"],
  "max_em_dashes_per_150_words": 2,
  "check_replies": true
}
```

- `block`, `warn`: new categories, or extra regex patterns for an existing one.
- `disable`: drop a category.
- `warn_only`: move a block category to warn.
- `max_em_dashes_per_150_words`: number, default 1.
- `check_replies`: also check the assistant's final reply at Stop. Off by default.

An unknown key or a bad regex exits 2 in the CLI and names the offender. The hook ignores a bad config.

## Off switches

- Session or one command: `AI_TELL_LINT=off` (also `0` or `false`), for example `AI_TELL_LINT=off git commit -m "..."`.
- Everywhere: create the file `~/.config/ai-tell-lint/off`.
- Log blocks: set `AI_TELL_LINT_LOG` to a file path. Nothing is logged otherwise.

## License

MIT.
