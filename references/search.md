# Search

Client-side, over a prebuilt index of summaries. No service, no route handler,
no dependency. For a help center under a few hundred articles this is the right
size: the index is ~1 KB per article, it is already in the page for navigation,
and results appear on the first keystroke with no network.

The earlier implementation searched `title` and `description` with
`includes()`. That misses
an article whose only mention of "webhook" is in a heading, ranks a description
hit equal to a title hit, and — because the query was lowercased but never
trimmed — returned nothing for `"webhooks "` with the trailing space mobile
keyboards add after autocomplete. All three are fixed below.

## Index document

```ts
export type HelpSearchDoc = HelpArticleSummary & { headings: string[] };
```

Defined in `model.ts` ([content-model.md](content-model.md)); built server-side
with `toSearchDoc(article)`. Headings come from the loader, extracted outside
code fences. Body text is deliberately not indexed — see
[extensions.md](extensions.md) for full-text options when headings stop being
enough.

## Scoring

```ts
// file: src/lib/help/search.ts
import type { HelpSearchDoc } from "./model";

export type HelpSearchOptions = {
  /** Maximum results returned. */
  limit?: number;
  /** Queries shorter than this (after trimming) return nothing. */
  minLength?: number;
};

const WEIGHTS = { title: 10, tags: 6, headings: 4, description: 2 } as const;
type Field = keyof typeof WEIGHTS;
const FIELDS = Object.keys(WEIGHTS) as Field[];

// NFD splits "é" into "e" + combining acute, which the range strips. Letters
// that do not decompose need an explicit map — Polish ł is the common one.
const NON_DECOMPOSABLE: Record<string, string> = {
  ł: "l",
  ß: "ss",
  ø: "o",
  æ: "ae",
  œ: "oe",
  đ: "d",
};

export function normalizeText(text: string): string {
  return text
    .toLowerCase()
    .normalize("NFD")
    .replace(/[\u0300-\u036f]/g, "")
    .replace(/[łßøæœđ]/g, (ch) => NON_DECOMPOSABLE[ch] ?? ch);
}

export function tokenize(text: string): string[] {
  return normalizeText(text)
    .split(/[^\p{L}\p{N}]+/u)
    .filter(Boolean);
}

export type PreparedSearchDoc = {
  doc: HelpSearchDoc;
  titleText: string;
  fields: Record<Field, string[]>;
};

/** Tokenise once; the hook memoises this across keystrokes. */
export function prepareSearchDocs(docs: readonly HelpSearchDoc[]): PreparedSearchDoc[] {
  return docs.map((doc) => ({
    doc,
    titleText: normalizeText(doc.title),
    fields: {
      title: tokenize(doc.title),
      tags: doc.tags.flatMap(tokenize),
      headings: doc.headings.flatMap(tokenize),
      description: tokenize(doc.description),
    },
  }));
}

function fieldMatches(tokens: readonly string[], query: string): boolean {
  return tokens.some((token) => token.startsWith(query));
}

/**
 * AND over query tokens: every token must prefix-match some field, or the
 * document scores 0. Each token adds the weight of every field it hits, so a
 * term in both title and a heading outranks the same term in a description.
 * Multi-word queries that appear verbatim in the title get a title-sized bonus.
 */
export function scorePreparedDoc(
  prepared: PreparedSearchDoc,
  queryTokens: readonly string[],
  phrase: string,
): number {
  let score = 0;
  for (const token of queryTokens) {
    let hit = 0;
    for (const field of FIELDS) {
      if (fieldMatches(prepared.fields[field], token)) hit += WEIGHTS[field];
    }
    if (hit === 0) return 0;
    score += hit;
  }
  if (queryTokens.length > 1 && prepared.titleText.includes(phrase)) score += WEIGHTS.title;
  return score;
}

export function searchPreparedDocs(
  prepared: readonly PreparedSearchDoc[],
  query: string,
  { limit = 8, minLength = 2 }: HelpSearchOptions = {},
): HelpSearchDoc[] {
  const tokens = tokenize(query);
  const phrase = tokens.join(" ");
  if (phrase.length < minLength) return [];

  return prepared
    .map((p, position) => ({ p, position, score: scorePreparedDoc(p, tokens, phrase) }))
    .filter((entry) => entry.score > 0)
    // Ties keep index order, which is category order then article order —
    // so equally-scored results appear in the same order as the sidebar.
    .sort((a, b) => b.score - a.score || a.position - b.position)
    .slice(0, limit)
    .map((entry) => entry.p.doc);
}

/** Convenience for one-off calls and tests; prefer the prepared form in components. */
export function searchHelp(
  docs: readonly HelpSearchDoc[],
  query: string,
  options?: HelpSearchOptions,
): HelpSearchDoc[] {
  return searchPreparedDocs(prepareSearchDocs(docs), query, options);
}
```

