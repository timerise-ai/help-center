---
name: help-center-markdown
description: >
  Build a markdown-backed help center in a Next.js App Router app — category
  folders of frontmatter articles, statically rendered landing, category, tag
  and article pages, client-side search with ranking and keyboard navigation,
  locale-aware routing with default-locale fallback, tags with slug identity
  and per-tag pages, related articles, breadcrumbs, JSON-LD and sitemap
  entries, plus a content validator for CI. Use when: (1) building or
  extending a help center, knowledge base, docs section, support articles or
  FAQ hub from markdown files in the repo, (2) adding article search, tag pages
  or tag chips, translated articles with fallback, hreflang/canonical rules,
  related links, or a collapsible article sidebar, (3) the user mentions: help
  center, knowledge base, docs site, support articles, markdown content,
  frontmatter, tags, article search, hreflang, i18n content, "help articles".
  Carries a loader with a CI validator that fails the build on unreachable
  or malformed articles, search that tolerates trailing spaces, punctuation
  and diacritics, and tags grouped by slug so one spelling variant cannot
  split a page. Next.js App Router; content in the repository, no CMS.
---

# Help Center — markdown articles, search, i18n

A help center is a small static site with one hard requirement: **every
article must be reachable** — from the sidebar, from search, from a translated
locale that does not have it yet. Written by the engineer who has shipped
this module; the earlier implementation was a marketing-site help center. The
module below holds that requirement as verified properties: the sidebar shows
every article of every category, search ranks a query however it is typed,
related links resolve or the validator fails the build. The loader, search
and tag suites cover each; the record is in `references/provenance.md`.

## When to use

Building or extending a help center, docs section or knowledge base whose
articles are markdown files in the repository, on Next.js App Router, with or
without multiple locales.

## When NOT to use

- **A blog.** Dated, authored posts with covers and localized slugs are a
  different content model; keep the host's blog module.
- **Docs generated from code** (OpenAPI, TypeDoc) — use their generators and
  link to the output.
- **A CMS-backed help site** with editors publishing at runtime — the index
  contract still applies, but the loader, static params and validation change;
  see the CMS note in [extensions.md](references/extensions.md).
- **Marketing pages that happen to be markdown.** Reuse the host's renderer;
  this module is the navigation, search and locale model around many articles.

## Architecture

```
content/help/<[locale]/><category>/<slug>.md
        │  gray-matter + normalisation
        ▼
getHelpIndex(locale)  ── React cache(): one parse per request ──┐
  categories · articles (sorted) · byKey · tags · byTag · skipped│
        │                                                        │
        ├─► pages: landing / category / tag / article ── metadata, JSON-LD, canonical/hreflang
        ├─► HelpShell ── toSummary → nav tree (sidebar + drawer)
        │             ── toSearchDoc → HelpSearch (client, tokenised prefix search)
        ├─► helpSitemapEntries()
        └─► validate:help (CI) ── unresolved related, order ties, bad dates, bad slugs, tag drift
```

Everything reads from the index. Nothing else touches the filesystem.

## Critical facts

1. **The filesystem is the schema.** Folder = category, filename = slug, and
   frontmatter is metadata only. `slug`/`category` in frontmatter are validated
   against the path, never used.
2. **Only summaries cross to the client.** `toSummary`/`toSearchDoc` strip the
   body; passing `HelpArticle` to a client component ships the whole corpus in
   every page.
3. **An untranslated page is a duplicate.** It renders from the default
   locale, keeps its place in navigation, shows a notice, canonicalises to the
   default-locale URL, and is absent from hreflang and the sitemap.
4. **The runtime forgives; CI does not.** Bad references are dropped, missing
   orders sort last — and `validate:help` fails the build for the author.
5. **Search is client-side by design.** ~1 KB per article, already in the page
   for navigation, results on the first keystroke. Body search is an extension.

## Hard rules

> **Never cap a collapsible list's height.** A `max-h-*` + `overflow-hidden`
> pair clips whatever does not fit and reports nothing. Use `hidden` or a real
> collapsible primitive, so a list renders every link or none.

> **Never sort by `order` alone.** Ties fall through to `readdir` order, which
> differs between filesystems. Break ties by title, then slug.

> **Never resolve `related` without validating it.** A dropped reference is
> invisible on the page; the validator is the only place an author learns a
> slug was renamed.

> **Never search an untrimmed query, and never search title alone.** Trim,
> tokenise, fold diacritics, include tags and headings, rank by field.

> **Never hardcode chrome strings** in a host with an i18n system. Every
> string goes through `HelpStrings`, so the header, footer and notices switch
> locale together.

> **Never key a tag by its spelling.** Case, hyphens and plurals drift across
> authors. Group by `tagSlug` so every variant lands on one page, and let the
> validator report the drift.

## Quick start

0. Fill in the seam contract and settle the category ids —
   [adaptation.md](references/adaptation.md).
1. Choose the content layout and write the config —
   [content-model.md](references/content-model.md).
2. Copy the loader and frontmatter parser; run the validator on the real
   content immediately — [content-loader.md](references/content-loader.md).
3. Wire strings and paths to the host's i18n — [i18n.md](references/i18n.md).
4. Add the three pages, SEO helpers and sitemap entries —
   [routes.md](references/routes.md) — then the tag page and linked chips —
   [tags.md](references/tags.md).
5. Build the shell, navigation tree and drawer — [ui.md](references/ui.md) —
   then the breadcrumb, cards, lists and renderer on the host's primitives —
   [ui-content.md](references/ui-content.md).
6. Drop in search and its hook; wire the zero-result hook to analytics —
   [search.md](references/search.md).
7. Run the shipped tests, then check the behaviour contract: longest category
   fully reachable, trailing-space query, accented query, untranslated page's
   canonical.

## Reference directory

| Scenario | Trigger keywords | Reference |
|---|---|---|
| Fitting this into an existing app | adapt, seam, rename categories, host probe, integrate, dependencies | [adaptation.md](references/adaptation.md) |
| Frontmatter, folders, slugs, validation | frontmatter, slug, category, order, related, updatedAt, featured, validate, CI | [content-model.md](references/content-model.md) |
| Reading files into the index | loader, gray-matter, cache, fallback, headings, sort, index, fs | [content-loader.md](references/content-loader.md) |
| Search box and ranking | search, combobox, ranking, tokenize, diacritics, prefix, keyboard, no results | [search.md](references/search.md) |
| Tags, tag pages, chips, tag cloud | tag, tags, tag page, chip, tag slug, spelling, plural, browse by tag | [tags.md](references/tags.md) |
| Locales, strings, dates, canonicals | i18n, locale, translation, fallback, hreflang, strings, plural, Intl | [i18n.md](references/i18n.md) |
| Pages, metadata, JSON-LD, sitemap | route, page.tsx, generateStaticParams, dynamicParams, notFound, canonical, TechArticle, sitemap | [routes.md](references/routes.md) |
| Shell, header, sidebar, drawer | layout, sidebar, mobile menu, drawer, collapsible, sticky, HelpShell | [ui.md](references/ui.md) |
| Breadcrumb, cards, lists, renderer, style hooks | breadcrumb, category card, article list, react-markdown, data-help, styling | [ui-content.md](references/ui-content.md) |
| Beyond the shipped module | full-text, Pagefind, TOC, feedback, git dates, MDX, CMS, redirects | [extensions.md](references/extensions.md) |
| Why the templates differ from the earlier implementation | provenance, audit, ledger, kept deliberately, fixing the earlier implementation | [provenance.md](references/provenance.md) |
