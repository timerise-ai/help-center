# Content model

Articles are markdown files in the repository. The filesystem *is* the schema:
the folder is the category, the filename is the slug, the frontmatter is the
metadata. There is no database, no CMS, no admin UI — an article ships the way
code ships, through a pull request.

## Directory layouts

Two layouts, selected by `HELP_CONTENT.perLocaleDirectories`:

```
Single-language tree (perLocaleDirectories: false)
content/help/
  app/                 ← category id
    get-started.md     ← slug
    add-a-location.md
  general/
  development/

Per-locale tree (perLocaleDirectories: true)
content/help/
  en/                  ← default locale: must be complete
    app/get-started.md
    development/webhooks.md
  pl/                  ← may be partial; missing files fall back to `en`
    app/get-started.md
  de/
```

The per-locale tree is the general case; the single tree is the degenerate one.
The loader handles both with the same fallback rule, so start single-language
and flip the flag when the first translation lands — no article moves except
into `<defaultLocale>/`. See [i18n.md](i18n.md) for what fallback means for
URLs, canonicals and the sitemap.

## Config

```ts
// file: src/lib/help/config.ts
/**
 * Category ids double as folder names and URL segments. Order here is display
 * order everywhere (landing cards, sidebar, search tie-breaks).
 */
export const HELP_CATEGORY_IDS = ["app", "general", "development"] as const;
export type HelpCategoryId = (typeof HELP_CATEGORY_IDS)[number];

export function isHelpCategory(value: string): value is HelpCategoryId {
  return (HELP_CATEGORY_IDS as readonly string[]).includes(value);
}

/**
 * A slug is a filename and a URL segment. Anything outside this pattern is
 * skipped by the loader (and reported by validation) rather than joined into a
 * filesystem path.
 */
export const HELP_SLUG_PATTERN = /^[a-z0-9]+(?:-[a-z0-9]+)*$/;

export type HelpContentConfig = {
  /** Root of the markdown tree; relative paths resolve from `process.cwd()`. */
  root: string;
  /** URL prefix of the help center, without locale. */
  basePath: string;
  /** Locales the routes render. Every one gets a page for every article. */
  routeLocales: readonly string[];
  /** Locale whose content tree must be complete; the fallback for all others. */
  defaultLocale: string;
  /** `true` → `<root>/<locale>/<category>/<slug>.md`; `false` → `<root>/<category>/<slug>.md`. */
  perLocaleDirectories: boolean;
  /** Landing page shows `featured: true` articles, or the first N if none is flagged. */
  featuredFallback: number;
  /** Used in JSON-LD and absolute breadcrumb URLs. */
  siteName: string;
  siteUrl: string;
  /**
   * URL segment of tag pages: `<basePath>/<tagSegment>/<slug>`. Must equal the
   * route folder name and must not be a category id — a static segment shadows
   * the dynamic `[category]` one. See tags.md.
   */
  tagSegment: string;
};

export const HELP_CONTENT: HelpContentConfig = {
  root: "content/help",
  basePath: "/help",
  routeLocales: ["en"],
  defaultLocale: "en",
  perLocaleDirectories: false,
  featuredFallback: 6,
  siteName: "Example",
  siteUrl: "https://example.com",
  tagSegment: "tag",
};

if (isHelpCategory(HELP_CONTENT.tagSegment)) {
  throw new Error(`HELP_CONTENT.tagSegment "${HELP_CONTENT.tagSegment}" is also a category id`);
}
```

## Frontmatter

```yaml
---
title: "Webhooks"
description: "Receive an HTTP POST when a booking is created, updated, or cancelled."
order: 4
tags: ["webhooks", "api", "events"]
related: ["api-overview", "general/what-is-api-first"]
updatedAt: "2026-03-01"
featured: false
---

## What are webhooks?
...
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `title` | string | yes | Page `<h1>`, nav label, search field (highest weight) |
| `description` | string | yes | Meta description, list excerpt, search field |
| `order` | integer | recommended | Sort key within the category. Ties break by title, then slug — never by filesystem order |
| `tags` | string[] | no | Free-text labels. Chips linking to `/help/tag/<slug>`, a search field, and the tag index. Identity is the slug — `API` and `api` are one tag, and validation says so. See [tags.md](tags.md) |
| `related` | string[] | no | Bare `slug` resolves in the same category first; `category/slug` is explicit |
| `updatedAt` | `YYYY-MM-DD` | recommended | Shown on the page, `dateModified` in JSON-LD, `lastmod` in the sitemap. Quote it — unquoted YAML dates parse as `Date` objects (the parser normalises them back, but quoting avoids the surprise) |
| `featured` | boolean | no | Landing-page pick. With none flagged, the first `featuredFallback` articles are shown |
| `slug`, `category` | string | no | **Ignored by the loader** — the filename and folder win. If present they must agree, or validation fails the build |

The body starts at `##`. Pages own their `<h1>`; the renderer demotes any `#`
it meets to `<h2>` styling so a stray one cannot produce two top-level headings.

