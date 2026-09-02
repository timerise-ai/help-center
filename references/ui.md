# UI

Structure, states and semantics. **No colours, radii, shadows or typography** —
those come from the host. Every component emits `data-help-*` attributes and
ARIA state (`aria-current`, `aria-expanded`, `data-active`) so the host attaches
its styles to selectors rather than editing the markup. Layout utilities that
survive any design system (`flex`, `grid`, `gap`, `sticky`, `hidden lg:block`)
are kept; everything decorative was stripped.

```
HelpShell (server)                 ← reads the index, builds nav + search data
├── HelpHeader (client)
│   ├── brand slot · title
│   ├── HelpSearch (desktop)       ← search.md
│   ├── actions slot
│   ├── mobile: search toggle, menu button, HelpSearch (collapsed)
│   └── HelpDrawer → HelpNavTree
├── <aside> sticky, scrollable → HelpNavTree
└── <main> {children}
    ├── HelpBreadcrumb
    ├── HelpCategoryCard / HelpArticleList
    └── HelpMarkdown
```

## Shell

```tsx
// file: src/components/help/HelpShell.tsx
import type { ReactNode } from "react";

import type { HelpCategoryId } from "@/lib/help/config";
import { getHelpIndex } from "@/lib/help/loader";
import { type HelpArticleSummary, toSearchDoc, toSummary } from "@/lib/help/model";
import { getHelpStrings } from "@/lib/help/strings";

import HelpHeader from "./HelpHeader";
import HelpNavTree, { type HelpNavData } from "./HelpNavTree";

type Props = {
  locale: string;
  activeCategory?: HelpCategoryId;
  activeSlug?: string;
  children: ReactNode;
  /** Logo or wordmark; rendered inside the home link. */
  brand?: ReactNode;
  /** Right-hand header slot — sign-in, primary CTA. */
  headerActions?: ReactNode;
  footer?: ReactNode;
};

export default function HelpShell({
  locale,
  activeCategory,
  activeSlug,
  children,
  brand,
  headerActions,
  footer,
}: Props) {
  const index = getHelpIndex(locale);
  const strings = getHelpStrings(locale);

  // Summaries only: the corpus must not be serialised into the client payload.
  const nav: HelpNavData = {
    categories: index.categories.map((category) => ({
      id: category.id,
      label: strings.categories[category.id].label,
      articleCount: category.articleCount,
    })),
    articlesByCategory: Object.fromEntries(
      index.categories.map((category) => [
        category.id,
        index.byCategory[category.id].map(toSummary),
      ]),
    ) as Record<HelpCategoryId, HelpArticleSummary[]>,
  };
  const searchDocs = index.articles.map(toSearchDoc);

  return (
    <div className="flex min-h-screen flex-col" data-help-shell>
      <HelpHeader
        locale={locale}
        strings={strings}
        nav={nav}
        searchDocs={searchDocs}
        activeCategory={activeCategory}
        activeSlug={activeSlug}
        brand={brand}
        actions={headerActions}
      />
      <main className="flex-1">
        <div className="mx-auto w-full max-w-7xl px-6 pt-24 pb-24">
          <div className="flex gap-12">
            <aside className="hidden w-56 shrink-0 lg:block">
              {/*
               * Sticky AND independently scrollable. A sticky element taller
               * than the viewport cannot be scrolled to its end until the
               * page reaches the bottom; with a few dozen articles in one
               * category the tree is taller than most screens.
               */}
              <nav
                aria-label={strings.title}
                className="sticky top-24 max-h-[calc(100vh-7rem)] overflow-y-auto"
              >
                <HelpNavTree
                  locale={locale}
                  nav={nav}
                  homeLabel={strings.allArticles}
                  activeCategory={activeCategory}
                  activeSlug={activeSlug}
                />
              </nav>
            </aside>
            <div className="min-w-0 flex-1">{children}</div>
          </div>
        </div>
      </main>
      {footer}
    </div>
  );
}
```

`brand`, `headerActions` and `footer` are `ReactNode` props — the one kind of
"function-like" thing that crosses the server → client boundary. The host
passes its logo, its CTA button and its footer without the module importing
any of them.

## Header