Trade-offs, so nobody "improves" them blind:

| Choice | Why | When to revisit |
|---|---|---|
| Prefix match, not fuzzy | Predictable; "webh" finds Webhooks, "wbhook" does not. Fuzzy matching on 60 short titles produces more noise than recall | Users report zero-result typos in the log — then consider a Levenshtein tolerance of 1 on tokens ≥ 5 chars |
| AND across tokens | "google calendar" should not return every calendar article | Never; OR belongs to full-text engines with relevance ranking |
| Diacritics folded | `platnosci` must find `Płatności` — users type without accents on phones | Never |
| No stemming | "bookings" does not find "booking"; prefix covers the common direction ("book" finds both) | If the language has heavy inflection (Polish), add a per-locale stem list |
| Ties by index order | Deterministic and matches the sidebar | Never |

## The hook

```tsx
// file: src/components/help/useHelpSearch.ts
"use client";

import { type KeyboardEvent, useCallback, useEffect, useMemo, useRef, useState } from "react";

import type { HelpSearchDoc } from "@/lib/help/model";
import { type HelpSearchOptions, prepareSearchDocs, searchPreparedDocs } from "@/lib/help/search";

type Options = HelpSearchOptions & {
  /**
   * Called ~400 ms after the user stops typing a query that matched nothing.
   * Zero-result queries are the single best signal for which article to write
   * next; wire this to analytics.
   */
  onNoResults?: (query: string) => void;
};

export function useHelpSearch(docs: readonly HelpSearchDoc[], options: Options = {}) {
  const { limit = 8, minLength = 2, onNoResults } = options;
  const prepared = useMemo(() => prepareSearchDocs(docs), [docs]);
  const [query, setQueryState] = useState("");
  const [activeIndex, setActiveIndex] = useState(0);

  const results = useMemo(
    () => searchPreparedDocs(prepared, query, { limit, minLength }),
    [prepared, query, limit, minLength],
  );

  // Latest callback without re-arming the debounce on every render.
  const onNoResultsRef = useRef(onNoResults);
  useEffect(() => {
    onNoResultsRef.current = onNoResults;
  });

  useEffect(() => {
    const trimmed = query.trim();
    if (trimmed.length < minLength || results.length > 0) return;
    const id = setTimeout(() => onNoResultsRef.current?.(trimmed), 400);
    return () => clearTimeout(id);
  }, [query, results.length, minLength]);

  const setQuery = useCallback((value: string) => {
    setQueryState(value);
    setActiveIndex(0);
  }, []);

  const clear = useCallback(() => {
    setQueryState("");
    setActiveIndex(0);
  }, []);

  const handleKeyDown = useCallback(
    (event: KeyboardEvent<HTMLInputElement>, select: (doc: HelpSearchDoc) => void) => {
      if (results.length === 0) {
        if (event.key === "Escape") clear();
        return;
      }
      switch (event.key) {
        case "ArrowDown":
          event.preventDefault();
          setActiveIndex((i) => (i + 1) % results.length);
          break;
        case "ArrowUp":
          event.preventDefault();
          setActiveIndex((i) => (i - 1 + results.length) % results.length);
          break;
        case "Enter": {
          const active = results[activeIndex];
          if (active) {
            event.preventDefault();
            select(active);
            clear();
          }
          break;
        }
        case "Escape":
          clear();
          break;
        default:
          break;
      }
    },
    [results, activeIndex, clear],
  );

  return {
    query,
    setQuery,
    results,
    activeIndex,
    setActiveIndex,
    clear,
    handleKeyDown,
    isOpen: results.length > 0,
  };
}
```