## Slugs

- Lowercase ASCII letters, digits, single hyphens: `add-a-location`, not
  `Add_a_Location` or `add--a-location`.
- **Unique within a category** by construction (they are filenames). Uniqueness
  across categories is not enforced by the filesystem; validation warns when a
  slug appears in two categories because bare `related` references then become
  ambiguous.
- Stable across locales. A translation is the same slug in another locale's
  folder. This is the whole locale model — there is no translation key to keep
  in sync, unlike marketing content where localized slugs matter for SEO.
- Renaming a slug is a URL change: add a redirect and update every `related`
  that pointed at it. Validation catches the second; nothing catches the first.

## Domain types

```ts
// file: src/lib/help/model.ts
import { HELP_CONTENT, type HelpCategoryId } from "./config";

/** What the client ever receives: enough for navigation and search, never the body. */
export type HelpArticleSummary = {
  slug: string;
  category: HelpCategoryId;
  title: string;
  description: string;
  tags: string[];
};

export type HelpArticle = HelpArticleSummary & {
  order: number;
  related: string[];
  updatedAt: string | null;
  featured: boolean;
  /** Locale of the file actually served — the default locale when falling back. */
  locale: string;
  /** False when the requested locale had no file and the default locale's was served. */
  translated: boolean;
  /** `##`/`###` headings, extracted at load time for search. */
  headings: string[];
  /** File-level problems found while loading; surfaced by validation, never at runtime. */
  issues: string[];
  content: string;
};

export type HelpCategory = {
  id: HelpCategoryId;
  order: number;
  articleCount: number;
};

export type HelpSkipped = { category: HelpCategoryId; slug: string; reason: string };

/** A tag as a chip links to it: URL slug plus the spelling to display. */
export type HelpTagRef = { slug: string; label: string };
/** A tag in the index. One per slug; see tags.md for how spellings collapse. */
export type HelpTag = HelpTagRef & { count: number };

export type HelpIndex = {
  locale: string;
  categories: HelpCategory[];
  /** Every article, in category order then `order`/title order. */
  articles: HelpArticle[];
  byCategory: Record<HelpCategoryId, HelpArticle[]>;
  /** Keyed by `articleKey(category, slug)`. */
  byKey: Map<string, HelpArticle>;
  /** Files the loader refused (bad slug, unparsable frontmatter). */
  skipped: HelpSkipped[];
  /** Every tag, most used first; `label` is the most frequent spelling. */
  tags: HelpTag[];
  /** Tag slug → articles carrying it, in index order. */
  byTag: Map<string, HelpArticle[]>;
};

export type HelpSearchDoc = HelpArticleSummary & { headings: string[] };

export function articleKey(category: HelpCategoryId, slug: string): string {
  return `${category}/${slug}`;
}

export function toSummary(article: HelpArticle): HelpArticleSummary {
  const { slug, category, title, description, tags } = article;
  return { slug, category, title, description, tags };
}

export function toSearchDoc(article: HelpArticle): HelpSearchDoc {
  return { ...toSummary(article), headings: article.headings };
}

/** Locale-free paths; wrap with `localizePath` from paths.ts for links. */
export const helpPath = {
  home: (): string => HELP_CONTENT.basePath,
  category: (category: HelpCategoryId): string => `${HELP_CONTENT.basePath}/${category}`,
  article: (category: HelpCategoryId, slug: string): string =>
    `${HELP_CONTENT.basePath}/${category}/${slug}`,
  tag: (slug: string): string => `${HELP_CONTENT.basePath}/${HELP_CONTENT.tagSegment}/${slug}`,
};
```

`toSummary` exists because the layout passes the whole corpus to client
components for navigation and search. Passing `HelpArticle` would serialise
every article body into every page's RSC payload, hundreds of kilobytes on
each request for a few dozen articles. Strip before it crosses the boundary,
always.

## Validation

Nothing in the runtime path warns. A `related` entry that resolves to nothing
is silently dropped; a missing `order` sorts last; a typo in `updatedAt` renders
"Invalid Date". Each of these is fine for the reader and invisible to the
author — so run validation in CI and fail the build.

```ts
// file: src/lib/help/validate.ts
import type { HelpCategoryId } from "./config";
import { resolveRef } from "./loader";
import type { HelpIndex } from "./model";
import { groupTags, tagSlug } from "./tags";

