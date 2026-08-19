# Schweppes — project guide

Blutui project. Canvas templating, Canopy blocks, Tailwind CSS v4, Parcel.
Built from Figma designs. Handle: `schweppes`.

Generic Blutui rules live in the `blutui-project-guidelines` and
`blutui-canopy-blocks` skills — load them for anything not covered here.
This file covers what is specific to **this** project.

## Working agreement

- **`courier dev` and `npm run dev` are already running.** Never start them, never
  run `npm run build`, never start a dev server on another port.
- **Do not verify by default.** No curling the site, no headless browser, no
  screenshots. Write the files and report. Verify only when explicitly asked, or
  when a change is genuinely unverifiable by reading (then say so first).
- **Build from the design, never from history.** The Figma frame and the blocks in
  `views/canopy/` are the only sources for new page code. Do not reconstruct a
  missing file with `git show`, `git log -p`, `git checkout <ref> -- <path>`,
  `git stash`, or the reflog, and do not go looking for a deleted version to copy.
  A file that is absent is meant to be written fresh. If you believe an older
  revision is genuinely the right answer, say so and ask first.
- Batch independent work — pull all Figma sections in one message, then write all
  files in one pass.

## File structure

```
views/
  templates/default.html      HTML shell: fonts, canopy.head/scripts, header + footer
  layouts/                    one file per page type; the only way to make a page
  components/                 header, footer, story-section (shared markup)
  canopy/                     one .canvas block template per file
    <parent>/                 child blocks for a nested parent
public/images/                design assets (moving to the Blutui asset library later)
src/styles/global.css         Tailwind entry + @theme tokens
```

Never create a `pages/` directory.

## Writing a layout

A layout is the only way to make a page. Every layout in this project is the same
shape — a block area listing the sections of the design, top to bottom:

```canvas
{% extends 'templates/default.html' %}

{% block body %}
  {{ canopy.blocks('main', [
    'block_a',
    'block_b'
  ]) }}
{% endblock %}
```

- **The area handle must be `'main'`.** `templates/default.html` hardcodes
  `canopy.head('main')` and `canopy.scripts('main')`. Any other handle still
  renders the blocks but silently drops their CSS and JS.
- **Always pass the explicit template array.** It fixes the section order and makes
  the page render complete from block defaults before an editor touches it.
- `{% block body %}` only — the template already supplies `<html>`, `<head>`, the
  font links, `global.css`, the skip link, `components/header.html`,
  `<main id="main">` and `components/footer.html`. A layout that repeats any of
  those is wrong.
- No `<section>` wrappers, no page gutters, no headings in the layout. Each block
  owns its own `<section>` and `.shell`. A layout with markup in it means a block
  is missing a setting.
- A name repeated in the array resolves to the same stored content — if a design
  repeats a section with different copy, make distinct blocks sharing a component
  (see `story-section.html` and its four wrappers).

## Canopy blocks

**The rule that bites:** for any block with children, the **file name, folder name
and config `name` must be identical** — `nutrition.canvas` + `nutrition/` +
`"name": "nutrition"`. A mismatch makes `canopy.children()` render nothing, with no
error anywhere. Same for the `allow` keys (`nutrition/row`).

- Config must be strict JSON. A missing or misspelled `name`/`type` is silently
  ignored — the field just never appears in the editor.
- Every setting gets a `default` so a block looks complete the moment it is added.
  Seed child blocks with `children.default`.
- Exactly one plain `{{ canopy.children() }}` per parent, or editors cannot manage
  children at all. `canopy.childList()` is metadata/display only.
- `richtext` needs `|raw`. Guard `url` / `file` / `list` / `entry` values.
- Block areas are **page-scoped** by default. One layout therefore serves many
  pages, each with its own content — a second product page is usually just an MCP
  `create_page` call, no new code.

### Images in blocks

Always this pattern, so blocks render from `public/` now and switch to the asset
library with no template change:

```canvas
{% set img = settings.image.path|default(asset('images/foo.png')) %}
<img src="{{ img | image_url({ width: 1200, format: 'webp' }) }}" alt="…">
```

## Existing blocks — reuse before creating

| Block                                      | Settings                                                                                                                    |
| ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------- |
| `hero`                                     | image, alt, show_scroll_cue — the artwork carries the headline; nothing is overlaid                                          |
| `product_hero`                             | heading, description (richtext), can, backdrop, badge_text, show_badge, show_scroll_cue                                     |
| `products`                                 | anchor, source (products collection), categories, write_review, read_reviews, show_share — pack shots and panels come from the collection, not from children |
| `tasting_notes` → `tasting_notes/note`     | heading → label, lead, body                                                                                                 |
| `nutrition` → `nutrition/row`              | heading, disclaimer, footnote, image, column\_\* → nutrient, serving, hundred                                               |
| `image_band`                               | image, alt, height (short/medium/tall)                                                                                      |
| `story`                                    | lead, emphasis, tail, signature, image, overlay_image, band_image, theme, image_side, media_style, emphasis_size, show_band |
| `recipes`                                  | anchor, display_heading, heading, body, cta, image, background                                                              |
| `heritage`                                 | anchor, display_heading, heading, emphasis, body, image                                                                     |
| `tradition`                                | text, signature, background                                                                                                 |
| `advertising`                              | lead, wordmark, wordmark_text, tail, photo, inset_photo                                                                     |
| `lightning` `warrant` `exhibition` `today` | narrative sections; thin wrappers over `components/story-section.html`                                                      |

