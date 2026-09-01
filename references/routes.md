# Routes

Three pages under a locale segment, all statically rendered from the index.
Content lives in the repository, so a new article is a deploy — there is no
revalidation story to design.

```
app/[lang]/help/
  page.tsx                       landing: category cards + featured articles + tag cloud
  [category]/page.tsx            category listing
  [category]/[slug]/page.tsx     article
  tag/[tag]/page.tsx             articles sharing a tag — in tags.md
```

If the host has no locale segment, drop `[lang]` and pass
`HELP_CONTENT.defaultLocale` where the pages read `lang`.

## Why each page renders its own shell

The natural instinct is a `help/layout.tsx`. It cannot work here: the sidebar
and the header need `activeCategory` and `activeSlug`, and a layout does not
receive its children's params. Each page therefore wraps itself in
`<HelpShell>` ([ui.md](ui.md)), which reads `getHelpIndex(locale)` — cached
per request, so the page and the shell share one parse.

## Static params and unknown slugs

```ts
export const dynamicParams = false;
```

on the category and article pages turns any URL outside `generateStaticParams`
into a routing-level 404 instead of rendering with `notFound()`. Two reasons:

1. The set of valid pages is finite and known at build time. Rendering on
   demand for an unknown slug is wasted compute that also has to be guarded.
2. On Next.js 16.2.x, `notFound()` from inside a page ignored the segment's
   `not-found.tsx` and rendered the unbranded default (vercel/next.js#90837,
   fixed in 16.3 previews). Routing-level 404s go to `global-not-found.tsx`
   (`experimental.globalNotFound`) and stay branded. Check whether the host's
   version still has the bug before relying on `notFound()` for the branded
   page; keep `dynamicParams = false` either way.

The `notFound()` calls in the pages remain as type narrowing and as the
fallback if someone flips `dynamicParams` later.

## SEO helpers

```ts
// file: src/lib/help/seo.ts
import type { Metadata } from "next";

import { HELP_CONTENT } from "./config";
import { translatedLocales } from "./loader";
import { type HelpArticle, helpPath } from "./model";
import { absoluteUrl, localizePath } from "./paths";

type Alternates = NonNullable<Metadata["alternates"]>;

function languageMap(locales: readonly string[], path: string): Record<string, string> {
  const map: Record<string, string> = {};
  for (const locale of locales) map[locale] = localizePath(locale, path);
  map["x-default"] = localizePath(HELP_CONTENT.defaultLocale, path);
  return map;
}

/**
 * Landing and category pages. With a single content tree, `/pl/help` is the
 * same document as `/help` — canonicalise to the default locale and declare
 * no hreflang. With per-locale trees each locale is its own page.
 */
export function helpPageAlternates(locale: string, path: string): Alternates {
  if (!HELP_CONTENT.perLocaleDirectories) {
    return { canonical: localizePath(HELP_CONTENT.defaultLocale, path) };
  }
  return {
    canonical: localizePath(locale, path),
    languages: languageMap(HELP_CONTENT.routeLocales, path),
  };
}

/**
 * An untranslated article page is a duplicate of the default-locale page:
 * canonical points there, and it is excluded from hreflang and the sitemap.
 * A translated one lists only the locales that actually have a file.
 */
export function helpArticleAlternates(locale: string, article: HelpArticle): Alternates {
  const path = helpPath.article(article.category, article.slug);
  if (!article.translated) {
    return { canonical: localizePath(HELP_CONTENT.defaultLocale, path) };
  }
  return {
    canonical: localizePath(locale, path),
    languages: languageMap(translatedLocales(article.category, article.slug), path),
  };
}

export function articleJsonLd(article: HelpArticle): Record<string, unknown> {
  return {
    "@context": "https://schema.org",
    "@type": "TechArticle",
    headline: article.title,
    description: article.description,
    inLanguage: article.locale,
    // Omit rather than emit `dateModified: ""` — an empty date is invalid structured data.
    ...(article.updatedAt ? { dateModified: article.updatedAt } : {}),
    author: { "@type": "Organization", name: HELP_CONTENT.siteName },
    publisher: { "@type": "Organization", name: HELP_CONTENT.siteName, url: HELP_CONTENT.siteUrl },
  };
}

export type BreadcrumbItem = { name: string; href: string | null };

export function breadcrumbJsonLd(items: readonly BreadcrumbItem[]): Record<string, unknown> {
  return {
    "@context": "https://schema.org",
    "@type": "BreadcrumbList",
    itemListElement: items.map((item, index) => ({
      "@type": "ListItem",
      position: index + 1,
      name: item.name,
      ...(item.href ? { item: absoluteUrl(item.href) } : {}),
    })),
  };
}

/** `</script>` inside a title would end the JSON-LD block early; escape `<`. */
export function serializeJsonLd(data: Record<string, unknown>): string {
  return JSON.stringify(data).replace(/</g, "\\u003c");
}
```

