# Extensions

Capabilities the module does not ship. Each is described far enough to build
without re-deriving the design; none was in the earlier implementation (see
[provenance.md](provenance.md)), so treat them as designs, not proven code.

## Full-text search

Headings cover most "the term is only in the body" misses. When they stop
being enough — a corpus past ~200 articles, or long reference pages — pick by
where the index should live:

| Approach | Index lives | Cost | When |
|---|---|---|---|
| Ship body tokens in `HelpSearchDoc` | client, in the page payload | +2–5 KB per article on every help page | < 100 articles, short bodies |
| Prebuilt JSON fetched on focus | `public/help-index.{locale}.json`, generated in `validate:help` | one request, cached | 100–1000 articles |
| Pagefind | static files built post-`next build` | zero runtime, excellent ranking | any size, static hosting |
| Route handler + server search | server memory | a request per keystroke | needs live content or auth |

For the first two, extend `toSearchDoc` with `body: tokenize(article.content)`
deduplicated, add `body: 1` to `WEIGHTS`, and keep everything else. Keep
`prepareSearchDocs` memoised — tokenising bodies on every keystroke is the
mistake to avoid.

## Table of contents and heading anchors

The loader already extracts headings. Add `id` slugs to rendered headings with
`rehype-slug` and render `article.headings` as an in-page TOC in the right
margin on `xl:` screens. Anchor ids must be stable across locales if links are
shared — derive from the English heading or from an explicit `{#id}` suffix.

## "Was this helpful?"

A two-button widget at the end of the article posting
`{ category, slug, locale, helpful: boolean, comment? }` to a route handler
that writes to the host's analytics or database. Rate-limit per IP per article.
Show the aggregate (`87% found this helpful`) only after N ≥ 20 votes — early
percentages are noise that discourages readers.

## `updatedAt` from git

Manual `updatedAt` rots: in the audited corpus more than half the articles
carried the same date, clearly the import date. Derive it at build:

```bash
git log -1 --format=%cs -- content/help/app/get-started.md
```

Run for every file in `validate:help`, emit `content/help/.dates.json`, and
have the loader prefer the frontmatter value when present (explicit override)
and the git date otherwise. Requires full history in CI (`fetch-depth: 0`).

## Components in articles (MDX)

Callouts, tabs, an embedded booking widget. Switch the renderer to
`next-mdx-remote` or `@next/mdx` with a small allowlist of components.
Frontmatter, loader, search and routes do not change: MDX is a rendering
concern. Validate that every component used in a file is in the allowlist —
an unknown component is a build error, which is the right outcome.

## A CMS or database as the source

The index is the only contract. Implement a source that returns
`HelpArticle[]` per locale:

```ts
export type HelpSource = { load(locale: string): Promise<HelpArticle[]> };
```

and have `buildIndex` call it instead of walking the filesystem. `assembleIndex`
already accepts in-memory articles. Then:

- Move `dynamicParams = false` to `true` and add `revalidate`, or keep static
  generation and trigger a rebuild on publish.
- Keep validation: run it against the CMS export on a schedule.
- Search stays client-side until the corpus outgrows it.

## AI assistant grounded on the articles

If the host has a chat surface, hand it the search index as a tool
(`searchHelp(docs, query)`) and the article bodies as a retrieval tool by
`category/slug`. Cite the article URL in the answer. Do not embed the whole
corpus in the system prompt: a few dozen articles already run to a hundred
thousand tokens. Log
questions with no matching article next to the zero-result search log; they
are the same signal.

## Redirects from a legacy help site

Consolidating a `help.example.com` into `/help` is the usual reason this module
gets built. Export the old URL list, map each to `category/slug`, and ship the
map as permanent redirects in the host config. Validate the map against the
index in `validate:help` so a renamed slug fails the build rather than 404ing
old inbound links.

## Search analytics

`reportHelpSearchMiss` is the hook. Also worth logging: queries that returned
results but were not clicked (the result was wrong), and the position of the
clicked result (ranking quality). Three events, one dashboard, and the content
roadmap writes itself.
