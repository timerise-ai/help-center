# Tags

Tags are the cross-category axis: a category answers "where does this
article live", a tag answers "what else is about calendars". They are cheap
for authors — one frontmatter line — and that is exactly the problem: nothing
stops `API` on Monday and `api` on Tuesday. The audited corpus, 65 articles,
had `API` and `api`, `integration` and `integrations`, `Booking page` and
`booking-page`. Rendered as plain chips nobody noticed; as listing pages each
pair would have been two half-empty pages.

So the module keys tags by **slug**, not by spelling. Spelling is display;
the slug is identity, the URL, and the search token.

## Model

| Concept | Where | Notes |
|---|---|---|
| `article.tags: string[]` | frontmatter, `HelpArticleSummary` | The author's spellings, untouched. Rendered as chips; a search field. |
| `tagSlug(label)` | `tags.ts` | `"Booking page"` → `booking-page`, `"Płatności"` → `platnosci`. Same folding as search. |
| `HelpTagRef { slug, label }` | `model.ts` | What a chip links with. |
| `HelpTag { slug, label, count }` | `model.ts`, `index.tags` | One per slug; `label` is the most frequent spelling. Sorted most-used first. |
| `index.byTag` | `HelpIndex` | slug → articles, in index order. The tag page's whole query. |
| `HELP_CONTENT.tagSegment` | `config.ts` | The literal URL segment (`tag`). Must match the route folder and must not be a category id, or the static segment shadows that category — `config.ts` throws at import if it is. |

```ts
// file: src/lib/help/tags.ts
import type { HelpArticle, HelpTag, HelpTagRef } from "./model";
import { normalizeText } from "./search";

/**
 * A tag's identity is its slug, not its spelling. Lowercase, fold diacritics
 * with the table search uses (so a typed query and a tag URL agree), collapse
 * every run of non-alphanumerics into one hyphen, trim the ends:
 * `"Booking page"` → `booking-page`, `"API"` → `api`, `"Płatności"` → `platnosci`.
 * Returns `""` for a label with nothing usable (`"#"`); callers skip those.
 */
export function tagSlug(label: string): string {
  return normalizeText(label)
    .replace(/[^\p{L}\p{N}]+/gu, "-")
    .replace(/^-+|-+$/g, "");
}

/** One article's tags as chips: deduplicated by slug, first spelling wins, unusable labels dropped. */
export function tagRefs(labels: readonly string[]): HelpTagRef[] {
  const seen = new Set<string>();
  const refs: HelpTagRef[] = [];
  for (const label of labels) {
    const slug = tagSlug(label);
    if (!slug || seen.has(slug)) continue;
    seen.add(slug);
    refs.push({ slug, label });
  }
  return refs;
}

export type HelpTagGroup = HelpTag & {
  /** Every spelling seen for this slug, most frequent first. More than one is a validation warning. */
  labels: string[];
  /** Articles carrying the tag, in index order. */
  articles: HelpArticle[];
};

/**
 * Groups articles by tag slug. The display label is the most frequent
 * spelling (ties: first seen in index order). Sorted most-used first, then by
 * label in the locale's collation — the order a tag cloud wants.
 */
export function groupTags(articles: readonly HelpArticle[], locale: string): HelpTagGroup[] {
  const collator = new Intl.Collator(locale);
  const groups = new Map<string, { counts: Map<string, number>; articles: HelpArticle[] }>();

  for (const article of articles) {
    for (const { slug, label } of tagRefs(article.tags)) {
      const group = groups.get(slug) ?? { counts: new Map<string, number>(), articles: [] };
      group.counts.set(label, (group.counts.get(label) ?? 0) + 1);
      group.articles.push(article);
      groups.set(slug, group);
    }
  }

  return Array.from(groups, ([slug, { counts, articles: tagged }]) => {
    // Sort is stable: equal counts keep insertion order, so the first spelling seen wins a tie.
    const labels = Array.from(counts)
      .sort((a, b) => b[1] - a[1])
      .map(([label]) => label);
    return { slug, label: labels[0] ?? slug, labels, count: tagged.length, articles: tagged };
  }).sort((a, b) => b.count - a.count || collator.compare(a.label, b.label));
}
```

Client-safe (no `fs`), so `HelpTagChips` imports `tagSlug`/`tagRefs` directly.
`assembleIndex` ([content-loader.md](content-loader.md)) calls `groupTags`
once per index build; pages read `index.tags` and `index.byTag`, never regroup.

## Tag page

```
app/[lang]/help/tag/[tag]/page.tsx     ← folder name = HELP_CONTENT.tagSegment
```

