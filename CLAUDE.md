# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this repository is

A **Claude Code skill package** — markdown only, no build step. It teaches an agent how to build a markdown-backed help center (category folders, frontmatter articles, static pages, client-side search, locale fallback, validation) in a Next.js App Router app.

The skill was written by the engineer who has shipped this module; `references/provenance.md` is the engineering ledger, thirteen entries recording what the audit of the earlier implementation changed and how the templates verify it. That file is the rationale layer: read it before "simplifying" anything.

## Structure

- `SKILL.md` — entry point. Frontmatter `description` is the trigger surface. The body carries the architecture, critical facts, hard rules, and the **reference directory table** mapping trigger keywords to `references/`.
- `references/*.md` — one topic per file, loaded on demand. `adaptation.md` (seam contract) and `content-model.md` (config + types) are the design entry points; the rest cover the loader, search, tags (`tags.md`: slug identity, tag page, chips, tag validation), i18n, routes, UI (`ui.md` shell/nav, `ui-content.md` cards/lists/renderer/style hooks), extensions and provenance.

## Editing conventions

- **Code blocks are compiled.** Every ` ```ts ` / ` ```tsx ` block whose first line is `// file: <path>` is extracted into a scratch project and type-checked under `strict` and `noUncheckedIndexedAccess`; the `*.test.ts` blocks are run. Keep that first line, keep imports complete, and re-run the check after editing any block.
- **Identifiers are shared across files.** `HELP_CONTENT`, `HelpIndex`, `getHelpIndex`, `toSummary`, `toSearchDoc`, `helpHref`, `helpPath`, `HelpStrings`, `HelpTag`, `tagSlug`, `tagRefs`, `groupTags` appear in several references. Rename in all of them or none.
- **Keep the reference directory in sync** with `references/` — the table in `SKILL.md` and the file table in `README.md` both list every reference. Links are relative: `[x.md](references/x.md)` from SKILL.md, `[x.md](x.md)` between references.
- **Do not remove the odd-looking parts.** `hidden` instead of a max-height transition, `toSummary` before client props, `?? Number.MAX_SAFE_INTEGER` for order, the untrimmed-query note in search, the `tagSegment` throw in `config.ts`, span (not link) chips inside list rows: each is a documented ledger entry or an HTML rule. Check `provenance.md` first.
- **Four non-negotiables** live in `adaptation.md` ("What is not negotiable"). Never present them as optional elsewhere.
- Category ids `app`, `general`, `development` are deliberately generic; the rename procedure is in `adaptation.md`. Frontmatter field names are the authoring contract and are never renamed.
- Additions beyond the earlier implementation are marked as such in `provenance.md` ("Added"). New capability goes there too, or in `extensions.md` as a design.
