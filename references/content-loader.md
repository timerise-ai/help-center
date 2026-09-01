# Content loader

Reads the markdown tree into one immutable `HelpIndex` per locale, once per
request. Every page, the layout, metadata, the sitemap and the static-params
generator read from that index instead of touching the filesystem themselves.

Server-only: it imports `node:fs`. Never import it from a `"use client"` file —
the client gets `HelpArticleSummary[]` and `HelpSearchDoc[]` as props.

## Why an index

The original loader had six exported functions that each walked the tree. A
single article page called them from the layout, the page, `generateMetadata`
and the related-articles resolver — parsing all 64 files four times per render,
and once more per locale in `generateStaticParams`. Correct, just wasteful, and
the waste grows with the corpus. `getHelpIndex(locale)` is wrapped in React's
`cache()`, so within one request every caller shares one parse; between
requests nothing is retained, so content edits show up in dev without a restart.

## Frontmatter parsing

```ts
// file: src/lib/help/frontmatter.ts
import matter from "gray-matter";

export type Frontmatter = Record<string, unknown>;

// YAML turns an unquoted `updatedAt: 2026-02-09` into a Date. Every consumer
// wants the string back — keep the time only when the source carried one.
function normalizeValue(value: unknown): unknown {
  if (value instanceof Date) {
    const iso = value.toISOString();
    return iso.endsWith("T00:00:00.000Z") ? iso.slice(0, 10) : iso;
  }
  if (Array.isArray(value)) return value.map(normalizeValue);
  return value;
}

/**
 * Throws with the file name on invalid YAML. A file the parser cannot read
 * should fail validation loudly, not degrade into a half-read article.
 */
export function parseFrontmatter(
  source: string,
  fileName: string,
): { data: Frontmatter; content: string } {
  let parsed: ReturnType<typeof matter>;
  try {
    parsed = matter(source);
  } catch (error) {
    const reason = error instanceof Error ? error.message : String(error);
    throw new Error(`Invalid frontmatter in ${fileName}: ${reason}`);
  }
  const data: Frontmatter = {};
  for (const [key, value] of Object.entries(parsed.data)) data[key] = normalizeValue(value);
  return { data, content: parsed.content };
}

export function str(data: Frontmatter, key: string): string | null {
  const value = data[key];
  return typeof value === "string" && value.trim() !== "" ? value.trim() : null;
}

export function num(data: Frontmatter, key: string): number | null {
  const value = data[key];
  if (typeof value === "number" && Number.isFinite(value)) return value;
  if (typeof value === "string" && value.trim() !== "" && !Number.isNaN(Number(value))) {
    return Number(value);
  }
  return null;
}

export function bool(data: Frontmatter, key: string): boolean {
  const value = data[key];
  return value === true || value === "true";
}

export function strList(data: Frontmatter, key: string): string[] {
  const value = data[key];
  if (!Array.isArray(value)) return [];
  return value.filter((v): v is string => typeof v === "string" && v.trim() !== "");
}
```

`num` accepts `"3"` as well as `3` because a hand-written line parser (which
the original kept as a fallback) yields strings. `order: 0` is a valid order;
the original's `|| 999` sent it to the end of the list.

## The loader

