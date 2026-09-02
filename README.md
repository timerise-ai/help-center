# help-center-markdown

An [Agent Skill](https://agentskills.io) that teaches an agent to build a markdown-backed help center in a
**Next.js App Router** app: category folders of frontmatter articles, statically rendered landing, category,
tag and article pages, client-side search with ranking and keyboard navigation, locale-aware routing with
default-locale fallback, tags with slug identity and their own pages, related articles, breadcrumbs, JSON-LD
and sitemap entries, and a content validator for CI. Content lives in the repository; there is no CMS.

A help center is a small static site with one hard requirement: **every article must be reachable**: from the
sidebar, from search, from a locale that has not been translated yet. This skill was written by the engineer
who built a production help center on this design (65 articles, three categories, three locale prefixes), and
every defect the audit of that earlier implementation found broke that requirement silently: a sidebar that
clipped 25 of 45 links, a search that returned nothing for a query with a trailing space, a frontmatter parser
that dropped multi-line arrays in 45 of 161 files, tags split in two by their spelling. The templates are
built so those cannot recur without a build failing or a test going red.

The filesystem is a seam, not a premise: one `getHelpIndex(locale)` builds the index every page, the search
box, the sitemap and the validator read from, and nothing else touches `fs`. Swapping in a CMS or a database
means implementing a source that returns articles for a locale, and the routes, search and tag pages do not
change. The module is public, read-only and file-backed: no auth, no tenancy, no database, no object storage.

## Install

One command, via the [skills.sh](https://www.skills.sh) CLI, which installs the skill into every
skills-compatible agent it detects, including Claude Code, Codex CLI and Gemini CLI:

```bash
npx skills add timerise-ai/help-center-markdown
```

Name the agents instead with `-a`, for example `npx skills add timerise-ai/help-center-markdown -a claude-code -a codex`.

Or clone it yourself. Nothing here is Claude-specific: the skill is a plain [Agent
Skills](https://agentskills.io) folder, `SKILL.md` plus markdown references with no file that calls a model,
so cloning it into an agent's skills directory is all an install is. For Claude Code:

```bash
git clone https://github.com/timerise-ai/help-center-markdown.git ~/.claude/skills/help-center-markdown
```

To scope it to a single project instead, clone it into that project's `.claude/skills/` directory. For another
agent, clone into that agent's skills directory, or symlink the Claude Code copy so one `git pull` updates
every agent:

```bash
mkdir -p ~/.agents/skills
ln -s ~/.claude/skills/help-center-markdown ~/.agents/skills/help-center-markdown
```

Update the skill with `git pull` in its directory. The current release is **0.2.5**. See
[`CHANGELOG.md`](CHANGELOG.md). The [skills index](https://github.com/timerise-ai/skills) lists the other
Timerise Skills and how to install them all at once.

## Activation

The skill activates automatically when a task matches its description: building or extending a help center,
knowledge base, docs section, support articles or FAQ hub from markdown in the repo, or adding article search,
tag pages and chips, translated articles with fallback, hreflang and canonical rules, related links or a
collapsible article sidebar. Invoke it explicitly with `/help-center-markdown` in Claude Code,
`$help-center-markdown` in Codex CLI, or from `/skills` in Gemini CLI.

Each host matches a task against the description its own way, so invoke the skill explicitly on a first run
rather than assuming it fired. Only `SKILL.md` is read up front; the `references/` files load on demand, so
the skill stays cheap in context until a topic is actually needed.

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Entry point: architecture, critical facts, hard rules, and the reference directory |
| `references/adaptation.md` | The seam contract with the host app:  styling, i18n, routing, the category rename |
| `references/content-model.md` | Config, types, frontmatter fields, slugs, and the CI validator |
| `references/content-loader.md` | Reading files into the index: gray-matter, caching, locale fallback, sorting |
| `references/search.md` | Client-side search: tokenizing, ranking, combobox keyboard behavior, no-results |
| `references/tags.md` | Tag slug identity, tag pages, chips, tag cloud, tag validation |
| `references/i18n.md` | Locales, strings, dates, default-locale fallback, hreflang and canonicals |
| `references/routes.md` | Pages, `generateStaticParams`, metadata, JSON-LD, sitemap entries |
| `references/ui.md` | Shell, header, sidebar, mobile drawer |
| `references/ui-content.md` | Breadcrumbs, category cards, article lists, the markdown renderer, style hooks |
| `references/extensions.md` | Full-text/Pagefind, table of contents, feedback, git dates, MDX, CMS, redirects |
| `references/provenance.md` | The thirteen defects found in the earlier implementation and how the templates fix them |

## The four non-negotiables

These travel with the module and are never optional (see `references/adaptation.md`):

1. **Summaries, never articles, cross to the client.** `toSummary` / `toSearchDoc` exist so the whole corpus
   is not serialised into every page.
2. **`hidden`, not a height cap, on collapsible nav.** A `max-height` cap with `overflow-hidden` is how the
   source lost half a category with no signal.
3. **Untranslated pages canonicalise to the default locale** and stay out of hreflang and the sitemap.
   Otherwise every fallback page is a duplicate.
4. **Validation runs in CI.** The runtime is deliberately forgiving: it drops bad refs and sorts missing
   orders last; the validator script is where authors find out.

Everything else is the host app's: styling, naming, renderer, i18n system.

## Not this

| Not this | Use instead |
|---|---|
| A blog: dated, authored posts with covers and localized slugs | The sibling [`blog-markdown`](https://github.com/timerise-ai/blog-markdown) skill; a different content model |
| Docs generated from code (OpenAPI, TypeDoc) | Their generators; link to the output |
| A CMS-backed help site with editors publishing at runtime | The index contract still applies, but the loader, static params and validation change; see the CMS note in `references/extensions.md` |
| Marketing pages that happen to be markdown | The host's renderer; this module is the navigation, search and locale model around many articles |

## Contributing

Issues and pull requests are welcome here. Pure markdown, with no build or lint step, but the code blocks are
checked: every ` ```ts ` and ` ```tsx ` block starting with `// file: <path>` is extracted into a scratch
project and type-checked under `strict` and `noUncheckedIndexedAccess`, with the `*.test.ts` blocks run.
Claims in this skill are meant to be verifiable: if you change a factual claim, say how you verified it,
whether against the library, the docs, or a reproduction.

Adding, removing or renaming a file in `references/` means updating the quick start and the reference
directory table in `SKILL.md`, the file table above, and any relative cross-links. The odd-looking parts of
the templates encode documented defects, and `references/provenance.md` is the ledger that must stay truthful:
read it before simplifying anything, and add an entry for anything you change. Commits follow Conventional
Commits and releases follow [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) in the
index; `CLAUDE.md` carries the full editing conventions.
## Part of the Timerise Skills

This is one of the [Timerise Skills](https://github.com/timerise-ai/skills): modules for **Next.js App
Router** apps written by our own senior engineers from the modules they have shipped, not synthetic, each
published as its own repository and indexed there. They share one layout, so an agent that has read one knows
how to read the next: a `SKILL.md` entry point, `references/` loaded on demand, and a seam contract carrying
the module's non-negotiables.

## Author

Built and maintained by [Timerise](https://timerise.ai).

## License

MIT. See [LICENSE](LICENSE).