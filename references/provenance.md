# Provenance

Written by the engineer who has shipped this module. The earlier
implementation it was audited against, a marketing-site help center, ran 65
markdown articles in three categories, English content served under three
locale prefixes, client-side search, a collapsible sidebar and a mobile drawer,
JSON-LD and sitemap integration. The architecture is that module's. The
templates are not a transcription of it: the audit below is why.

Every claim here was verified against the earlier implementation, its code or
its rendered pages, not inferred from a summary.

Defects 1 to 11 were found on audit day (0.1.0); 12 and 13 surfaced in the
same earlier implementation over the following hours and were recorded in
0.2.0.

## Fixed in the templates

### 1. The sidebar hid half of the biggest category

The desktop sidebar animated category collapse with `max-h-[600px]` +
`overflow-hidden`. The largest category had 45 articles at ~31 px each. Measured
on the live site: the list's scroll height was 1410 px inside a 600 px box, and
**25 of 45 links were unreachable** from the sidebar. The mobile drawer used a
different cap (2000 px) and was fine — so nobody testing on a phone saw it, and
on desktop the list simply looked complete.

**Shipped:** `hidden` attribute, no height cap; one `HelpNavTree` for both
surfaces; the sticky sidebar scrolls independently so the fixed sidebar cannot
hide its own tail. See [ui.md](ui.md).

### 2. Search that missed what people typed

Substring match on title and description only. Consequences, each verified in
the code:

- The query was lowercased but not trimmed: `"webhooks "` (trailing space —
  what mobile keyboards insert after autocomplete) matched nothing.
- Tags and headings were not searched, so a term that only appeared in a
  section heading returned no result.
- No ranking: a title hit and a description hit tied, ordered by folder.
- Enter always opened the *first* result; no arrow keys, no ARIA.

**Shipped:** tokenised prefix search with field weights, diacritics folding,
AND semantics, keyboard navigation and combobox semantics, plus a zero-result
hook. See [search.md](search.md).

### 3. Order ties resolved by filesystem order

Articles sorted by `order` only. The corpus had three pairs sharing an order
(`2`, `5`, `27`). JavaScript's sort is stable, so ties kept `readdir` order —
which is alphabetical on APFS, creation order on some Linux filesystems, and
not guaranteed anywhere. The same deploy could list two articles in a different
order on a different build machine.

**Shipped:** `order`, then title (locale collation), then slug; validation
warns on ties. See [content-loader.md](content-loader.md).

### 4. Related links that vanish silently

`related` entries were resolved by slug across every category, and unresolved
ones were dropped without a warning anywhere. Renaming an article broke every
inbound `related` reference invisibly. Slugs were also only unique per folder,
so a bare reference could resolve to the wrong category's article.

**Shipped:** same-category-first resolution with an explicit `category/slug`
form; a validator that fails the build on unresolved references and warns on
ambiguous ones. See [content-model.md](content-model.md).

### 5. The corpus re-parsed several times per page

Six loader functions each walked the tree. One article render called them
from the layout, the page, `generateMetadata` and the related resolver —
around four full parses of every file per page, and again per locale in
`generateStaticParams`.

**Shipped:** one `getHelpIndex(locale)` under React `cache()`, read by every
call site. See [content-loader.md](content-loader.md).

### 6. "Popular articles" that were not

The landing page showed `getAllHelpArticles().slice(0, 6)` under the heading
*Popular articles*: the first six files of the first folder, by `order`.

**Shipped:** a `featured` frontmatter flag with the previous behaviour as the
fallback, and a heading that says what it is.

### 7. English chrome on a three-locale site

The site rendered Polish and German everywhere except the help center, whose
header, sidebar, buttons, category labels and "Updated {en-US date}" were
literals — in an app with a working i18n dictionary that the same component
tree used for its footer.

**Shipped:** a typed strings table with a locale seam, `Intl` for dates,
plurals and collation. See [i18n.md](i18n.md).

### 8. `order: 0` sorted last

`(frontMatter.order as number) || 999` — zero is falsy. Minor, but the kind of
thing that wastes an afternoon.

**Shipped:** `num(data, "order") ?? Number.MAX_SAFE_INTEGER`.

### 9. Invalid structured data when a date was missing

`dateModified: article.updatedAt` emitted `""` for articles without a date.
JSON-LD was also interpolated unescaped; a `</script>` in a title would have
ended the block.

**Shipped:** conditional fields and `serializeJsonLd`. See
[routes.md](routes.md). The escaping is hardening — no title in the corpus
contained `<`.

### 10. Dead code

A `HelpSearchBar` component rendering a disabled input labelled "Soon", and a
`getHelpArticlesByTag` function — both exported, neither imported anywhere.

**Shipped:** neither. Tag pages arrived in 0.2.0 ([tags.md](tags.md)), not
built on this function — it matched exact strings and would have made `API`
and `api` two pages (defect 13).

### 11. Frontmatter that lied without consequence

`slug` and `category` in frontmatter were ignored by the loader (filename and
folder win). Nothing detected a mismatch. The corpus had none — but the next
copy-pasted file would.

**Shipped:** the loader records mismatches as `issues`; validation fails on
them.

### 12. A frontmatter parser that dropped multi-line values

