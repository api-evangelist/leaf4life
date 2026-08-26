---
name: Harvest the LEAF4Life media library
description: Page through the 59 image attachments behind leafforlife.com, pick the right rendition from media_details.sizes, and attribute each asset to the page it belongs to.
api: openapi/leaf4life-media-api-openapi.yml
operations: [listMedia, getMediaItem, listPages]
method: generated
generated: '2026-08-25'
---

# Harvest the LEAF4Life media library

`/wp/v2/media` is the one collection on this deployment with real volume — 59 attachments on
2026-08-25 — and, because page bodies are empty over the API, it is also the only place the
company's visual material is reachable as data rather than as scraped HTML.

## Step 1 — list (`listMedia`)

```
GET https://leafforlife.com/wp-json/wp/v2/media?per_page=100&_fields=id,slug,title,source_url,mime_type,media_details,post,date
```

`X-WP-Total: 59` on 2026-08-25 — 43 `image/png` and 16 `image/jpeg`, uploaded between 2020-09-22 and
2026-07-05. There are **no PDFs, posters, protocols, publications or datasets**. None of LEAF4Life's
scientific material is published as a downloadable asset; if you are looking for a trial protocol or
a poster, it is not here and it is not anywhere else on the site either.

`per_page` maxes at 100, so 59 items is one request. With more, follow the RFC 8288 `Link` header
(`rel="next"`) rather than incrementing `page` yourself.

## Step 2 — pick a rendition

Each item carries `media_details.sizes` with WordPress's generated renditions (`thumbnail`,
`medium`, `large`, `full`, plus theme-registered sizes). Each has its own `source_url`, `width` and
`height`. Take the smallest rendition that meets your need — `source_url` at the top level is the
original upload and several of these are large (the primary logo PNG is ~460 KB).

Brand marks worth knowing by slug:

- `leaf4life_logo_cropped_final` — the current primary logo (uploaded 2025-11)
- `l4l-logo-rectangle`, `l4l-logo-square` — the August 2025 mark set
- `cropped-copy-of-untitled-design-6-1-png` — the source of the site favicon

## Step 3 — attribute each asset (`listPages`)

An attachment's `post` field holds the id of the page it was uploaded to, or `0` when unattached.
Join it against the 8-page index from `listPages` to say which asset belongs to the science page
versus the leadership page. A page's own `featured_media` field points the other way, holding an
attachment id or `0`.

## Step 4 — resolve one item (`getMediaItem`)

```
GET /wp/v2/media/3540
```

Same object, single fetch. A bad id returns `404 rest_post_invalid_id`.

## Cautions

- **Do not fetch the CDN copies.** Page HTML references NitroPack-optimized variants under
  `cdn-ildiemf.nitrocdn.com/...`. Those URLs carry a build revision and rotate. The
  `source_url` from the API is the stable origin URL — use it.
- Media is the freshest signal on this site: the newest upload (2026-07-05) is more recent than the
  newest page modification (2026-02-06). If you are watching this company for activity, poll
  `listMedia` with `?orderby=date&order=desc&per_page=1` rather than the page collection.
- No rate-limit headers are returned. Self-throttle; 59 items is one request, so there is no excuse
  for hammering it.