## The component

Combobox semantics so screen readers announce the result count and the active
option, arrow keys move, Enter follows the active result (the earlier
implementation followed the *first* result on Enter regardless), Escape clears,
and focus leaving the
container closes the list without a document-level listener.

```tsx
// file: src/components/help/HelpSearch.tsx
"use client";

import Link from "next/link";
import { useRouter } from "next/navigation";
import { useId, useRef } from "react";

import type { HelpCategoryId } from "@/lib/help/config";
import type { HelpSearchDoc } from "@/lib/help/model";
import { helpHref } from "@/lib/help/paths";
import { formatMessage, type HelpStrings } from "@/lib/help/strings";

import { useHelpSearch } from "./useHelpSearch";

type Props = {
  locale: string;
  docs: readonly HelpSearchDoc[];
  strings: Pick<HelpStrings, "searchPlaceholder" | "searchLabel" | "noResults">;
  categoryLabels: Record<HelpCategoryId, string>;
  autoFocus?: boolean;
  onNoResults?: (query: string) => void;
  /** Called after a result is chosen — the mobile header closes its search panel here. */
  onSelect?: () => void;
};

export default function HelpSearch({
  locale,
  docs,
  strings,
  categoryLabels,
  autoFocus,
  onNoResults,
  onSelect,
}: Props) {
  const router = useRouter();
  const listId = useId();
  const containerRef = useRef<HTMLDivElement>(null);
  const search = useHelpSearch(docs, { onNoResults });
  const showEmpty = search.query.trim().length >= 2 && search.results.length === 0;

  function go(doc: HelpSearchDoc): void {
    router.push(helpHref.article(locale, doc.category, doc.slug));
    onSelect?.();
  }

  return (
    <div
      ref={containerRef}
      className="relative w-full"
      data-help-search
      onBlur={(event) => {
        // relatedTarget is null when focus leaves the document or lands on a
        // non-focusable element; both mean "closed".
        const next = event.relatedTarget;
        if (!(next instanceof Node) || !containerRef.current?.contains(next)) search.clear();
      }}
    >
      <input
        type="text"
        role="combobox"
        aria-label={strings.searchLabel}
        aria-expanded={search.isOpen}
        aria-controls={listId}
        aria-autocomplete="list"
        aria-activedescendant={search.isOpen ? `${listId}-${search.activeIndex}` : undefined}
        placeholder={strings.searchPlaceholder}
        autoComplete="off"
        autoFocus={autoFocus}
        value={search.query}
        onChange={(event) => search.setQuery(event.target.value)}
        onKeyDown={(event) => search.handleKeyDown(event, go)}
        className="w-full"
      />

      {(search.isOpen || showEmpty) && (
        <div className="absolute inset-x-0 top-full z-50 mt-2" data-help-search-results>
          {showEmpty ? (
            <p role="status">{formatMessage(strings.noResults, { query: search.query.trim() })}</p>
          ) : (
            <ul id={listId} role="listbox" aria-label={strings.searchLabel}>
              {search.results.map((doc, index) => (
                <li
                  key={`${doc.category}/${doc.slug}`}
                  id={`${listId}-${index}`}
                  role="option"
                  aria-selected={index === search.activeIndex}
                  data-active={index === search.activeIndex || undefined}
                  onMouseEnter={() => search.setActiveIndex(index)}
                >
                  <Link
                    href={helpHref.article(locale, doc.category, doc.slug)}
                    tabIndex={-1}
                    onClick={() => {
                      search.clear();
                      onSelect?.();
                    }}
                    className="flex flex-col"
                  >
                    <span>{doc.title}</span>
                    <span>
                      {categoryLabels[doc.category]}
                      {doc.description ? ` · ${doc.description}` : ""}
                    </span>
                  </Link>
                </li>
              ))}
            </ul>
          )}
        </div>
      )}
    </div>
  );
}
```

