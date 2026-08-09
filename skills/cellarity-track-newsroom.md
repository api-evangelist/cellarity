---
name: Track the Cellarity newsroom
description: Pull Cellarity press releases, publications and external coverage from the cellarity.com content API, separating original announcements from third-party pickup using the article-type taxonomy.
api: openapi/cellarity-content-openapi.yml
base_url: https://cellarity.com/wp-json/wp/v2
operations:
  - listArticleTypes
  - listNewsItems
  - getNewsItem
  - getMediaItem
generated: '2026-08-09'
method: generated
source: openapi/cellarity-content-openapi.yml + conventions/cellarity-conventions.yml
---

# Track the Cellarity newsroom

Cellarity has no press API and no RSS feed — `https://cellarity.com/feed/` 301-redirects to the
homepage. The newsroom is only readable through the WordPress content API, as the `news_item`
custom post type. 48 records as of 2026-08-09.

## Authentication

None. Every operation here is anonymous. Do not send credentials.

## Step 1 — resolve the article-type taxonomy first

`article-type` is the single most important filter on this API: it is what separates a Cellarity
press release from a magazine writing about Cellarity. Resolve the term IDs before filtering,
because they are site-specific integers, not slugs.

    listArticleTypes
    GET /wp/v2/article-type?per_page=100&_fields=id,slug,name,count

Observed terms: `news` (29), `press` (14), `ext` (11), `publication` (2), `presentation` (2),
`media` (1). Counts total 59 against 48 items — items carry more than one term, so treat these as
overlapping labels, not a partition.

## Step 2 — list news items

    listNewsItems
    GET /wp/v2/news_item?per_page=100&orderby=date&order=desc&_fields=id,date,modified,slug,link,title,excerpt,article-type

Filter to a term with the taxonomy parameter, using the integer IDs from step 1:

    GET /wp/v2/news_item?article-type=<press_term_id>&per_page=100

Date-window the pull for incremental syncs:

    GET /wp/v2/news_item?after=2026-01-01T00:00:00&orderby=date&order=asc

## Step 3 — read a single item

    getNewsItem
    GET /wp/v2/news_item/{id}

Or resolve by slug without knowing the ID:

    GET /wp/v2/news_item?slug=<slug>

## Step 4 — attach the image, if you need it

`featured_media` is an integer media ID, `0` when unset.

    getMediaItem
    GET /wp/v2/media/{featured_media}?_fields=id,source_url,alt_text,media_details

Cheaper in one round trip: add `_embed=1` to step 2 or 3 and read
`_embedded['wp:featuredmedia'][0].source_url`.

## Rules

- **Content fields are objects, not strings.** Use `title.rendered`, `excerpt.rendered`,
  `content.rendered`. A client that treats `title` as a string will break on every record.
- **`rendered` values contain HTML entities** (`&#038;` for `&`). Decode before display.
- **Pagination is bounded.** `per_page` maximum is 100; above it you get `400 rest_invalid_param`
  with the detail in `data.params.per_page`. Read `X-WP-Total` and `X-WP-TotalPages` to size the
  walk; loop `page=1..X-WP-TotalPages`.
- **Use `_fields`.** Full records carry rendered HTML bodies. Requesting only what you need cuts
  payloads by an order of magnitude.
- **Do not retry writes.** There is no idempotency contract on this API — but you should not be
  writing at all: reads are the entire supported anonymous surface.
- **Branch on `code`, not on status.** Errors return
  `{code, message, data:{status}}` on `application/json` — this is the WordPress envelope, not
  RFC 9457 problem+json. See `errors/cellarity-problem-types.yml`.
- **`ext` is not Cellarity's voice.** Items tagged `ext` are external coverage. Do not attribute
  their claims to Cellarity.
- **No rate-limit headers are published.** Back off on 429/503 and keep concurrency low.