```ts
// file: src/lib/help/loader.ts
import fs from "node:fs";
import path from "node:path";
import { cache } from "react";

import {
  HELP_CATEGORY_IDS,
  HELP_CONTENT,
  HELP_SLUG_PATTERN,
  isHelpCategory,
  type HelpCategoryId,
} from "./config";
import { bool, num, parseFrontmatter, str, strList } from "./frontmatter";
import {
  articleKey,
  type HelpArticle,
  type HelpCategory,
  type HelpIndex,
  type HelpSkipped,
} from "./model";
import { groupTags } from "./tags";

const DATE_ONLY = /^\d{4}-\d{2}-\d{2}$/;

function localeRoot(locale: string): string {
  const root = path.isAbsolute(HELP_CONTENT.root)
    ? HELP_CONTENT.root
    : path.join(process.cwd(), HELP_CONTENT.root);
  return HELP_CONTENT.perLocaleDirectories ? path.join(root, locale) : root;
}

function categoryDir(locale: string, category: HelpCategoryId): string {
  return path.join(localeRoot(locale), category);
}

function listFileSlugs(dir: string): string[] {
  if (!fs.existsSync(dir)) return [];
  return fs
    .readdirSync(dir)
    .filter((name) => name.endsWith(".md"))
    .map((name) => name.slice(0, -".md".length));
}

/** Locale directories to try, most specific first. */
function candidateLocales(locale: string): string[] {
  if (!HELP_CONTENT.perLocaleDirectories || locale === HELP_CONTENT.defaultLocale) {
    return [HELP_CONTENT.defaultLocale];
  }
  return [locale, HELP_CONTENT.defaultLocale];
}

function resolveFile(
  locale: string,
  category: HelpCategoryId,
  slug: string,
): { file: string; locale: string } | null {
  for (const candidate of candidateLocales(locale)) {
    const file = path.join(categoryDir(candidate, category), `${slug}.md`);
    if (fs.existsSync(file)) return { file, locale: candidate };
  }
  return null;
}

// Written as `{3}` so the literal never closes a markdown fence that quotes this file.
const FENCED_CODE = /`{3}[\s\S]*?`{3}/g;
const HEADING = /^#{1,3}[ \t]+(.+?)[ \t]*#*[ \t]*$/gm;

/** `#`–`###` headings outside code fences — the cheap half of full-text search. */
export function extractHeadings(markdown: string): string[] {
  const body = markdown.replace(FENCED_CODE, "");
  return Array.from(body.matchAll(HEADING), (m) => (m[1] ?? "").trim()).filter(Boolean);
}

function readArticle(
  requestedLocale: string,
  category: HelpCategoryId,
  slug: string,
): HelpArticle | null {
  const hit = resolveFile(requestedLocale, category, slug);
  if (!hit) return null;

  const { data, content } = parseFrontmatter(fs.readFileSync(hit.file, "utf8"), hit.file);
  const issues: string[] = [];

  const declaredSlug = str(data, "slug");
  if (declaredSlug && declaredSlug !== slug) {
    issues.push(`frontmatter slug "${declaredSlug}" differs from filename "${slug}"`);
  }
  const declaredCategory = str(data, "category");
  if (declaredCategory && declaredCategory !== category) {
    issues.push(`frontmatter category "${declaredCategory}" differs from folder "${category}"`);
  }
  const updatedAt = str(data, "updatedAt");
  if (updatedAt && !DATE_ONLY.test(updatedAt)) issues.push("updatedAt is not YYYY-MM-DD");

  return {
    slug,
    category,
    title: str(data, "title") ?? "",
    description: str(data, "description") ?? "",
    tags: strList(data, "tags"),
    // Missing order sorts last; validation warns. `?? ` keeps `order: 0` valid.
    order: num(data, "order") ?? Number.MAX_SAFE_INTEGER,
    related: strList(data, "related"),
    updatedAt: updatedAt && DATE_ONLY.test(updatedAt) ? updatedAt : null,
    featured: bool(data, "featured"),
    locale: hit.locale,
    translated: hit.locale === requestedLocale,
    headings: extractHeadings(content),
    issues,
    content,
  };
}

/**
 * Pure assembly step, exported so tests can build an index from in-memory
 * articles. Sorting is deterministic: `order`, then title, then slug —
 * never `readdir` order, which differs between filesystems.
 */
export function assembleIndex(
  locale: string,
  articles: HelpArticle[],
  skipped: HelpSkipped[] = [],
): HelpIndex {
  const collator = new Intl.Collator(locale);
  const byCategory = Object.fromEntries(
    HELP_CATEGORY_IDS.map((id) => [id, [] as HelpArticle[]]),
  ) as Record<HelpCategoryId, HelpArticle[]>;

  for (const article of articles) byCategory[article.category].push(article);
  for (const list of Object.values(byCategory)) {
    list.sort(
      (a, b) =>
        a.order - b.order ||
        collator.compare(a.title, b.title) ||
        collator.compare(a.slug, b.slug),
    );
  }

  const ordered = HELP_CATEGORY_IDS.flatMap((id) => byCategory[id]);
  const categories: HelpCategory[] = HELP_CATEGORY_IDS.map((id, order) => ({
    id,
    order,
    articleCount: byCategory[id].length,
  }));
  const byKey = new Map(ordered.map((a) => [articleKey(a.category, a.slug), a]));
  // Tags are grouped once here; pages read `tags` / `byTag`, never regroup (tags.md).
  const tagGroups = groupTags(ordered, locale);
  const tags = tagGroups.map(({ slug, label, count }) => ({ slug, label, count }));
  const byTag = new Map(tagGroups.map((group) => [group.slug, group.articles]));

  return { locale, categories, articles: ordered, byCategory, byKey, skipped, tags, byTag };
}