## Landing page

```tsx
// file: src/app/[lang]/help/page.tsx
import type { Metadata } from "next";
import Link from "next/link";

import HelpCategoryCard from "@/components/help/HelpCategoryCard";
import HelpShell from "@/components/help/HelpShell";
import HelpTagChips from "@/components/help/HelpTagChips";
import { HELP_CONTENT } from "@/lib/help/config";
import { getFeaturedArticles, getHelpIndex } from "@/lib/help/loader";
import { helpPath } from "@/lib/help/model";
import { helpHref } from "@/lib/help/paths";
import { helpPageAlternates } from "@/lib/help/seo";
import { getHelpStrings } from "@/lib/help/strings";

type Props = { params: Promise<{ lang: string }> };

export function generateStaticParams() {
  return HELP_CONTENT.routeLocales.map((lang) => ({ lang }));
}

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { lang } = await params;
  const strings = getHelpStrings(lang);
  return {
    title: strings.title,
    description: strings.tagline,
    alternates: helpPageAlternates(lang, helpPath.home()),
  };
}

export default async function HelpHomePage({ params }: Props) {
  const { lang } = await params;
  const index = getHelpIndex(lang);
  const strings = getHelpStrings(lang);
  const featured = getFeaturedArticles(index);

  return (
    <HelpShell locale={lang}>
      <header className="mb-12">
        <p>{strings.title}</p>
        <h1>{strings.heading}</h1>
        <p>{strings.tagline}</p>
      </header>

      <section className="mb-16 grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3">
        {index.categories.map((category) => (
          <HelpCategoryCard key={category.id} locale={lang} category={category} strings={strings} />
        ))}
      </section>

      {featured.length > 0 && (
        <section>
          <h2>{strings.featuredHeading}</h2>
          <ul className="grid grid-cols-1 gap-3 sm:grid-cols-2">
            {featured.map((article) => (
              <li key={`${article.category}/${article.slug}`}>
                <Link href={helpHref.article(lang, article.category, article.slug)} data-help-card>
                  {article.title}
                </Link>
              </li>
            ))}
          </ul>
        </section>
      )}

      {index.tags.length > 0 && (
        <section className="mt-16">
          <h2>{strings.tagsHeading}</h2>
          <HelpTagChips locale={lang} tags={index.tags.slice(0, 12)} />
        </section>
      )}
    </HelpShell>
  );
}
```

The tag cloud is the twelve most-used tags ([tags.md](tags.md)); `index.tags`
is already sorted that way.

`getFeaturedArticles` returns the `featured: true` articles, or the first N in
category order when none is flagged. The original labelled that fallback
"Popular articles" — it was the first six files of the first folder. Label it
for what it is (`featuredHeading: "Start here"`) and flag real picks.

## Category page

