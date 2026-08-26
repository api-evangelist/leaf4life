---
name: Describe the LEAF4Life API deployment before calling it
description: Read the route index, post types, taxonomies and statuses to learn the shape of the surface, and — critically — establish which routes are open and which return 401, before writing any integration.
api: openapi/leaf4life-discovery-api-openapi.yml
operations: [listTypes, listTaxonomies, listStatuses, getOEmbed]
method: generated
generated: '2026-08-25'
---

# Describe the LEAF4Life API deployment before calling it

There is no developer portal, no API reference and no OpenAPI published by LEAF4Life. The deployment
describes itself instead. Start here rather than assuming a stock WordPress shape — the interesting
facts about this one are all about what is *closed*.

## Step 1 — the route index

```
GET https://leafforlife.com/wp-json/
```

Returns `name` ("LEAF4life"), `description`, `url`, `home`, `namespaces`, all 150 `routes` with their
`args`, and an `authentication` block. This single document is the contract; the OpenAPI files in
this repository are a derivation of it.

Namespaces registered on 2026-08-25: `oembed/1.0`, `akismet/v1`, `wp-super-cache/v1`, `wp/v2`,
`wp-site-health/v1`, `wp-block-editor/v1`, `wp-abilities/v1`.

## Step 2 — what is actually open

Do not infer reachability from registration. Measured anonymously on 2026-08-25:

**Open (HTTP 200):** `/wp/v2/pages`, `/wp/v2/media`, `/wp/v2/categories`, `/wp/v2/tags`,
`/wp/v2/posts`, `/wp/v2/comments`, `/wp/v2/blocks`, `/wp/v2/navigation`, `/wp/v2/search`,
`/wp/v2/types`, `/wp/v2/taxonomies`, `/wp/v2/statuses`, `/wp/v2/users`, `/oembed/1.0/embed`.

**Closed (HTTP 401):** `/wp/v2/settings`, `/wp/v2/block-types`, `/wp/v2/menu-items`,
`/wp/v2/template-parts`, and every route in `akismet/v1`, `wp-super-cache/v1`, `wp-site-health/v1`
and `wp-abilities/v1`.

That last one matters if you are an agent. `wp-abilities/v1` is WordPress 7.0.3's own **ability
registry** — the nearest thing on this deployment to a native agent-callable tool surface — and it
returns `401 rest_forbidden` to anonymous callers. Its tool schemas cannot be read without site
credentials, and LEAF4Life ships no MCP server either (`/mcp` → 404). There is no agent-callable
action here; only reads.

**Empty but open:** `/wp/v2/posts`, `/wp/v2/comments`, `/wp/v2/blocks` and `/wp/v2/navigation` all
return `X-WP-Total: 0`. An empty 200 is not an error and not a permission problem.

## Step 3 — post types (`listTypes`)

```
GET /wp/v2/types
```

11 registered: `post`, `page`, `attachment`, `nav_menu_item`, `wp_block`, `wp_template`,
`wp_template_part`, `wp_global_styles`, `wp_navigation`, `wp_font_family`, `wp_font_face`. Only
`page` and `attachment` carry published content. Each entry's `rest_base` is the path segment to use —
read it rather than hardcoding, since a plugin update can change the set.

## Step 4 — taxonomies and statuses (`listTaxonomies`, `listStatuses`)

```
GET /wp/v2/taxonomies
GET /wp/v2/statuses
```

4 taxonomies (`category`, `post_tag`, `nav_menu`, `wp_pattern_category`) and, anonymously, one status
(`publish`). Note that the taxonomy terms are unused — 1 category and 21 tags, every one with a count
of 0, and the tag slugs (`animation`, `beard`, `camera`, `sandwich`, `watch`) are theme-demo residue.
**Do not mine them as a company vocabulary.**

## Step 5 — oEmbed (`getOEmbed`)

```
GET /wp-json/oembed/1.0/embed?url=https%3A%2F%2Fleafforlife.com%2F
```

Returns oEmbed 1.0 with `provider_name: "LEAF4life"`. This is the only formally standardised
interface the deployment implements. A URL not on this site returns `404 oembed_invalid_url`.

## What you will not find, and should stop looking for

Probed and 404 on 2026-08-25: `/openapi.json`, `/openapi.yaml`, `/swagger.json`, `/api-docs`,
`/docs`, `/llms.txt`, `/apis.json`, `/graphql`, `/mcp`, `/asyncapi.yaml`,
`/.well-known/security.txt`, `/.well-known/openid-configuration`,
`/.well-known/oauth-authorization-server`, `/.well-known/agent-card.json`,
`/.well-known/agent.json`, `/.well-known/api-catalog`, `/.well-known/ai-plugin.json`.

Every one of those returns the site's themed HTML 404 page — an HTTP 404 with a 47 KB HTML body.
**Check the body, not just the status**, and treat any non-JSON body under this host as an absence.

One provider-side defect worth reporting rather than working around: `robots.txt` advertises
`Sitemap: https://leafforlife.com/wp-sitemap.xml`, and that URL returns 404.
