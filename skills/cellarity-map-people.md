---
name: Map Cellarity leadership, board and event speakers
description: Build a roster of Cellarity's management, board, founders and symposium speakers from the cellarity.com content API, with the PII discipline that applies to profile data.
api: openapi/cellarity-content-openapi.yml
base_url: https://cellarity.com/wp-json/wp/v2
operations:
  - listTeamCategories
  - listTeamMembers
  - getTeamMember
  - listEventSpeakers
  - getEventSpeaker
  - listUsers
generated: '2026-08-09'
method: generated
source: openapi/cellarity-content-openapi.yml + data-model/cellarity-data-model.yml
---

# Map Cellarity leadership, board and event speakers

Cellarity publishes people as two custom post types: `team-member` (19 records) and
`event-speaker` (11 records), as of 2026-08-09.

## Authentication

None. All operations here are anonymous.

## Step 1 — resolve the team taxonomy

    listTeamCategories
    GET /wp/v2/team-category?per_page=100&_fields=id,slug,name,count

Observed terms: `management` (10), `board` (7), `founders` (4). Counts total 21 against 19
members — individuals hold more than one term, so a person can be both a founder and on the
board. Treat membership as many-to-many; do not bucket each person once.

## Step 2 — list team members

    listTeamMembers
    GET /wp/v2/team-member?per_page=100&_embed=1&_fields=id,slug,link,title,content,featured_media,team-category

Filter to a group using the integer term ID from step 1:

    GET /wp/v2/team-member?team-category=<board_term_id>&per_page=100

`title.rendered` is the person's name. Role and bio are in `content.rendered` as HTML — there are
no typed `role` or `title` fields on this post type.

## Step 3 — read one profile

    getTeamMember
    GET /wp/v2/team-member/{id}

Or by slug: `GET /wp/v2/team-member?slug=ornella-barrandon`.

## Step 4 — event speakers

Speakers featured on the symposium/events pages are a separate type with no taxonomy.

    listEventSpeakers
    GET /wp/v2/event-speaker?per_page=100&_embed=1&_fields=id,slug,link,title,content,featured_media

    getEventSpeaker
    GET /wp/v2/event-speaker/{id}

Speakers are **not** necessarily Cellarity employees — the type covers external presenters. Do not
merge this list into the staff roster without checking each bio.

## Step 5 — headshots

Use `_embed=1` on steps 2 and 4 and read `_embedded['wp:featuredmedia'][0].source_url` plus
`alt_text`. Avoid a separate `getMediaItem` call per person.

## Rules

- **PII discipline.** These are real named people. Publish only what Cellarity already publishes:
  name, role, bio, headshot, and the `link` to their own profile page. Do not enrich these records
  against other sources, do not infer or construct email addresses, and do not build a contactable
  dataset from them.
- **`listUsers` is not the staff directory.** `/wp/v2/users` returns 3 anonymously readable
  *authoring accounts* (one is the generic "Cellarity" account whose `url` leaks the WP Engine
  origin `cellaritypro.wpenginepowered.com`). It is CMS plumbing, not people. Do not present it as
  a roster.
- **No emails are exposed** on any of these endpoints, and none should be added.
- **Rosters go stale.** `team-member` records were last modified 2026-07-13. Re-pull rather than
  cache indefinitely, and date-stamp anything you publish.
- **Content fields are `{rendered: ...}` HTML** with entities. Decode before display.
- **Respect the crawl-delay.** `robots.txt` sets `Crawl-delay: 10`. Even though the API is not
  disallowed, keep request rates polite; no rate-limit headers are published, so you get no
  warning before an edge block.