```tsx
// file: src/app/[lang]/help/[category]/page.tsx
import type { Metadata } from "next";
import { notFound } from "next/navigation";

import HelpArticleList from "@/components/help/HelpArticleList";
import HelpBreadcrumb from "@/components/help/HelpBreadcrumb";
import HelpShell from "@/components/help/HelpShell";
import { HELP_CATEGORY_IDS, HELP_CONTENT, isHelpCategory } from "@/lib/help/config";
import { getHelpIndex } from "@/lib/help/loader";
import { helpPath } from "@/lib/help/model";
import { helpPageAlternates } from "@/lib/help/seo";
import { getHelpStrings, plural } from "@/lib/help/strings";

type Props = { params: Promise<{ lang: string; category: string }> };

// Finite set, prerendered; anything else is a routing-level 404 (see "Static params").
export const dynamicParams = false;

export function generateStaticParams() {
  return HELP_CONTENT.routeLocales.flatMap((lang) =>
    HELP_CATEGORY_IDS.map((category) => ({ lang, category })),
  );
}

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { lang, category } = await params;
  const strings = getHelpStrings(lang);
  if (!isHelpCategory(category)) return { title: strings.notFound };
  const meta = strings.categories[category];
  return {
    title: `${meta.label} | ${strings.title}`,
    description: meta.description,
    alternates: helpPageAlternates(lang, helpPath.category(category)),
  };
}

export default async function HelpCategoryPage({ params }: Props) {
  const { lang, category } = await params;
  if (!isHelpCategory(category)) notFound();

  const index = getHelpIndex(lang);
  const strings = getHelpStrings(lang);
  const articles = index.byCategory[category];
  const meta = strings.categories[category];

  return (
    <HelpShell locale={lang} activeCategory={category}>
      <HelpBreadcrumb locale={lang} strings={strings} category={category} />
      <header className="mb-10">
        <h1>{meta.label}</h1>
        <p>{meta.description}</p>
        <p>{plural(lang, strings.articleCount, articles.length)}</p>
      </header>
      {articles.length > 0 ? (
        <HelpArticleList locale={lang} articles={articles} />
      ) : (
        <p>{strings.comingSoon}</p>
      )}
    </HelpShell>
  );
}
```

## Article page

```tsx
// file: src/app/[lang]/help/[category]/[slug]/page.tsx
import type { Metadata } from "next";
import Link from "next/link";
import { notFound } from "next/navigation";

import HelpBreadcrumb from "@/components/help/HelpBreadcrumb";
import HelpMarkdown from "@/components/help/HelpMarkdown";
import HelpShell from "@/components/help/HelpShell";
import HelpTagChips from "@/components/help/HelpTagChips";
import {
  getHelpArticle,
  getHelpIndex,
  getHelpStaticParams,
  getRelatedArticles,
} from "@/lib/help/loader";
import { helpHref } from "@/lib/help/paths";
import { articleJsonLd, helpArticleAlternates, serializeJsonLd } from "@/lib/help/seo";
import { formatDate, formatMessage, getHelpStrings } from "@/lib/help/strings";
import { tagRefs } from "@/lib/help/tags";

type Props = { params: Promise<{ lang: string; category: string; slug: string }> };

export const dynamicParams = false;

export function generateStaticParams() {
  return getHelpStaticParams();
}

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { lang, category, slug } = await params;
  const strings = getHelpStrings(lang);
  const article = getHelpArticle(lang, category, slug);
  if (!article) return { title: strings.notFound };

  return {
    title: `${article.title} | ${strings.title}`,
    description: article.description,
    alternates: helpArticleAlternates(lang, article),
    openGraph: {
      title: article.title,
      description: article.description,
      type: "article",
      ...(article.updatedAt ? { modifiedTime: article.updatedAt } : {}),
    },
  };
}

export default async function HelpArticlePage({ params }: Props) {
  const { lang, category, slug } = await params;
  const article = getHelpArticle(lang, category, slug);
  if (!article) notFound();

  const index = getHelpIndex(lang);
  const strings = getHelpStrings(lang);
  const related = getRelatedArticles(index, article);
  const categoryLabel = strings.categories[article.category].label;

  return (
    <HelpShell locale={lang} activeCategory={article.category} activeSlug={article.slug}>
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: serializeJsonLd(articleJsonLd(article)) }}
      />
      <HelpBreadcrumb
        locale={lang}
        strings={strings}
        category={article.category}
        articleTitle={article.title}
      />

      <article>
        <header className="mb-10">
          <h1>{article.title}</h1>
          {article.description && <p>{article.description}</p>}
          <div className="mt-4 flex flex-wrap items-center gap-3">
            {article.updatedAt && (
              <span>
                {formatMessage(strings.updated, { date: formatDate(lang, article.updatedAt) })}
              </span>
            )}
            <HelpTagChips locale={lang} tags={tagRefs(article.tags)} />
          </div>
          {!article.translated && (
            <p role="note" data-help-untranslated>
              {strings.notTranslated}
            </p>
          )}
        </header>

        {/* Body language may differ from the page when falling back. */}
        <div lang={article.locale}>
          <HelpMarkdown content={article.content} />
        </div>
      </article>

      {related.length > 0 && (
        <section className="mt-16">
          <h2>{strings.relatedHeading}</h2>
          <ul>
            {related.map((rel) => (
              <li key={`${rel.category}/${rel.slug}`}>
                <Link href={helpHref.article(lang, rel.category, rel.slug)}>{rel.title}</Link>
              </li>
            ))}
          </ul>
        </section>
      )}

      <p className="mt-12">
        <Link href={helpHref.category(lang, article.category)}>
          {formatMessage(strings.backTo, { label: categoryLabel })}
        </Link>
      </p>
    </HelpShell>
  );
}
```

