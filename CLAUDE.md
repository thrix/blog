# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this is

`thrix`'s personal blog ([vadkerti.net](https://vadkerti.net)) — a **Hugo**
static site using the **PaperMod** theme. Posts live in `content/posts/` as
Markdown. The site auto-deploys to GitHub Pages on every push to `main` via the
`.github/workflows/deploy.yaml` Action (Hugo extended `0.152.2`).

## Writing style — how posts should sound

These are the author's explicit preferences. Follow them when drafting or editing
any post.

- **Always load and apply the `stop-slop` skill** before drafting or editing any
  post. Run the prose through its checks and cut the patterns it flags.
- **Keep it technical and on point, not a fairy tale.** Lead with the mechanism,
  the commands, and the output. Trim narrative scene-setting and dramatic
  build-up; the story serves the technical content, not the other way around.
- **Sound human, not like an AI.** Write the way a tired engineer recounts a real
  session out loud. Specifically avoid the "AI tells":
  - Vary sentence length. Don't make every paragraph the same shape.
  - Go easy on em-dashes and the symmetrical "X — Y — Z" cadence.
  - Don't scaffold everything into tidy bold-lead bullet lists. Prose paragraphs
    are usually better, especially for takeaways.
  - Drop small human asides and reactions ("My laptop was useless for this",
    "Too late", "Nothing changed but the clock") instead of polished topic
    sentences.
  - Cut filler superlatives ("classic, under-appreciated", "powerful", "simply").
- **Redact / mangle sensitive details.** Don't leak more infra than the story
  needs. In particular:
  - Replace real IPs with RFC documentation ranges: `192.0.2.0/24`,
    `198.51.100.0/24`, `203.0.113.0/24` (and `10.x`/placeholder for internal).
  - Mangle real nameservers, internal hostnames, CI job IDs, ARNs, account IDs.
    Keep public project/domain names (e.g. `testing-farm.io`) when they're core
    to the story.
- **Default structure is a debugging war story:** symptoms → how I reproduced it
  → investigation steps with real command output → root cause → the fix →
  what I'd take away. (See `how-a-single-regex-stalled-30-testing-farm-jobs.md`
  and `dns-negative-caching-broke-our-ephemeral-ci.md` for the tone.)
- First person, concrete, show real `console`/`text` blocks. Explain the
  underlying concept briefly when it's the crux (e.g. a short "what is X anyway"
  detour), then get back to the story.

## Post conventions

- **Location & name:** `content/posts/<kebab-case-title>.md`.
- **Front matter** (see `archetypes/default.md`):

  ```yaml
  ---
  title: "Sentence-case Title"
  date: 2026-06-30          # YYYY-MM-DD
  draft: true
  tags: ["topic", "tool"]
  summary: "1–2 sentence teaser shown in listings and meta description."
  ---
  ```

- **Drafts:** new posts start `draft: true`. The site is built with
  `buildDrafts: false` (in `hugo.yaml`) and the deploy Action does a plain
  `hugo` build, so **a draft is never published even when pushed** — it's safe to
  commit/push drafts.
- **Publishing is an explicit, separate step:** only flip `draft: false` when the
  author explicitly asks to publish. Pushing that change to `main` triggers the
  deploy and the post goes live. Never remove draft status on your own.
- Line wrapping isn't enforced (`MD013` is disabled); roughly matching the ~80-col
  soft-wrap of existing posts is fine but optional.
- Only touch the file you're working on — don't commit other unfinished drafts
  that happen to be untracked in the tree.

## Local workflow

```bash
hugo server -D     # preview INCLUDING drafts (the -D is the only way to see them)
hugo server        # preview as it will publish (drafts hidden)
hugo               # production build into ./public (no drafts)
```

The PaperMod theme is a **git submodule** (`themes/PaperMod`). After a fresh
clone: `git submodule update --init --recursive`.

## Commits

- Use `git commit -s` (Signed-off-by). **No** `Co-Authored-By` trailers.
- When Claude assisted, add an `Assisted-by: Claude Code` trailer.
- Markdown in commit messages: wrap constants/paths/commands in backticks.

### Pre-commit hooks (they block the commit on failure)

Configured in `.pre-commit-config.yaml`; run automatically on `git commit`:

- **codespell** — spell-checks prose. Watch for false positives on technical
  abbreviations (e.g. `ser` → "set"); either spell the word out or add it to
  `--ignore-words-list` in `.pre-commit-config.yaml`. Skips `themes/`, `public/`,
  `resources/`, `archetypes/`.
- **markdownlint-cli2** — config in `.markdownlint-cli2.yaml` (`MD013` line-length
  off, `MD033` inline-HTML allowed, `MD041` off, `MD024` siblings-only).
- **yamllint** (`--strict`), **gitleaks**, **actionlint**, plus the standard
  whitespace/EOF/merge-conflict/private-key hooks.

If a commit is rejected, fix the reported issue and re-commit — don't bypass with
`--no-verify`.

## Deploy

Push to `main` → GitHub Actions **"Deploy Hugo site to GitHub Pages"** builds and
publishes to `vadkerti.net`. Verify a run with the public API (the GitHub MCP
server may be offline):

```bash
curl -s "https://api.github.com/repos/thrix/blog/actions/runs?per_page=1" \
  | jq -r '.workflow_runs[0] | "\(.head_commit.message) -> \(.status)/\(.conclusion)"'
```
