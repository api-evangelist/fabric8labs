---
name: Search and inventory all Fabric8Labs public content
description: Enumerate every publicly readable record on www.fabric8labs.com in a handful of calls
  using the unified search endpoint and the type/taxonomy discovery routes, then resolve results to
  their source collections.
api: openapi/fabric8labs-search-api-openapi.yml
operations: [listSearch, listTypes, getTypesById, listTaxonomies, getTaxonomiesById, listStatuses, listPosts, listPages]
generated: '2026-08-12'
method: generated
source: openapi/ + data-model/fabric8labs-data-model.yml + conventions/fabric8labs-conventions.yml
---

# Search and inventory all Fabric8Labs public content

Base URL: `https://www.fabric8labs.com/wp-json`. Anonymous — send no credential.

Use this skill when you need the *whole* public corpus rather than one collection: 13 posts,
27 pages and a 3-record team type, all reachable through one endpoint.

## Step 1 — discover what exists before you fetch it

Call `listTypes` — `GET /wp/v2/types`. This returns every registered post type with its `rest_base`,
`rest_namespace`, `taxonomies` and `supports`. It tells you which collections are real on **this**
site rather than assuming WordPress defaults. The company-specific one is `team`.

Call `listTaxonomies` — `GET /wp/v2/taxonomies` — for the vocabularies and which types they apply to.

Call `listStatuses` — `GET /wp/v2/statuses` — if you need to reason about visibility. Anonymously you
will only ever see `publish`.

Do this once per session and cache it. It is the map for everything below.

## Step 2 — sweep the corpus with one endpoint

Call `listSearch` — `GET /wp/v2/search`.

```
GET /wp/v2/search?per_page=100&search=
```

Returns a thin projection: `id`, `title`, `url`, `type`, `subtype`. Read `X-WP-Total` for the
corpus size (28 at last measurement for a cooling-related query; an empty `search` returns the full
public set).

**The critical detail:** `type` is almost always the literal string `post`, while `subtype` carries
the real collection — `post`, `page` or `team`. Resolve ids against **`subtype`**, not `type`. A
result with `subtype: page` and id 509 lives at `/wp/v2/pages/509`, and fetching `/wp/v2/posts/509`
will return `404 rest_post_invalid_id`.

The safe alternative is to skip the guesswork entirely and follow `_links.self[0].href` on each
result, which WordPress already points at the correct collection.

## Step 3 — hydrate what you need

For each id, call the operation for its subtype:

| subtype | operation | route |
|---|---|---|
| `post` | `getPostsById` | `/wp/v2/posts/{id}` |
| `page` | `getPagesById` | `/wp/v2/pages/{id}` |
| `team` | `getTeamById` | `/wp/v2/team/{id}` |

Or bulk-hydrate with `include`: `GET /wp/v2/pages?include=509,512,514&_fields=id,title,link,excerpt`
— one call instead of three.

## Step 4 — know what you cannot get

- `/wp/v2/users` returns **401** `rest_user_cannot_view`. The `author` id on every post, page and
  attachment is therefore unresolvable. Do not report an author name; report that it is not public.
- `/wp/v2/comments` returns **403** `rest_comment_disabled`.
- `/wp/v2/settings`, `/wp-abilities/v1/abilities` and `/gf/v2/forms` return **401**.
- `/wp/v2/team` records are unedited theme placeholders titled "Team Member Name". The type is real
  and public but was never populated. Do not present them as the leadership team — the real one is
  rendered on `https://www.fabric8labs.com/about/`.

## Rules

- Page with `page` + `per_page` (max 100); follow the `Link: rel="next"` header rather than
  incrementing blindly.
- No rate-limit headers exist. Pace to the published `Crawl-delay: 10` and batch with `include` and
  `_fields`.
- Errors are the WordPress envelope `{"code","message","data":{"status"}}`, not problem+json.
- Every operation here is a safe GET. Nothing in this skill mutates anything.