Next.js prefers the static `tag` segment over `[category]`, which is why the
segment must not also be a category id. `/help/tag` on its own falls through to
the category page, is not a category, and 404s — correct.

```tsx
// file: src/app/[lang]/help/tag/[tag]/page.tsx
import type { Metadata } from "next";
import { notFound } from "next/navigation";

import HelpArticleList from "@/components/help/HelpArticleList";
import HelpBreadcrumb from "@/components/help/HelpBreadcrumb";
import HelpShell from "@/components/help/HelpShell";
import { HELP_CONTENT } from "@/lib/help/config";
import { getHelpIndex } from "@/lib/help/loader";
import { helpPath } from "@/lib/help/model";
import { helpPageAlternates } from "@/lib/help/seo";
import { formatMessage, getHelpStrings, plural } from "@/lib/help/strings";

type Props = { params: Promise<{ lang: string; tag: string }> };

// The tag set is known at build time; anything else is a routing-level 404 (see routes.md).
export const dynamicParams = false;

export function generateStaticParams() {
  return HELP_CONTENT.routeLocales.flatMap((lang) =>
    getHelpIndex(lang).tags.map((tag) => ({ lang, tag: tag.slug })),
  );
}

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { lang, tag: slug } = await params;
  const strings = getHelpStrings(lang);
  const tag = getHelpIndex(lang).tags.find((t) => t.slug === slug);
  if (!tag) return { title: strings.notFound };
  return {
    title: `${tag.label} | ${strings.title}`,
    description: formatMessage(strings.tagHeading, { tag: tag.label }),
    alternates: helpPageAlternates(lang, helpPath.tag(slug)),
  };
}

export default async function HelpTagPage({ params }: Props) {
  const { lang, tag: slug } = await params;
  const index = getHelpIndex(lang);
  const strings = getHelpStrings(lang);
  const tag = index.tags.find((t) => t.slug === slug);
  const articles = index.byTag.get(slug);
  if (!tag || !articles) notFound();

  return (
    <HelpShell locale={lang}>
      {/* No category: the trail is Help › <tag label>. */}
      <HelpBreadcrumb locale={lang} strings={strings} articleTitle={tag.label} />
      <header className="mb-10">
        <h1>{formatMessage(strings.tagHeading, { tag: tag.label })}</h1>
        <p>{plural(lang, strings.articleCount, articles.length)}</p>
      </header>
      <HelpArticleList locale={lang} articles={articles} />
    </HelpShell>
  );
}
```

`HelpArticleList` links each row through `article.category`, so a mixed-
category list needs nothing special ([ui-content.md](ui-content.md)).

### Canonical, hreflang, sitemap

A tag page follows the **category-page** rule, not the article rule: with a
single content tree every locale's `/help/tag/x` is the same document and
canonicalises to the default locale; with per-locale trees each locale is its
own page. `helpPageAlternates` encodes that; the sitemap
([routes.md](routes.md)) lists tag pages at priority 0.5 under the same locale
rule. The tag set itself is per locale — it is built from the locale's index,
which includes fallback articles — so a tag never disappears from a locale
because a translation is missing.

## Chips

Two surfaces, one rule: **a chip is a link except inside something that is
already a link.** Article header and landing-page cloud render `HelpTagChips`
(links); `HelpArticleList` rows are links themselves, so their chips stay
spans — nested anchors are invalid HTML and browsers split them
unpredictably.

```tsx
// file: src/components/help/HelpTagChips.tsx
import Link from "next/link";

import type { HelpTagRef } from "@/lib/help/model";
import { helpHref } from "@/lib/help/paths";

type Props = {
  locale: string;
  /** `tagRefs(article.tags)` on an article page; `index.tags` for a cloud. */
  tags: readonly HelpTagRef[];
};

export default function HelpTagChips({ locale, tags }: Props) {
  if (tags.length === 0) return null;
  return (
    <ul className="flex flex-wrap gap-1.5" data-help-tags>
      {tags.map((tag) => (
        <li key={tag.slug}>
          <Link href={helpHref.tag(locale, tag.slug)} data-help-tag>
            {tag.label}
          </Link>
        </li>
      ))}
    </ul>
  );
}
```

On an article page the chip shows *that article's* spelling; the tag page it
opens is titled with the corpus-majority spelling. Validation is where the
two get reconciled, not the renderer.

## Validation

`validateHelpIndex` ([content-model.md](content-model.md)) adds three
warnings, all cheap and all found in a real corpus:

