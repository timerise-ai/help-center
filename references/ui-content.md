# UI — content components

Breadcrumb, cards, lists and the markdown renderer: the pieces the three pages
compose inside [HelpShell](ui.md). Same rule as the shell — structure and
semantics only; every visual state is a `data-help-*` or ARIA hook the host
styles (table at the end).

## Breadcrumb, cards, lists

```tsx
// file: src/components/help/HelpBreadcrumb.tsx
import Link from "next/link";

import type { HelpCategoryId } from "@/lib/help/config";
import { helpHref } from "@/lib/help/paths";
import { type BreadcrumbItem, breadcrumbJsonLd, serializeJsonLd } from "@/lib/help/seo";
import type { HelpStrings } from "@/lib/help/strings";

type Props = {
  locale: string;
  strings: Pick<HelpStrings, "title" | "breadcrumbs" | "categories">;
  category?: HelpCategoryId;
  articleTitle?: string;
};

export default function HelpBreadcrumb({ locale, strings, category, articleTitle }: Props) {
  const items: BreadcrumbItem[] = [{ name: strings.title, href: helpHref.home(locale) }];
  if (category) {
    items.push({ name: strings.categories[category].label, href: helpHref.category(locale, category) });
  }
  if (articleTitle) items.push({ name: articleTitle, href: null });

  return (
    <>
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: serializeJsonLd(breadcrumbJsonLd(items)) }}
      />
      <nav aria-label={strings.breadcrumbs} className="mb-8">
        <ol className="flex flex-wrap items-center gap-1">
          {items.map((item, index) => (
            <li key={item.name} className="flex items-center gap-1">
              {index > 0 && <span aria-hidden="true">/</span>}
              {item.href ? (
                <Link href={item.href}>{item.name}</Link>
              ) : (
                <span aria-current="page">{item.name}</span>
              )}
            </li>
          ))}
        </ol>
      </nav>
    </>
  );
}
```

```tsx
// file: src/components/help/HelpCategoryCard.tsx
import Link from "next/link";

import type { HelpCategory } from "@/lib/help/model";
import { helpHref } from "@/lib/help/paths";
import { type HelpStrings, plural } from "@/lib/help/strings";

type Props = {
  locale: string;
  category: HelpCategory;
  strings: Pick<HelpStrings, "categories" | "articleCount">;
};

export default function HelpCategoryCard({ locale, category, strings }: Props) {
  const meta = strings.categories[category.id];
  return (
    <Link
      href={helpHref.category(locale, category.id)}
      className="block"
      data-help-card
      data-category={category.id}
    >
      <div className="flex items-start gap-4">
        <span aria-hidden="true" data-help-icon={category.id} />
        <div className="min-w-0 flex-1">
          <div className="flex items-baseline justify-between gap-2">
            <h2>{meta.label}</h2>
            <span>{plural(locale, strings.articleCount, category.articleCount)}</span>
          </div>
          <p>{meta.description}</p>
        </div>
      </div>
    </Link>
  );
}
```

```tsx
// file: src/components/help/HelpArticleList.tsx
import Link from "next/link";

import type { HelpArticleSummary } from "@/lib/help/model";
import { helpHref } from "@/lib/help/paths";
import { tagRefs } from "@/lib/help/tags";

type Props = {
  locale: string;
  /** One category's articles, or one tag's across categories — each row links by its own category. */
  articles: readonly HelpArticleSummary[];
};

export default function HelpArticleList({ locale, articles }: Props) {
  return (
    <ol className="flex flex-col gap-2">
      {articles.map((article, index) => (
        <li key={`${article.category}/${article.slug}`}>
          <Link
            href={helpHref.article(locale, article.category, article.slug)}
            className="flex items-start gap-4"
            data-help-card
          >
            <span aria-hidden="true">{index + 1}</span>
            <span className="min-w-0 flex-1">
              <span className="block">{article.title}</span>
              {article.description && <span className="block">{article.description}</span>}
              {article.tags.length > 0 && (
                // Spans, not links: the whole row is already an anchor.
                <span className="mt-2 flex flex-wrap gap-1.5">
                  {tagRefs(article.tags).map((tag) => (
                    <span key={tag.slug} data-help-tag>
                      {tag.label}
                    </span>
                  ))}
                </span>
              )}
            </span>
          </Link>
        </li>
      ))}
    </ol>
  );
}
```

Linked chips — the article header and the landing-page cloud — are
`HelpTagChips` in [tags.md](tags.md).

## Markdown rendering (seam)

The host almost certainly renders markdown already (legal pages, marketing). Reuse
that renderer. If it has none, this is the minimum:

```tsx
// file: src/components/help/HelpMarkdown.tsx
import ReactMarkdown, { type Components } from "react-markdown";
import remarkGfm from "remark-gfm";

const components: Components = {
  // Pages own their <h1>. Authors paste one in anyway; demote it so the
  // document never has two top-level headings.
  h1: ({ children }) => <h2>{children}</h2>,
  // Wide tables scroll inside the article instead of widening the page.
  table: ({ children }) => (
    <div className="overflow-x-auto">
      <table>{children}</table>
    </div>
  ),
};

export default function HelpMarkdown({ content }: { content: string }) {
  return (
    <div data-help-prose>
      <ReactMarkdown remarkPlugins={[remarkGfm]} components={components}>
        {content}
      </ReactMarkdown>
    </div>
  );
}
```

`react-markdown` without `rehype-raw` escapes inline HTML, so the renderer is
safe even if content ever stops coming from the repo. Style `[data-help-prose]`
with the host's typography (`@tailwindcss/typography`'s `prose`, or the
existing article styles).

## States the host must style

| Hook | Meaning |
|---|---|
| `[data-help-nav] a[aria-current=page]` | active article / home |
| `[data-help-chevron][data-open]` | category expanded |
| `[role=option][data-active]` | keyboard-highlighted search result |
| `[data-help-search-results]` | result popover |
| `[data-help-card]` | category card, article row, featured link |
| `[data-help-tag]` | tag chip — an `<a>` in `HelpTagChips`, a `<span>` inside list rows |
| `[data-help-tags]` | chip list |
| `[data-help-untranslated]` | fallback-language notice |
| `[data-help-icon=search\|menu\|close\|<category>]` | icon slots |

Hooks emitted by the shell, header, navigation tree and drawer
([ui.md](ui.md)) and by search ([search.md](search.md)) are included above.

## Checklist

- [ ] Existing host markdown renderer reused, or `HelpMarkdown` styled via `[data-help-prose]`
- [ ] Every hook in the table styled; `[data-help-card]` matches the host's card surface
- [ ] Breadcrumb JSON-LD validated on one article page
- [ ] Icon slots (`data-help-icon`) replaced with the host's icon set
