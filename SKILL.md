---
name: wordpress-kadence
description: >
  Use when working on any WordPress site that uses the Kadence theme. Provides
  the correct data structures, API workflows via Mega Agent Bridge, Kadence
  internals, and a repeatable playbook for making precise, verified changes.
  Triggers on: Kadence, WordPress site, WP site, mega.management, header layout,
  theme mods, custom CSS, block content, WP-CLI, WordPress build.
---

# WordPress + Kadence Playbook

You have SSH access and the Mega Agent Bridge plugin installed. Never guess.
Never ask the user to hard refresh. Verify every change yourself before reporting.

---

## 0. First — Get Your Bearings

Before touching anything, run these two calls:

```bash
# 1. Full settings dump
curl -s -H "X-Mega-Bridge-Key: $KEY" "$WP/wp-json/mega-bridge/v1/kadence/settings" | jq

# 2. Rendered HTML of the front page
curl -s -H "X-Mega-Bridge-Key: $KEY" "$WP/wp-json/mega-bridge/v1/render?path=/" | jq '.header_class, .head_classes'
```

Set these env vars at the start of every session:
```bash
WP="https://mega.management"
KEY="$(ssh -i $SSH_KEY -p $SSH_PORT $SSH_USER@$SSH_HOST \
  "wp option get mega_bridge_api_key --path=$WP_PATH")"
```

Credentials are always at: `/home/jon/.openclaw/workspace/config/api-keys.md`

---

## 1. Kadence Data Structures

### The #1 Rule
**Kadence reads theme mods via `get_theme_mod($key)` which returns the raw PHP value from `theme_mods_{stylesheet}` in wp_options.**

`wp theme mod set` stores a **JSON string** — Kadence's `sub_option()` cannot read it. Always set theme mods through the bridge's `/theme-mods/{key}` endpoint or via `wp eval` with a PHP array.

### Header Layout — `header_main_layout`
Controls whether the header is contained or full-width.

**Correct PHP array:**
```php
[
  "layout"      => "left-logo",   // items arrangement
  "itemsLayout" => "left-logo",
  "desktop"     => "contained",   // THIS is what sets contained/full-width
  "tablet"      => "",
  "mobile"      => "",
]
```

**Valid desktop values:** `contained` | `standard` | `fullwidth`

**Set via bridge:**
```bash
curl -s -X POST -H "X-Mega-Bridge-Key: $KEY" \
  -H "Content-Type: application/json" \
  -d '{"value":{"layout":"left-logo","itemsLayout":"left-logo","desktop":"contained","tablet":"","mobile":""}}' \
  "$WP/wp-json/mega-bridge/v1/theme-mods/header_main_layout"
```

**Verify — check rendered HTML for:**
```
site-header-row-layout-contained
```
(Not `site-header-row-layout-` with nothing after the dash.)

### Content Width — `--global-content-width`
Kadence sets this CSS variable from `content_width` in theme settings. If it's unset or misconfigured, `site-header-row-layout-contained` has nothing to constrain to.

Always set explicitly in custom CSS:
```css
:root { --global-content-width: 1200px; }
```

### Header Border — `header_main_bottom_border`
**Correct structure:**
```php
[ [ "width" => 0, "unit" => "px", "style" => "solid", "color" => "#222222", "inheritFromGlobal" => false ] ]
```
Setting `"width" => 0` removes the border line.

### Header Background — `header_main_background`
```php
[ "desktop" => [ "color" => "#0A0A0A" ], "tablet" => [ "color" => "#0A0A0A" ], "mobile" => [ "color" => "#0A0A0A" ] ]
```

---

## 2. The Change Workflow (no guessing)

```
1. GET /render?path=/        →  read current state
2. Identify exact element and class
3. Make the change (bridge API or WP-CLI)
4. Purge LiteSpeed via WP-CLI (see below) — ALWAYS, no exceptions
5. POST /cache/flush         →  flush WP object cache + transients
6. Verify with real browser request (curl with User-Agent, NOT /render)
7. Only then tell the user it's done
```

Never skip steps 4-6. The bridge's /render endpoint fetches internally and can get a cached response. Always verify with a real external curl.