export type HelpIssueLevel = "error" | "warning";

export type HelpContentIssue = {
  level: HelpIssueLevel;
  locale: string;
  category: HelpCategoryId | null;
  slug: string | null;
  message: string;
};

export function validateHelpIndex(index: HelpIndex): HelpContentIssue[] {
  const issues: HelpContentIssue[] = [];
  const push = (
    level: HelpIssueLevel,
    category: HelpCategoryId | null,
    slug: string | null,
    message: string,
  ): void => {
    issues.push({ level, locale: index.locale, category, slug, message });
  };

  for (const s of index.skipped) push("error", s.category, s.slug, `skipped: ${s.reason}`);

  // Slugs present in more than one category make bare `related` refs ambiguous.
  const categoriesBySlug = new Map<string, HelpCategoryId[]>();
  for (const a of index.articles) {
    categoriesBySlug.set(a.slug, [...(categoriesBySlug.get(a.slug) ?? []), a.category]);
  }
  for (const [slug, cats] of categoriesBySlug) {
    if (cats.length > 1) push("warning", null, slug, `slug exists in ${cats.join(", ")}`);
  }

  for (const category of index.categories) {
    const articles = index.byCategory[category.id];
    const seenOrder = new Map<number, string>();
    for (const a of articles) {
      for (const issue of a.issues) push("error", a.category, a.slug, issue);
      if (!a.title) push("error", a.category, a.slug, "missing title");
      if (!a.description) push("warning", a.category, a.slug, "missing description");
      if (a.order === Number.MAX_SAFE_INTEGER) {
        push("warning", a.category, a.slug, "no order: sorts last, by title");
      } else {
        const other = seenOrder.get(a.order);
        if (other) push("warning", a.category, a.slug, `order ${a.order} also used by ${other}`);
        seenOrder.set(a.order, a.slug);
      }
      for (const ref of a.related) {
        const target = resolveRef(index, a.category, ref);
        if (!target) {
          push("error", a.category, a.slug, `related "${ref}" does not resolve`);
        } else if (target === a) {
          push("warning", a.category, a.slug, `related "${ref}" points at itself`);
        } else if (
          !ref.includes("/") &&
          target.category !== a.category &&
          (categoriesBySlug.get(ref)?.length ?? 0) > 1
        ) {
          push("warning", a.category, a.slug, `related "${ref}" is ambiguous; qualify it`);
        }
      }
      for (const label of a.tags) {
        if (!tagSlug(label)) push("warning", a.category, a.slug, `tag "${label}" has no URL-safe form`);
      }
    }
  }

  // Tags are keyed by slug (tags.md). Spelling variants of one slug, and
  // singular/plural pairs, each split one topic across two chips or pages.
  for (const group of groupTags(index.articles, index.locale)) {
    if (group.labels.length > 1) {
      const spellings = group.labels.map((label) => `"${label}"`).join(", ");
      push("warning", null, null, `tag "${group.slug}" is spelled ${spellings}; pick one`);
    }
  }
  const tagSlugs = new Set(index.tags.map((tag) => tag.slug));
  for (const slug of tagSlugs) {
    // English plural heuristic: cheap, and it is what the audited corpus had.
    if (tagSlugs.has(`${slug}s`)) {
      push("warning", null, null, `tags "${slug}" and "${slug}s" both exist; merge them`);
    }
  }

  return issues;
}
```

The loader populates `article.issues` with file-level problems (see
[content-loader.md](content-loader.md)): `slug`/`category` frontmatter
disagreeing with the path, an `updatedAt` that is not `YYYY-MM-DD`. The tag
checks and their tests are explained in [tags.md](tags.md).

### The script

```ts
// file: scripts/validate-help.ts
import { HELP_CONTENT } from "@/lib/help/config";
import { getHelpIndex } from "@/lib/help/loader";
import { validateHelpIndex } from "@/lib/help/validate";