Styling hooks: `[data-help-search]`, `[data-help-search-results]`,
`[role=option][data-active]`. Attach the host's input, popover and list-item
styles to those; the component ships layout intent only.

## Tests

```ts
// file: src/lib/help/search.test.ts
import { describe, expect, it } from "vitest";

import type { HelpSearchDoc } from "./model";
import { normalizeText, searchHelp, tokenize } from "./search";

const docs: HelpSearchDoc[] = [
  {
    slug: "webhooks",
    category: "development",
    title: "Webhooks",
    description: "Receive events over HTTP.",
    tags: ["integration"],
    headings: ["Supported events", "Retries"],
  },
  {
    slug: "add-google-calendar",
    category: "app",
    title: "Add Google Calendar",
    description: "Sync bookings with Google.",
    tags: ["calendar"],
    headings: ["Connect your account"],
  },
  {
    slug: "configure-stripe-payments",
    category: "app",
    title: "Płatności Stripe",
    description: "Accept card payments.",
    tags: ["payments", "stripe"],
    headings: [],
  },
  {
    slug: "api-overview",
    category: "development",
    title: "API overview",
    description: "Webhooks, GraphQL and REST endpoints.",
    tags: [],
    headings: [],
  },
];

describe("searchHelp", () => {
  it("ignores surrounding whitespace in the query", () => {
    expect(searchHelp(docs, "  webhooks  ").map((d) => d.slug)[0]).toBe("webhooks");
  });

  it("ranks a title hit above a description hit", () => {
    expect(searchHelp(docs, "webhooks").map((d) => d.slug)).toEqual(["webhooks", "api-overview"]);
  });

  it("matches tags and headings", () => {
    expect(searchHelp(docs, "integration").map((d) => d.slug)).toEqual(["webhooks"]);
    expect(searchHelp(docs, "retries").map((d) => d.slug)).toEqual(["webhooks"]);
  });

  it("folds diacritics in both directions", () => {
    expect(searchHelp(docs, "platnosci").map((d) => d.slug)).toEqual(["configure-stripe-payments"]);
    expect(searchHelp(docs, "płatności").map((d) => d.slug)).toEqual(["configure-stripe-payments"]);
  });

  it("requires every token to match (AND)", () => {
    expect(searchHelp(docs, "google calendar")).toHaveLength(1);
    expect(searchHelp(docs, "google webhooks")).toHaveLength(0);
  });

  it("prefix-matches tokens", () => {
    expect(searchHelp(docs, "webh").map((d) => d.slug)).toContain("webhooks");
    expect(searchHelp(docs, "ebhooks")).toHaveLength(0);
  });

  it("respects minLength and limit", () => {
    expect(searchHelp(docs, "w")).toHaveLength(0);
    expect(searchHelp(docs, "a", { minLength: 1, limit: 2 })).toHaveLength(2);
  });

  it("does not mutate its input", () => {
    const copy = structuredClone(docs);
    searchHelp(docs, "stripe");
    expect(docs).toEqual(copy);
  });
});

describe("tokenize", () => {
  it("splits on punctuation and folds case", () => {
    expect(tokenize("Add: Google-Calendar!")).toEqual(["add", "google", "calendar"]);
    expect(normalizeText("Łódź")).toBe("lodz");
  });
});
```

## Checklist

- [ ] `search.ts` copied; tests passing
- [ ] `HelpSearch` mounted in the header with `toSearchDoc` output from the layout
- [ ] `onNoResults` wired to analytics (or at least `console.debug` in dev)
- [ ] Verified: trailing-space query, accented query, keyboard-only selection