## Sitemap

Merge into the host's `app/sitemap.ts`. Only pages with their own file are
listed: an untranslated locale page carries a canonical elsewhere and listing
it would contradict that.

```ts
// file: src/lib/help/sitemap.ts
import type { MetadataRoute } from "next";

import { HELP_CONTENT } from "./config";
import { getHelpIndex } from "./loader";
import { helpPath } from "./model";
import { absoluteUrl, localizePath } from "./paths";

export function helpSitemapEntries(now: Date = new Date()): MetadataRoute.Sitemap {
  const locales = HELP_CONTENT.perLocaleDirectories
    ? HELP_CONTENT.routeLocales
    : [HELP_CONTENT.defaultLocale];

  return locales.flatMap((locale) => {
    const index = getHelpIndex(locale);
    const url = (path: string): string => absoluteUrl(localizePath(locale, path));
    const entries: MetadataRoute.Sitemap = [
      { url: url(helpPath.home()), lastModified: now, changeFrequency: "weekly", priority: 0.8 },
      ...index.categories.map((category) => ({
        url: url(helpPath.category(category.id)),
        lastModified: now,
        changeFrequency: "weekly" as const,
        priority: 0.7,
      })),
      ...index.tags.map((tag) => ({
        url: url(helpPath.tag(tag.slug)),
        lastModified: now,
        changeFrequency: "weekly" as const,
        priority: 0.5,
      })),
      ...index.articles
        .filter((article) => article.translated)
        .map((article) => ({
          url: url(helpPath.article(article.category, article.slug)),
          lastModified: article.updatedAt ? new Date(article.updatedAt) : now,
          changeFrequency: "monthly" as const,
          priority: 0.6,
        })),
    ];
    return entries;
  });
}
```

## Route surface

| URL | Renders | 404 when |
|---|---|---|
| `/{lang}/help` | landing | `lang` not in `routeLocales` (host proxy/middleware) |
| `/{lang}/help/{category}` | category list | `category` not in `HELP_CATEGORY_IDS` |
| `/{lang}/help/{category}/{slug}` | article | no file in `lang` **or** the default locale |
| `/{lang}/help/tag/{tag}` | every article with the tag, across categories — [tags.md](tags.md) | `tag` not in `index.tags` |

## Checklist

- [ ] Four pages in place; `dynamicParams = false` on category, tag and article
- [ ] Locale middleware/proxy already rewrites `/help` → `/{default}/help` (host)
- [ ] Canonicals verified on a non-default locale URL for both translated and untranslated articles
- [ ] JSON-LD validated (Rich Results test) on one article
- [ ] `helpSitemapEntries()` merged into the host sitemap
- [ ] Any legacy help URLs (an old subdomain, old slugs) redirected in the host config