const locales = HELP_CONTENT.perLocaleDirectories
  ? HELP_CONTENT.routeLocales
  : [HELP_CONTENT.defaultLocale];

const issues = locales.flatMap((locale) => validateHelpIndex(getHelpIndex(locale)));

for (const issue of issues) {
  const where = [issue.locale, issue.category, issue.slug].filter(Boolean).join("/");
  const line = `${issue.level.toUpperCase()} ${where}: ${issue.message}`;
  if (issue.level === "error") console.error(line);
  else console.warn(line);
}

const errors = issues.filter((i) => i.level === "error").length;
console.log(`help content: ${issues.length} issue(s), ${errors} error(s)`);
if (errors > 0) process.exit(1);
```

Wire it as `"validate:help": "bun scripts/validate-help.ts"` (or `tsx`) and
run it before `build`. Warnings are informational; errors fail. Run against a
real corpus this reported order ties and a batch of `updatedAt` values that had
clearly never been updated since import — both invisible on the site.

### Tests for the validator

```ts
// file: src/lib/help/validate.test.ts
import { describe, expect, it } from "vitest";

import { assembleIndex } from "./loader";
import type { HelpArticle } from "./model";
import { validateHelpIndex } from "./validate";

function article(overrides: Partial<HelpArticle> & Pick<HelpArticle, "slug" | "category">): HelpArticle {
  return {
    title: overrides.slug,
    description: "d",
    tags: [],
    order: 1,
    related: [],
    updatedAt: "2026-01-01",
    featured: false,
    locale: "en",
    translated: true,
    headings: [],
    issues: [],
    content: "",
    ...overrides,
  };
}

describe("validateHelpIndex", () => {
  it("reports unresolved and self related refs", () => {
    const index = assembleIndex("en", [
      article({ slug: "a", category: "app", related: ["missing", "a"] }),
    ]);
    const messages = validateHelpIndex(index).map((i) => `${i.level}:${i.message}`);
    expect(messages).toContain('error:related "missing" does not resolve');
    expect(messages).toContain('warning:related "a" points at itself');
  });

  it("warns on order ties and ambiguous bare refs", () => {
    const index = assembleIndex("en", [
      article({ slug: "setup", category: "app", order: 2 }),
      article({ slug: "other", category: "app", order: 2 }),
      article({ slug: "setup", category: "general", order: 1 }),
      article({ slug: "dev", category: "development", related: ["setup"] }),
    ]);
    const warnings = validateHelpIndex(index).filter((i) => i.level === "warning");
    expect(warnings.some((w) => w.message.startsWith("order 2 also used by"))).toBe(true);
    expect(warnings.some((w) => w.message.includes("is ambiguous"))).toBe(true);
    expect(warnings.some((w) => w.message === "slug exists in app, general")).toBe(true);
  });

  it("promotes loader issues and skipped files to errors", () => {
    const index = assembleIndex(
      "en",
      [article({ slug: "a", category: "app", issues: ["updatedAt is not YYYY-MM-DD"] })],
      [{ category: "app", slug: "Bad Name", reason: "invalid slug" }],
    );
    const errors = validateHelpIndex(index).filter((i) => i.level === "error");
    expect(errors.map((e) => e.message)).toEqual([
      "skipped: invalid slug",
      "updatedAt is not YYYY-MM-DD",
    ]);
  });
});
```

## Authoring rules worth writing down for contributors

- One article, one task. "Add a location" and "Location working hours" are two
  articles even though one screen does both; search and related links reward
  small units.
- Description is the sentence under the title in every list and the meta
  description. Write it as the answer to "what will I be able to do".
- `updatedAt` means "content last changed", not "file last touched". If it is
  going to be wrong, leave it out — the page hides the line rather than showing
  a stale date. Deriving it from git is an extension
  ([extensions.md](extensions.md)).
- Prefer `related` over inline cross-links: it survives renames (validation
  catches it) and renders consistently.

## Checklist

- [ ] Category ids chosen, ordered, and given labels in the strings table
- [ ] Layout chosen (`perLocaleDirectories`) and `defaultLocale` tree complete
- [ ] Every article has `title`, `description`, `order`; `updatedAt` quoted
- [ ] `validate:help` script wired into CI before `build`
- [ ] Validator tests copied and passing