```tsx
// file: src/components/help/HelpHeader.tsx
"use client";

import Link from "next/link";
import { type ReactNode, useCallback, useState } from "react";

import { reportHelpSearchMiss } from "@/lib/help/analytics";
import type { HelpCategoryId } from "@/lib/help/config";
import type { HelpSearchDoc } from "@/lib/help/model";
import { helpHref } from "@/lib/help/paths";
import type { HelpStrings } from "@/lib/help/strings";

import HelpDrawer from "./HelpDrawer";
import HelpNavTree, { type HelpNavData } from "./HelpNavTree";
import HelpSearch from "./HelpSearch";

type Props = {
  locale: string;
  strings: HelpStrings;
  nav: HelpNavData;
  searchDocs: HelpSearchDoc[];
  activeCategory?: HelpCategoryId;
  activeSlug?: string;
  brand?: ReactNode;
  actions?: ReactNode;
};

export default function HelpHeader({
  locale,
  strings,
  nav,
  searchDocs,
  activeCategory,
  activeSlug,
  brand,
  actions,
}: Props) {
  const [menuOpen, setMenuOpen] = useState(false);
  const [mobileSearchOpen, setMobileSearchOpen] = useState(false);
  // Stable callbacks: HelpDrawer's effect depends on onClose.
  const closeMenu = useCallback(() => setMenuOpen(false), []);
  const closeMobileSearch = useCallback(() => setMobileSearchOpen(false), []);

  const categoryLabels = Object.fromEntries(
    nav.categories.map((category) => [category.id, category.label]),
  ) as Record<HelpCategoryId, string>;

  return (
    <>
      <header className="fixed inset-x-0 top-0 z-40" data-help-header>
        <div className="mx-auto flex h-16 w-full max-w-7xl items-center gap-4 px-6">
          <Link href={helpHref.home(locale)} className="flex shrink-0 items-center gap-2 lg:flex-1">
            {brand}
            <span>{strings.title}</span>
          </Link>

          <div className="hidden flex-1 justify-center lg:flex">
            <div className="w-full max-w-md">
              <HelpSearch
                locale={locale}
                docs={searchDocs}
                strings={strings}
                categoryLabels={categoryLabels}
                onNoResults={reportHelpSearchMiss}
              />
            </div>
          </div>

          <div className="hidden flex-1 justify-end lg:flex">{actions}</div>
          <div className="flex-1 lg:hidden" />

          <button
            type="button"
            className="lg:hidden"
            aria-label={strings.openSearch}
            aria-expanded={mobileSearchOpen}
            onClick={() => setMobileSearchOpen((open) => !open)}
          >
            <span aria-hidden="true" data-help-icon="search" />
          </button>
          <button
            type="button"
            className="lg:hidden"
            aria-label={strings.openMenu}
            onClick={() => setMenuOpen(true)}
          >
            <span aria-hidden="true" data-help-icon="menu" />
          </button>
        </div>

        {mobileSearchOpen && (
          <div className="px-6 pb-3 lg:hidden">
            <HelpSearch
              locale={locale}
              docs={searchDocs}
              strings={strings}
              categoryLabels={categoryLabels}
              autoFocus
              onNoResults={reportHelpSearchMiss}
              onSelect={closeMobileSearch}
            />
          </div>
        )}
      </header>

      <HelpDrawer
        open={menuOpen}
        onClose={closeMenu}
        title={strings.title}
        closeLabel={strings.closeMenu}
      >
        <HelpNavTree
          locale={locale}
          nav={nav}
          homeLabel={strings.allArticles}
          activeCategory={activeCategory}
          activeSlug={activeSlug}
          onNavigate={closeMenu}
        />
      </HelpDrawer>
    </>
  );
}
```

`data-help-icon="search|menu"` spans are icon slots: swap them for the host's
icon component. The analytics seam lives in a module import because a callback
prop cannot come from the server shell:

```ts
// file: src/lib/help/analytics.ts
/**
 * SEAM: forward zero-result searches to the host's analytics. They are the
 * best signal for which article to write next. Default: dev console only.
 */
export function reportHelpSearchMiss(query: string): void {
  if (process.env.NODE_ENV !== "production") console.debug(`[help] no results for "${query}"`);
}
```

## Navigation tree

One component for both the sidebar and the drawer. The earlier implementation
had two copies with the same collapsible logic and different height caps, and
the sidebar's cap is what hid half a category.

