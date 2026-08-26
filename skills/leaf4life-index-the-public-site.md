---
name: Index the LEAF4Life public site
description: Enumerate every published page on leafforlife.com through the WordPress REST API, detect what changed since a prior run, and — because page bodies are empty over the API — fetch the HTML for the pages that actually moved.
api: openapi/leaf4life-pages-api-openapi.yml
operations: [listPages, getPage, listTypes]
method: generated
generated: '2026-08-25'
---

# Index the LEAF4Life public site

LEAF4Life's whole public website is 8 pages, and the WordPress REST API enumerates all of them in a
single call. **Read the caveat in Step 2 before you write any code against this** — the API gives you
identity and timestamps, not text.

## Step 1 — list the pages (`listPages`)

```
GET https://leafforlife.com/wp-json/wp/v2/pages?per_page=100&_fields=id,slug,title,link,date,modified,parent,template
```

No credentials. Read `X-WP-Total` from the response headers to confirm the count (8 on 2026-08-25).
`per_page` is capped at 100 and a larger value is **rejected** with `400 rest_invalid_param`, not
clamped — so never send `per_page=500` as a "give me everything" idiom.

The 8 pages, with their stable ids:

| id | slug | what it is |
|----|------|-----------|
| 137 | `home` | mission, KizaVie™ overview, hypoxia across indications |
| 881 | `our-science` | transcrocetin pharmacology and preclinical model summaries |
| 1236 | `management-team` | leadership, scientific advisors, clinical advisors |
| 2603 | `advisory-board` | advisory board |
| 3182 | `contact-us` | WPForms contact form |
| 2479 | `clet-niyikiza-bio` | founder/CEO biography |
| 2509 | `victor-moyo-bio` | biography |
| 2540 | `xavier-pivot-bio` | biography |

All 8 have `parent: 0` — the hierarchy is flat even though the three bio pages are conceptually
subordinate to `management-team`. Do not build a tree from `parent`; there isn't one.

## Step 2 — the caveat that decides your whole design

`content.rendered` and `excerpt.rendered` are the **empty string on every page**. Verified
2026-08-25. Every page uses the theme template `template-modular.php`, and its body is assembled by a
page builder that stores blocks in postmeta rather than in `post_content`. The REST API never sees it.

Two consequences you must design around:

1. **You cannot read the science, pipeline or leadership narrative through this API.** Fetch the HTML
   at the page's `link` field and parse it.
2. **Search is dead on arrival.** `/wp/v2/search?search=hypoxia`, the same query with
   `subtype=page`, and `/wp/v2/pages?search=hypoxia` all return `X-WP-Total: 0`. WordPress searches
   `post_content`, and `post_content` is empty. The route is live and correct; there is simply no
   indexed text behind it. Do not conclude the term is absent from the site — it is on the home page.

So the honest pattern is: **REST for the index, HTTP for the content.**

## Step 3 — incremental sync (`listPages` with `modified_after`)

```
GET /wp/v2/pages?modified_after=2026-01-01T00:00:00&per_page=100&_fields=id,slug,link,modified
```

`modified_after` and `modified_before` take ISO 8601 and are the cheap way to avoid re-fetching HTML
for pages that have not moved. Store the max `modified` you have seen and pass it next run.

For calibration: the most recently modified page on 2026-08-25 was `management-team` at
`2026-02-06T16:24:04`. The three biography pages have not changed since October 2020. This site moves
a few times a year — polling it more than weekly is wasted work.

## Step 4 — resolve one page (`getPage`)

```
GET /wp/v2/pages/881
```

Returns the same identity fields for a single page. A bad id returns `404 rest_post_invalid_id`;
resolve ids through the collection rather than guessing them.

## Errors you will actually see

| status | code | meaning |
|--------|------|---------|
| 400 | `rest_invalid_param` | a parameter failed validation; read `data.details[param].code` |
| 401 | `rest_forbidden_context` | you asked for `context=edit`; use `view` or `embed` |
| 404 | `rest_post_invalid_id` | no page with that id |
| 404 | `rest_no_route` | the path is not registered |

The envelope is `{code, message, data:{status}}` as `application/json`. It is **not** RFC 9457
problem+json — there is no `type` URI. Branch on `code`, never on `message`.

## Rate limits and manners

None are published and no `RateLimit-*`, `X-RateLimit-*` or `Retry-After` header is returned on any
response. That means you have no runtime signal, so self-throttle: this is a small managed-WordPress
site behind a NitroPack edge cache, and the whole corpus is 8 pages. One pass a week is generous.
API responses carry `Cache-Control: no-cache` and `X-Robots-Tag: noindex`; the HTML pages are cached.
