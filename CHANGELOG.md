# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.7] - 2026-09-02

Wording release. Templates and technical content are unchanged from 0.2.6.

### Changed
- The corpus figures of the earlier implementation (article, file, tag and category
  counts, and ratios such as "25 of 45 links") are gone from `SKILL.md`, `CLAUDE.md` and
  `references/`, replaced by the shape of each finding. Design thresholds and the
  skill's own test counts stay. Six remaining "the original" mentions in
  `references/` now say the earlier implementation.

## [0.2.6] - 2026-09-02

Documentation-only release. The skill itself, `SKILL.md` and `references/`, is
unchanged from 0.2.5.

### Changed
- README: the intro states the origin in the words of the skill standard and names the
  audit's findings by kind, without the corpus figures, which stay in
  `references/provenance.md` where they were measured. The intro links the ledger.

## [0.2.5] - 2026-09-02

Wording release. Templates and technical content are unchanged from 0.2.4.

### Changed
- `SKILL.md`, `references/provenance.md` and `references/tags.md`: the tag drift the
  audit found is described by its shape (case pairs, a hyphenated two-word feature
  name, one integration's name in three spellings, singular/plural pairs) with the
  measured counts kept, instead of listing the corpus's own tag labels.

## [0.2.4] - 2026-09-02

Wording release. The origin and audit statements across the skill follow section 2 of
the skill standard; templates and technical content are unchanged from 0.2.3. The
repository history starts at this release.

### Changed
- Origin and audit wording across `SKILL.md`, `CLAUDE.md`, `README.md` and `references/` now
  follows the skill standard: the reference point is the earlier implementation, stated
  in the standard's own words. The frontmatter description says
  the loader is audited and the fixes are for the defects the audit found.

## [0.2.3] - 2026-09-02

Documentation-only release. The skill itself, `SKILL.md` and `references/`, is
unchanged from 0.2.2.

### Changed
- README: the install leads with `npx skills add timerise-ai/help-center-markdown`, which installs the skill
  into every skills-compatible agent it detects, with the `-a` form for named agents; the
  Claude Code clone moves under a *Manual install* heading. Activation gets its own
  heading, and a *Not this* table points neighbouring problems to the right skill or tool.
- README: the skill's origin is reworded. It was written by the engineers who built the
  module it describes; the reference point for `provenance.md` is the earlier
  implementation rather than "the source"; the index is called Timerise Skills.
- README: every em-dash, arrow and en-dash in the prose is rewritten as a comma, colon,
  full stop or conjunction.

## [0.2.2] - 2026-09-01

### Changed
- `README.md` is host-neutral and every claim is checked against `SKILL.md` and
  `references/`: the skill is introduced as an [Agent Skill](https://agentskills.io)
  rather than a Claude Code skill, the pitch names the tag layer, related
  articles, breadcrumbs, JSON-LD and sitemap entries it already ships, the
  extraction figures match `provenance.md` (65 articles, three categories,
  three locale prefixes), and a new paragraph states the `getHelpIndex` seam —
  nothing but the loader touches `fs`, so a CMS or database source swaps in
  without changing routes, search or tag pages — and that the module is public,
  read-only and file-backed.
- The install section names the current release and how to scope the skill to a
  single project.
- Repository metadata: a host-neutral GitHub description, and topics for the
  other skills-compatible agents (`agent-skill`, `codex`, `gemini-cli`) in place
  of `claude-code-skill`.

### Added
- `README.md` install instructions for every skills-compatible agent, in the
  wording used by the [skills index](https://github.com/timerise-ai/skills):
  the one-command [skills.sh](https://www.skills.sh) form
  (`npx skills add timerise-ai/help-center-markdown`), and a Codex CLI / Gemini
  CLI section covering `~/.agents/skills`, symlinking rather than cloning twice,
  the differing invocation syntax, and testing activation explicitly on a first
  run.

## [0.2.1] - 2026-08-30

### Changed
- Renamed the skill and repository from `help-center` to `help-center-markdown`:
  `SKILL.md` frontmatter `name`, the README title, install path and clone URL,
  and the `origin` remote. GitHub redirects the old repository URL.

### Added
- `README.md` links the [Timerise skills index](https://github.com/timerise-ai/skills)
  — from the install section and a "Part of the Timerise skills" section naming
  the shared layout the skills follow.
- `LICENSE` — MIT. The README declared MIT but no license text shipped,
  leaving the terms unstated for anyone cloning the skill.

## [0.2.0] - 2026-08-29

Second pass: tags become a first-class layer with their own pages.

### Added
- `references/tags.md`: tags keyed by slug (`tagSlug`, folded like search),
  `index.tags` / `index.byTag`, a `/help/tag/[tag]` page, linked `HelpTagChips`
  on the article header and a landing-page tag cloud, tag sitemap entries,
  `tagHeading` / `tagsHeading` strings, and `HELP_CONTENT.tagSegment` with an
  import-time guard against shadowing a category.
- Validation warnings for tag spelling variants, singular/plural pairs and
  labels with no URL-safe form, with tests (`tags.test.ts`).
- Per-key English fallback for JSON-dictionary hosts in `getHelpStrings`.
- `provenance.md` adds defects 12 (a line-based
  frontmatter parser that dropped multi-line arrays in 45 of 161 files) and
  13 (tags keyed by spelling: 6 variant groups and 5 plural pairs in a
  65-article corpus).
- `README.md`: what the skill is, install, the reference file table, the four
  non-negotiables, contributing conventions, and author credit.

### Changed
- `HelpArticleList` links each row by `article.category` and no longer takes
  a `category` prop, so one list serves category and tag pages.
- The "Tag pages" extension sketch is gone; the capability is shipped.
- `CLAUDE.md` now names both `SKILL.md` and `README.md` as the tables that must
  list every file in `references/`.

## [0.1.0] - 2026-08-29

Initial release of the help-center skill.

### Added
- `SKILL.md` entry point with architecture, critical facts, hard rules, and the
  reference directory table.
- `references/` covering the seam contract (`adaptation.md`), content model and
  validation, the cached content loader with locale fallback, tokenised search
  with keyboard navigation, i18n strings/paths/dates, routes with SEO helpers
  and sitemap entries, UI shell/navigation/drawer/lists, extensions, and
  provenance.
- Tests for the search scorer, the validator, and the loader (fixture tree),
  embedded in the references and verified under `strict` +
  `noUncheckedIndexedAccess`.

### Fixed
- Eleven defects from the source module, documented in
  `references/provenance.md` — including a sidebar height cap that hid 25 of 45
  articles and a search that ignored trailing whitespace, tags and headings.