```tsx
// file: src/components/help/HelpNavTree.tsx
"use client";

import Link from "next/link";
import { useState } from "react";

import type { HelpCategoryId } from "@/lib/help/config";
import type { HelpArticleSummary } from "@/lib/help/model";
import { helpHref } from "@/lib/help/paths";

export type HelpNavCategory = { id: HelpCategoryId; label: string; articleCount: number };

export type HelpNavData = {
  categories: HelpNavCategory[];
  articlesByCategory: Record<HelpCategoryId, HelpArticleSummary[]>;
};

type Props = {
  locale: string;
  nav: HelpNavData;
  homeLabel: string;
  activeCategory?: HelpCategoryId;
  activeSlug?: string;
  /** Drawer passes its close handler; the sidebar passes nothing. */
  onNavigate?: () => void;
};

export default function HelpNavTree({
  locale,
  nav,
  homeLabel,
  activeCategory,
  activeSlug,
  onNavigate,
}: Props) {
  // Everything open by default: a help center is browsed, and a collapsed tree hides the corpus.
  const [open, setOpen] = useState<Record<string, boolean>>(() =>
    Object.fromEntries(nav.categories.map((category) => [category.id, true])),
  );
  const toggle = (id: string): void =>
    setOpen((previous) => ({ ...previous, [id]: !(previous[id] ?? true) }));

  return (
    <div data-help-nav>
      <Link
        href={helpHref.home(locale)}
        aria-current={activeCategory ? undefined : "page"}
        onClick={onNavigate}
        className="block"
      >
        {homeLabel}
      </Link>

      <ul>
        {nav.categories.map((category) => {
          const isOpen = open[category.id] ?? true;
          const listId = `help-nav-${category.id}`;
          return (
            <li key={category.id}>
              <button
                type="button"
                aria-expanded={isOpen}
                aria-controls={listId}
                onClick={() => toggle(category.id)}
                className="flex w-full items-center justify-between gap-2"
              >
                <span>
                  {category.label} <span data-help-count>{category.articleCount}</span>
                </span>
                <span aria-hidden="true" data-help-chevron data-open={isOpen || undefined} />
              </button>
              {/*
               * `hidden`, not a max-height transition. A capped max-height with
               * overflow-hidden silently clips whatever does not fit; in the
               * earlier implementation more than half the links in one
               * category were unreachable from the sidebar and nothing
               * signalled it. Animate with the host's Collapsible primitive
               * if wanted, never with a fixed cap.
               */}
              <ul id={listId} hidden={!isOpen}>
                {(nav.articlesByCategory[category.id] ?? []).map((article) => {
                  const active = activeCategory === category.id && activeSlug === article.slug;
                  return (
                    <li key={article.slug}>
                      <Link
                        href={helpHref.article(locale, category.id, article.slug)}
                        aria-current={active ? "page" : undefined}
                        onClick={onNavigate}
                        className="block"
                      >
                        {article.title}
                      </Link>
                    </li>
                  );
                })}
              </ul>
            </li>
          );
        })}
      </ul>
    </div>
  );
}
```

Style hooks: `[data-help-nav] a[aria-current=page]`, `[data-help-chevron][data-open]`.

## Drawer

Use the host's Sheet/Drawer primitive if it has one (shadcn `Sheet`, Radix
`Dialog`). This reference version covers what a bare `div` overlay misses:
Escape closes, body scroll is locked, focus moves in and is restored.

```tsx
// file: src/components/help/HelpDrawer.tsx
"use client";

import { type ReactNode, useEffect, useRef } from "react";

type Props = {
  open: boolean;
  /** Must be referentially stable (useCallback) or the effect re-runs every render. */
  onClose: () => void;
  title: string;
  closeLabel: string;
  children: ReactNode;
};

export default function HelpDrawer({ open, onClose, title, closeLabel, children }: Props) {
  const panelRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!open) return;
    const previouslyFocused = document.activeElement instanceof HTMLElement ? document.activeElement : null;
    const previousOverflow = document.body.style.overflow;
    document.body.style.overflow = "hidden";
    panelRef.current?.focus();
    const onKeyDown = (event: KeyboardEvent): void => {
      if (event.key === "Escape") onClose();
    };
    document.addEventListener("keydown", onKeyDown);
    return () => {
      document.removeEventListener("keydown", onKeyDown);
      document.body.style.overflow = previousOverflow;
      previouslyFocused?.focus();
    };
  }, [open, onClose]);

  if (!open) return null;

  return (
    <div className="fixed inset-0 z-50 lg:hidden" data-help-drawer>
      <button
        type="button"
        aria-label={closeLabel}
        onClick={onClose}
        className="absolute inset-0 h-full w-full"
        data-help-drawer-backdrop
      />
      <div
        ref={panelRef}
        tabIndex={-1}
        role="dialog"
        aria-modal="true"
        aria-label={title}
        className="absolute inset-y-0 right-0 flex w-72 max-w-full flex-col overflow-y-auto outline-none"
        data-help-drawer-panel
      >
        <div className="flex h-16 shrink-0 items-center justify-between px-5">
          <span>{title}</span>
          <button type="button" aria-label={closeLabel} onClick={onClose}>
            <span aria-hidden="true" data-help-icon="close" />
          </button>
        </div>
        <div className="flex-1 px-4 py-4">{children}</div>
      </div>
    </div>
  );
}
```

Breadcrumb, category cards, article lists, the markdown renderer and the
table of `data-help-*` style hooks are in [ui-content.md](ui-content.md).

## Checklist

- [ ] `HelpShell` receives the host's brand, CTA and footer as props
- [ ] Host primitives swapped in for input, drawer, icons where they exist
- [ ] Sidebar scrolls independently; longest category fully reachable
- [ ] Mobile: search toggle, drawer, Escape, scroll lock verified
- [ ] Active/expanded states styled — hook table in [ui-content.md](ui-content.md)