function buildIndex(locale: string): HelpIndex {
  const articles: HelpArticle[] = [];
  const skipped: HelpSkipped[] = [];

  for (const category of HELP_CATEGORY_IDS) {
    // Union of the requested locale and the default locale, so an untranslated
    // article still appears in navigation (served from the fallback).
    const slugs = new Set<string>();
    for (const candidate of candidateLocales(locale)) {
      for (const slug of listFileSlugs(categoryDir(candidate, category))) slugs.add(slug);
    }
    for (const slug of slugs) {
      if (!HELP_SLUG_PATTERN.test(slug)) {
        skipped.push({ category, slug, reason: "invalid slug" });
        continue;
      }
      try {
        const article = readArticle(locale, category, slug);
        if (article) articles.push(article);
      } catch (error) {
        skipped.push({ category, slug, reason: error instanceof Error ? error.message : "unreadable" });
      }
    }
  }

  return assembleIndex(locale, articles, skipped);
}

/** One parse per request per locale. Safe to call from layout, page, metadata and sitemap alike. */
export const getHelpIndex = cache(buildIndex);

export function getHelpArticle(locale: string, category: string, slug: string): HelpArticle | null {
  if (!isHelpCategory(category)) return null;
  return getHelpIndex(locale).byKey.get(articleKey(category, slug)) ?? null;
}

/** `slug` resolves in the caller's category first, then anywhere; `category/slug` is explicit. */
export function resolveRef(
  index: HelpIndex,
  fromCategory: HelpCategoryId,
  ref: string,
): HelpArticle | null {
  if (ref.includes("/")) return index.byKey.get(ref) ?? null;
  return (
    index.byKey.get(articleKey(fromCategory, ref)) ??
    index.articles.find((a) => a.slug === ref) ??
    null
  );
}

export function getRelatedArticles(index: HelpIndex, article: HelpArticle): HelpArticle[] {
  const related: HelpArticle[] = [];
  for (const ref of article.related) {
    const target = resolveRef(index, article.category, ref);
    if (target && target !== article && !related.includes(target)) related.push(target);
  }
  return related;
}

export function getFeaturedArticles(
  index: HelpIndex,
  limit: number = HELP_CONTENT.featuredFallback,
): HelpArticle[] {
  const flagged = index.articles.filter((a) => a.featured);
  return (flagged.length > 0 ? flagged : index.articles).slice(0, limit);
}

/** Every (locale, category, slug) the routes should prerender. */
export function getHelpStaticParams(): { lang: string; category: HelpCategoryId; slug: string }[] {
  return HELP_CONTENT.routeLocales.flatMap((lang) =>
    getHelpIndex(lang).articles.map((a) => ({ lang, category: a.category, slug: a.slug })),
  );
}

/** Locales in which this article has its own file (not the fallback). */
export function translatedLocales(category: HelpCategoryId, slug: string): string[] {
  return HELP_CONTENT.routeLocales.filter(
    (locale) => getHelpArticle(locale, category, slug)?.translated === true,
  );
}
```

## Behaviour contract

| Situation | Result |
|---|---|
| `pl` requested, `pl/app/x.md` exists | served from `pl`, `translated: true` |
| `pl` requested, only `en/app/x.md` exists | served from `en`, `translated: false`, still listed in `pl` navigation |
| Single tree, any non-default locale | every article `translated: false` — the page is a duplicate of the default-locale URL and its canonical says so |
| File `Bad_Name.md` | not loaded; `skipped` with `invalid slug`; validation error |
| Invalid YAML | not loaded; `skipped` with the parser message and file path |
| Two articles with `order: 5` | stable order by title; validation warning |
| `related: ["x"]` where `x` is in another category only | resolves globally; warning if `x` exists in two categories |
| `related: ["x"]` where `x` does not exist | dropped at runtime; validation **error** |
| `tags: ["API"]` here, `tags: ["api"]` there | one tag `api` in `index.tags`, both articles in `byTag`; validation warning ([tags.md](tags.md)) |
| `slug` in URL fails `HELP_SLUG_PATTERN` | never reaches `path.join` — the index has no such key |

## Tests

These run against real files in a temp directory; they exercise fallback,
skipping, heading extraction and sort order. `HELP_CONTENT` is a plain mutable
object precisely so a test can point it at a fixture tree.

```ts
// file: src/lib/help/loader.test.ts
import fs from "node:fs";
import os from "node:os";
import path from "node:path";