The earlier implementation parsed frontmatter line by line: `key: value`,
JSON-parse anything in `[...]`. Prettier wraps arrays past 100 columns onto
several lines, and every such value read as empty, silently. Found in the
earlier implementation hours *after* the audit, when a new article's `related`
list came out blank: **45 of 161** content files across the site's four
markdown content types (the help center and three others sharing the parser)
had an empty list field in production, `related` and `tags` among them,
including cards that had rendering code and had never once rendered.

The templates never had this — the skill's loader used `gray-matter` from
0.1.0 — so this entry exists to name the trap: **never hand-roll a
frontmatter parser** for content a formatter touches.

The earlier implementation's fix kept its line parser as a *fallback* when
YAML fails, because one file elsewhere on the site had a quoted scalar
containing raw quotes. The skill deliberately does not: an unparsable file is
`skipped`, and `validate:help` fails the build with the file name. Fallback
parsing is right
for content with no validator; with one, it only hides the file that needs
fixing. If a host insists, wrap
`parseFrontmatter` — do not weaken the loader.

### 13. Tags keyed by spelling

The earlier implementation's only tag lookup, `getHelpArticlesByTag`, matched
the **exact** string (`tags.includes(tag)`), and nothing ever normalised a tag
for a URL.
Run over the corpus, the 0.2.0 validator found, invisible on the site:

- **6 slugs with several spellings** — `api`/`API`, `rest`/`REST`,
  `sdk`/`SDK`, `graphql`/`GraphQL`, `booking page`/`booking-page`, and
  `google calendar`/`Google Calendar`/`google-calendar` (three);
- **5 singular/plural pairs** — `booking(s)`, `service(s)`, `location(s)`,
  `email(s)`, `integration(s)`;
- **157 distinct tags on 65 articles, 101 used once** — chips, not
  navigation, until a tag has a page and a count next to it.

**Shipped:** slug identity (`tagSlug`, folded like search), majority-spelling
labels, per-tag pages generated from `index.tags`, and validation warnings for
each of the three drifts above. See [tags.md](tags.md).

### What validation found in the real corpus

Running the shipped validator over the earlier implementation's content on
audit day: 3 order ties, 0 unresolved references, 0 slug mismatches, and 37 of
65 `updatedAt` values identical (the import date). None of these were visible
on the site. The date problem is a content problem; deriving dates from git is in
[extensions.md](extensions.md).

The 0.2.0 run added the tag findings in defect 13 — 11 warnings, 0 errors —
and found the same 3 order ties still present.

## Kept deliberately

- **Filename is the slug; folder is the category.** One source of truth,
  visible in a file listing. Frontmatter duplicates are validated, not used.
- **Only summaries reach the client.** The layout stripped `content` before
  passing articles to client components. Kept and made explicit.
- **Client-side search over a prebuilt index.** Right-sized for hundreds of
  articles; no service to run.
- **`dynamicParams = false`** with routing-level 404s — a workaround for a
  real Next.js 16.2 bug, and independently the right choice for a finite,
  build-time-known page set.
- **Untranslated locale pages canonicalise to the default locale** and stay
  out of the sitemap. The earlier implementation did this correctly for its
  English-only content; the templates generalise it.
- **All categories open by default** in both the sidebar and the drawer.
- **A per-page shell component instead of `layout.tsx`** — a layout cannot
  see its children's `slug`.
- **The renderer demotes `#` to `<h2>`**: 23 of 65 articles started with an
  H1 despite the page owning it.
- **Content in the repository**, published by deploy. No CMS, no
  `revalidate` — a content change is a code change.

## Added

Not in the earlier implementation; designed in the skill, never run there, and
marked as such wherever it appears:

- Per-locale content directories with default-locale fallback and a
  `translated` flag driving canonical, hreflang, sitemap and a reader notice.
  Modelled on the host's other localized content, simplified because help
  slugs are stable across languages.
- The validation script and its tests.
- The search scoring model, keyboard handling and zero-result hook, with tests.
- The `featured` flag.
- The per-request index.
- The strings table.
- Drawer accessibility: Escape, scroll lock, focus restore.
- Loader tests against a fixture tree.
- **0.2.0:** the tag model — `tagSlug`, `index.tags`/`byTag`, the tag page,
  linked chips, tag sitemap entries — and the three tag validation checks
  with tests. The earlier implementation had plain chips and an unused
  exact-match filter; all of this is an addition.

## If you are fixing the earlier implementation

Fix in this order — the first is live for every desktop visitor:

1. **Defect 1** — the sidebar cap. One-line change (`max-h-[600px]` → conditional render), immediate effect.
2. **Defect 2** — trim the query, add tags and headings to the index.
3. **Defect 4** — add the validator and run it; fix what it reports.
4. **Defect 3** — the sort tiebreak.
5. **Defect 7** — strings, if the site has more than one locale.
6. Then 5, 6, 8, 9, 10, 11 as hygiene.
7. **Defect 12** — if any content type still has a line-based parser,
   replace it before adding a single article; diff every field before and
   after, as the earlier implementation did (161 files, no data lost).
8. **Defect 13** — before adding tag pages, key by slug and run the validator;
   fix the spellings in content, not in code.
