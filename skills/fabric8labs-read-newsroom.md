---
name: Read the Fabric8Labs newsroom
description: Retrieve, filter and page through Fabric8Labs press releases and technical insight
  posts from the company's public WordPress REST API, resolving categories and featured images.
api: openapi/fabric8labs-posts-api-openapi.yml
operations: [listPosts, getPostsById, listCategories, getCategoriesById, listMedia, getMediaById]
generated: '2026-08-12'
method: generated
source: openapi/ + conventions/fabric8labs-conventions.yml + errors/fabric8labs-problem-types.yml
---

# Read the Fabric8Labs newsroom

Fabric8Labs has no developer program and no API keys. The newsroom is readable anonymously over the
site's own WordPress REST API. Base URL: `https://www.fabric8labs.com/wp-json`. Send no
`Authorization` header — there is nothing to send.

There are only 13 posts. Treat this as a small, slow-moving corpus, not a feed.

## Step 1 — list the posts

Call `listPosts` — `GET /wp/v2/posts`.

Ask for only the fields you need; the full record includes rendered HTML content and is large.

```
GET /wp/v2/posts?per_page=100&_fields=id,date,modified,slug,link,title,excerpt,categories,featured_media
```

- `per_page` is capped at **100**. Asking for more returns `400 rest_invalid_param`. With 13 posts
  one page is the whole corpus.
- Read `X-WP-Total` and `X-WP-TotalPages` from the response headers rather than counting.
- Default sort is `date` descending. Use `orderby=modified` to find recently edited items.
- `title`, `excerpt` and `content` are **objects** with a `rendered` string, not strings. Read
  `title.rendered`.

To narrow by date: `after=2026-01-01T00:00:00` / `before=`. To full-text search: `search=ECAM`.

## Step 2 — resolve categories

`categories` on each post is an array of integer term ids. Call `listCategories` —
`GET /wp/v2/categories?per_page=100&_fields=id,name,slug,count` — once and cache the map for the
session. The vocabulary is six terms and effectively only two are used: **News** (id 7, 11 posts)
and **Insights** (id 11, 2 posts).

To filter server-side instead, pass the id straight to the posts call:
`GET /wp/v2/posts?categories=7`.

Do not use `listTags` for this. The `post_tag` vocabulary is registered but has **zero** terms —
Fabric8Labs does not tag its content.

## Step 3 — resolve the featured image

`featured_media` is an attachment id, or `0` when unset. Call `getMediaById` —
`GET /wp/v2/media/{id}?_fields=id,source_url,alt_text,media_details,mime_type` — and use
`source_url`.

Cheaper alternative: add `_embed` to the Step 1 request and read
`_embedded['wp:featuredmedia'][0].source_url` inline, avoiding a call per post.

## Step 4 — fetch one post in full

Call `getPostsById` — `GET /wp/v2/posts/{id}`. `content.rendered` is HTML, not markdown or plain
text; strip or parse it before handing it to a model.

## Rules

- **No rate-limit signal exists.** No `X-RateLimit-*`, no `RateLimit-*`, no `Retry-After`. The host
  publishes `Crawl-delay: 10` in robots.txt — pace at roughly one request per 10 seconds and prefer
  one wide call over many narrow ones.
- **Cache.** Responses carry `Cache-Control: max-age=600` and `Last-Modified`. Send
  `If-Modified-Since` on repeat polls. There is no `ETag`.
- **Errors** are `{"code","message","data":{"status"}}` — not RFC 9457 problem+json. Branch on
  `code`: `rest_invalid_param` (400, bad parameter, read `data.params`), `rest_post_invalid_id`
  (404, no such published post).
- **Never retry a 4xx unchanged.** Nothing here is transient except a 5xx or a Cloudflare challenge.
- **Do not attempt writes.** POST/PUT/DELETE are registered on these routes but require WordPress
  application-password credentials that Fabric8Labs does not issue to third parties. You will get
  `401 rest_forbidden`.