import { afterAll, beforeAll, describe, expect, it } from "vitest";

import { HELP_CONTENT } from "./config";
import { extractHeadings, getHelpIndex, getRelatedArticles } from "./loader";

const original = { ...HELP_CONTENT };
let root = "";

function write(file: string, body: string): void {
  const full = path.join(root, file);
  fs.mkdirSync(path.dirname(full), { recursive: true });
  fs.writeFileSync(full, body);
}

beforeAll(() => {
  root = fs.mkdtempSync(path.join(os.tmpdir(), "help-"));
  Object.assign(HELP_CONTENT, {
    root,
    perLocaleDirectories: true,
    routeLocales: ["en", "pl"],
    defaultLocale: "en",
  });
  write(
    "en/app/get-started.md",
    `---\ntitle: "Get started"\ndescription: "First steps"\norder: 1\nrelated: ["add-a-location", "development/webhooks", "nowhere"]\nupdatedAt: "2026-01-02"\n---\n\n## Create a service\n\n\`\`\`md\n## not a heading\n\`\`\`\n\n### Pick a slot\n`,
  );
  write("en/app/add-a-location.md", `---\ntitle: "Add a location"\norder: 2\n---\n`);
  write("en/app/b-tie.md", `---\ntitle: "B tie"\norder: 2\n---\n`);
  write("en/app/Bad_Name.md", `---\ntitle: "Bad"\n---\n`);
  write("en/development/webhooks.md", `---\ntitle: "Webhooks"\norder: 1\n---\n`);
  write("pl/app/get-started.md", `---\ntitle: "Pierwsze kroki"\norder: 1\n---\n`);
});

afterAll(() => {
  Object.assign(HELP_CONTENT, original);
  fs.rmSync(root, { recursive: true, force: true });
});

describe("getHelpIndex", () => {
  it("falls back to the default locale and flags it", () => {
    const pl = getHelpIndex("pl");
    const started = pl.byKey.get("app/get-started");
    const location = pl.byKey.get("app/add-a-location");
    expect(started?.title).toBe("Pierwsze kroki");
    expect(started?.translated).toBe(true);
    expect(location?.title).toBe("Add a location");
    expect(location?.translated).toBe(false);
    expect(location?.locale).toBe("en");
  });

  it("skips invalid slugs instead of loading them", () => {
    const en = getHelpIndex("en");
    expect(en.byKey.has("app/Bad_Name")).toBe(false);
    expect(en.skipped).toEqual([{ category: "app", slug: "Bad_Name", reason: "invalid slug" }]);
  });

  it("orders by `order` then title, never by filename", () => {
    const slugs = getHelpIndex("en").byCategory.app.map((a) => a.slug);
    expect(slugs).toEqual(["get-started", "add-a-location", "b-tie"]);
  });

  it("resolves related refs in-category, cross-category, and drops unknown ones", () => {
    const en = getHelpIndex("en");
    const started = en.byKey.get("app/get-started");
    expect(started).toBeDefined();
    if (!started) return;
    expect(getRelatedArticles(en, started).map((a) => a.slug)).toEqual([
      "add-a-location",
      "webhooks",
    ]);
  });

  it("extracts headings outside code fences", () => {
    const started = getHelpIndex("en").byKey.get("app/get-started");
    expect(started?.headings).toEqual(["Create a service", "Pick a slot"]);
  });
});

describe("extractHeadings", () => {
  it("handles closing hashes and h1", () => {
    expect(extractHeadings("# Top ##\n\n#### too deep\n## Second")).toEqual(["Top", "Second"]);
  });
});
```

Run with `vitest run` or `bun test` — Bun aliases `vitest` imports to its own
runner, so the same file works under both.

## Checklist

- [ ] `frontmatter.ts`, `loader.ts` copied; `gray-matter` present in the host
- [ ] Every page/metadata/sitemap call site reads from `getHelpIndex` — no direct `fs` elsewhere
- [ ] Nothing under `"use client"` imports `loader.ts`
- [ ] Loader tests passing against a fixture tree