| Warning | Trigger | Why it matters |
|---|---|---|
| `tag "api" is spelled "API", "api"; pick one` | one slug, several spellings | The chips differ; the page title is whichever spelling won the count |
| `tags "integration" and "integrations" both exist; merge them` | `x` and `xs` both present (English plural heuristic) | Two pages for one topic, each missing half the articles |
| `tag "#" has no URL-safe form` | `tagSlug(label) === ""` | The chip renders; nothing links to it |

Warnings, not errors: a spelling drift should not block a deploy, but the
author must see it. Search is unaffected either way — both spellings tokenise
to the same term.

## Behaviour contract

| Situation | Result |
|---|---|
| `tags: ["API"]` on one article, `["api"]` on another | one tag `api`, count 2; label is the majority spelling, ties → first in index order |
| `tags: ["api", "API"]` on the **same** article | one chip, counted once for that article |
| `tags: ["Płatności"]` | slug `platnosci`; `/help/tag/platnosci`; searching `platnosci` or `płatności` finds it |
| `tags: ["#"]` | no chip link, no tag page; validation warning |
| Category id equal to `tagSegment` | `config.ts` throws at import — the route would be shadowed |
| Tag with no remaining articles after a content change | no page generated; inbound links 404 — add a redirect if the URL was shared |

## Tests

```ts
// file: src/lib/help/tags.test.ts
import { describe, expect, it } from "vitest";

import { assembleIndex } from "./loader";
import type { HelpArticle } from "./model";
import { groupTags, tagRefs, tagSlug } from "./tags";
import { validateHelpIndex } from "./validate";

function article(
  slug: string,
  tags: string[],
  category: HelpArticle["category"] = "app",
): HelpArticle {
  return {
    slug,
    category,
    title: slug,
    description: "d",
    tags,
    order: 1,
    related: [],
    updatedAt: null,
    featured: false,
    locale: "en",
    translated: true,
    headings: [],
    issues: [],
    content: "",
  };
}

describe("tagSlug", () => {
  it("lowercases, folds diacritics and hyphenates", () => {
    expect(tagSlug("Booking page")).toBe("booking-page");
    expect(tagSlug("API")).toBe("api");
    expect(tagSlug("Płatności / Stripe")).toBe("platnosci-stripe");
    expect(tagSlug("  C#  ")).toBe("c");
    expect(tagSlug("#")).toBe("");
  });
});

describe("tagRefs", () => {
  it("dedupes by slug and drops unusable labels", () => {
    expect(tagRefs(["API", "api", "#", "Api keys"])).toEqual([
      { slug: "api", label: "API" },
      { slug: "api-keys", label: "Api keys" },
    ]);
  });
});

describe("groupTags", () => {
  const articles = [
    article("a", ["api", "Setup"]),
    article("b", ["API", "setup"]),
    article("c", ["api", "integrations"], "development"),
  ];

  it("merges spellings under one slug and picks the majority label", () => {
    const groups = groupTags(articles, "en");
    expect(groups.map((g) => `${g.slug}:${g.label}:${g.count}`)).toEqual([
      "api:api:3",
      "setup:Setup:2",
      "integrations:integrations:1",
    ]);
    expect(groups[0]?.labels).toEqual(["api", "API"]);
    expect(groups[0]?.articles.map((a) => a.slug)).toEqual(["a", "b", "c"]);
  });

  it("populates the index", () => {
    const index = assembleIndex("en", articles);
    expect(index.tags.map((t) => t.slug)).toEqual(["api", "setup", "integrations"]);
    expect(index.byTag.get("setup")?.map((a) => a.slug)).toEqual(["a", "b"]);
  });
});

describe("validateHelpIndex tag checks", () => {
  it("warns on spelling variants, plural pairs and unusable labels", () => {
    const index = assembleIndex("en", [
      article("a", ["API", "integration", "#"]),
      article("b", ["api", "integrations"]),
    ]);
    const warnings = validateHelpIndex(index)
      .filter((i) => i.level === "warning")
      .map((i) => i.message);
    expect(warnings).toContain('tag "api" is spelled "API", "api"; pick one');
    expect(warnings).toContain('tags "integration" and "integrations" both exist; merge them');
    expect(warnings).toContain('tag "#" has no URL-safe form');
  });
});
```

## Checklist

- [ ] `tags.ts` copied; `assembleIndex` populates `tags` and `byTag`
- [ ] Route folder name equals `HELP_CONTENT.tagSegment`; no category shares it
- [ ] `tagHeading` and `tagsHeading` in every locale's strings
- [ ] Article page and landing render `HelpTagChips`; list rows keep span chips
- [ ] `validate:help` run on the real corpus; spelling variants fixed in content, not code
- [ ] Tag entries present in the sitemap; `/help/tag/<slug>` canonical checked on a non-default locale