`story` is the generic narrative section — reach for it before writing a new block.
Prefer adding a setting to an existing block over creating a near-duplicate.

Read a block's `.canvas` file before assuming what it renders — the settings column
above is an index, not a spec. Blocks already carry the design's copy as defaults,
so a page built only from existing blocks needs no content passed anywhere.

Sections that menu links target (`products`, `recipes`, `heritage`) expose an
`anchor` setting that becomes the section `id`.

## Collections

| Collection | Drives                                                                     |
| ---------- | -------------------------------------------------------------------------- |
| `products` | the `products` showcase block — one entry per can or bottle, 18 seeded across the four categories |

`products` fields: name, category (select — the four showcase tabs), order, pack_shot
(file), pack_shot_file (filename in `public/images`, used until a pack shot is
uploaded), availability, caffeine, serving_size, the nutrition panel numbers
(calories, total_fat(_dv), sodium(_dv), total_carbohydrates(_dv), total_sugars,
added_sugars(_dv), protein), ingredients, nutrition_footnote, daily_value_note,
rating (text, halves allowed), review_count.

- **The API stores `0` as `null`**, so every number is rendered `|default(0)`.
- Product slugs are derived — `product.name|slug` — there is no slug field.
- A `pack_shot_file` containing `bottle` renders taller than a can. That heuristic is
  the only thing telling the two apart; a `pack_size` field would do it properly.
- **Crop pack shots to their content before adding them.** The strip bottom-aligns by
  box, so transparent padding under a pack shot leaves it floating above the others —
  which is exactly how the 2 L bottles arrived from the reference site.
- An entry with no pack shot renders a dashed placeholder rather than the wrong can —
  Club Soda, Tonic Water and the sparkling waters are all waiting on artwork.
- Number fields only accept an integer `step`, which is why `rating` is a text field.
- Collection id `01a01bf6-2482-72aa-a2eb-ef136c231e80`; `create_collection_entry`
  needs that id, not the `products` handle.

## Design tokens (`src/styles/global.css`)

Colours: `ink body heritage plum plum-mist warrant silver footer off-white gold gold-ink olive`
Fonts: `font-sans` (Montserrat), `font-display` (Libre Baskerville)
Helpers: `.shell` (page gutters, 135px at lg), `.pill-cta`, `.scroll-cue`, `.scroll-thin`
(slim on-brand scrollbar for horizontally-scrolling strips)

Build mobile-first: stack below `lg`, match the 1440 comp at `lg` and up.

## Figma → page workflow

1. `mcp__figma-desktop__get_metadata` with no nodeId (uses the current selection),
   then `get_screenshot` on the frame and `get_design_context` on each section —
   batched in one message. The frame's child names are the section list; ignore the
   header and footer frames, the template already renders those.
2. Download new images from the returned `http://localhost:3845/assets/<hash>` URLs
   into `public/images/`, kebab-case names. Run `file` on them: Figma serves JPEGs
   named `.png`, so fix the extension. Reuse images already present — check
   `public/images/` before downloading anything.
3. Map each design section to a block in the table above, reading the `.canvas`
   files to confirm the match. Only write a new `.canvas` for sections nothing
   covers, and prefer a new setting on an existing block over a near-duplicate.
4. Write the layout — see **Writing a layout**. List the mapped block names in
   design order.
5. `list_pages`, then register with MCP `create_page`: layout path relative to
   `views/` (`layouts/product.html`), a `uri`, a `title` and a `description`.
6. Report the mapping — which design section became which block, and what you
   added. Do not verify in a browser unless asked.

Some elements in a frame are siblings of the section frames rather than children of
them (overlapping pack shots, badges). Check the frame's own child list for these —
they belong to whichever section they sit over. Figma cannot export `text-path`
nodes via `get_design_context`; rebuild those as inline SVG `textPath`.

## Blutui MCP

- Run the matching `list_*` before any `create_*`.
- **There is no `update_*` or `delete_*`.** Anything already created — pages, menus,
  their URLs — must be changed by the user in the dashboard. Say so rather than
  creating a duplicate.
- Pass plain text, never HTML entities (`&`, not `&amp;`).
- Navigation comes from `cms.menu()`; never hardcode nav links.

## Current pages

| URL | Layout              |
| --- | ------------------- |
| `/` | `layouts/home.html` |

The product detail page does not exist yet — no layout, no page in the dashboard.
Build it from the Figma frame following the workflow above.