### Cache Purge — Run After Every Single Change
```bash
# Step 1: LiteSpeed full purge (Hostinger)
ssh -i $SSH_KEY -p $SSH_PORT $SSH_USER@$SSH_HOST \
  "wp litespeed-purge all --path=$WP_PATH"

# Step 2: WP object cache + transients via bridge
curl -s -X POST -H "X-Mega-Bridge-Key: $KEY" "$WP/wp-json/mega-bridge/v1/cache/flush"

# Step 3: Verify with a real browser-simulated request
curl -s -A "Mozilla/5.0" "$WP" | grep 'your-selector'
```

### Why both steps?
- `cache/flush` hits WP object cache, transients, and Kadence CSS cache
- `wp litespeed-purge all` hits the LiteSpeed page cache (Hostinger's CDN layer)
- Without LiteSpeed purge, browsers see stale HTML regardless of what's in the DB

---

## 3. Custom CSS

### ⚠️ Critical: `kadence_custom_css` is a DEAD option
Kadence no longer reads from `get_option('kadence_custom_css')`. That option does nothing.
The correct storage is `wp_update_custom_css_post()` / `wp_get_custom_css()` — WordPress core custom CSS.
The bridge's `/kadence/css` endpoints handle this correctly.

### Get current CSS:
```bash
curl -s -H "X-Mega-Bridge-Key: $KEY" "$WP/wp-json/mega-bridge/v1/kadence/css"
```

### Update CSS:
```bash
curl -s -X POST -H "X-Mega-Bridge-Key: $KEY" \
  -H "Content-Type: application/json" \
  -d "{\"css\": \"$(cat your-styles.css | python3 -c 'import json,sys; print(json.dumps(sys.stdin.read()))')\"}" \
  "$WP/wp-json/mega-bridge/v1/kadence/css"
```

### CSS Selector Priority Rules
1. Read the rendered HTML first — identify the actual class names on the element
2. Never fight inline styles with `!important` on generic selectors
3. For inline styles on block content — edit the post content directly (bridge `/posts/{id}`)
4. Use attribute selectors only as last resort: `p[style*="color:#555"]`

### Common Kadence CSS selectors
| Target | Selector |
|--------|----------|
| Gap below header | `#primary.content-area { margin-top: 0 }` |
| Hero section row | `.kb-row-layout-id_row-hero .kb-row-layout-inner-wrap` |
| Stats row | `.kb-row-layout-id_row-stats` |
| Button block | `.wp-block-buttons { margin-top: 32px }` |
| Single button | `.wp-block-button__link` |
| Kadence button | `.kb-button` |

---

## 4. Post Content

### Find the front page post ID:
```bash
curl -s -H "X-Mega-Bridge-Key: $KEY" "$WP/wp-json/mega-bridge/v1/posts/find?path=/"
```

### Read it:
```bash
curl -s -H "X-Mega-Bridge-Key: $KEY" "$WP/wp-json/mega-bridge/v1/posts/5"
```

### Update inline style in block content:
Get the full content, modify the specific style attribute, POST it back.
```bash
# Pattern: find inline color and replace
NEW_CONTENT=$(echo "$CONTENT" | sed 's/color:#555555/color:#FF5500/g')
curl -s -X POST -H "X-Mega-Bridge-Key: $KEY" \
  -H "Content-Type: application/json" \
  -d "{\"content\": $NEW_CONTENT}" \
  "$WP/wp-json/mega-bridge/v1/posts/5"
```

---

## 5. Cache

Flush after every change:
```bash
curl -s -X POST -H "X-Mega-Bridge-Key: $KEY" "$WP/wp-json/mega-bridge/v1/cache/flush"
```

Then verify with `/render` — not by asking the user.

---

## 6. Hostinger Specifics

- **Page cache:** LiteSpeed — purged via `do_action('litespeed_purge_all')` which the bridge calls
- **WP-CLI path:** `/usr/local/bin/wp-cli-2.12.0.phar` aliased as `wp`
- **SSH:** Port 65002, key at `config/keys/mega_hostinger`
- **WP root:** `/home/u616193506/domains/mega.management/public_html`
- **DB prefix:** `wp_`

---

## 7. MEGA Brand & Design Tokens

| Token | Value |
|-------|-------|
| Primary orange | `#FF5500` |
| Orange hover | `#E04A00` |
| Near black bg | `#0A0A0A` |
| Surface | `#111111` |
| Surface 2 | `#1A1A1A` |
| Text primary | `#F0F0F0` |
| Text secondary | `#888888` |
| Border | `#222222` |
| Heading font | Barlow Condensed 700 |
| Body font | Inter 400 |
| Content width | 1200px |
