# TileView (DataviewJS) — README

A generic, configurable **tile/grid view** for Obsidian Notes powered by **DataviewJS**.  
Supports:

- Folder‑scoped queries
- Checkbox **facet filters** (e.g., `ecosystem`, `type`, `basemodel`)
- Declarative **Boolean filters** (e.g., “Show only latest”, “Hide NSFW”)
- Flexible **badges** with stacked **corner positioning** and ordering
- Image or **MP4 video** thumbnails
- Themeable, minimal CSS with variables

---

## 1) Prerequisites

- **Obsidian** (desktop or mobile)
- **Dataview** plugin installed & enabled (Settings → Community Plugins → Dataview)
- (Optional) A folder of notes with YAML frontmatter fields you want to display (examples below)

---

## 2) Files & Setup

### A. Add the CSS snippet

Create a file at:  
`.obsidian/snippets/tileview.css`

Paste the **TileView CSS** you’re using (the generic styles with `.tv-*` classes).  
Then enable it in Obsidian: **Settings → Appearance → CSS snippets** → toggle `tileview.css` **On**.

> Tip: The CSS supports theming via variables; you can tweak spacing, colors, and more without touching JS.

### B. Add the DataviewJS block

Create or open a note and add a DataviewJS code block with the **TileView script** (the version with `CONFIG` up top, boolean filters, and stacked badges). Example header for the block:

````
```dataviewjs
// (TileView script goes here)
```
````

---

## 3) Configure `CONFIG` (all knobs up top)

All behavior is controlled by the `CONFIG` object at the top of the script.

```js
const CONFIG = {
  sourcePath: "Visual/LoRAs/loras",
  fields: {
    title:   ["aliases", "title", "name"],
    subtitle:["version", "subtitle"],
    media:   ["image", "cover", "thumbnail"],
    date:    ["published", "date"],
  },
  groupBy: ["title", "basemodel"],
  filters: ["ecosystem", "type", "basemodel"],
  booleanFilters: [
    {
      id: "latestOnly",
      label: "Show only latest in group",
      default: true,
      when: (ctx) => ctx.isLatest === true,
    },
    {
      id: "hideNSFW",
      label: "Hide NSFW",
      default: true,
      when: (ctx) => !ctx.tags.has("nsfw"),
    },
  ],
  badges: [
    { field: "type",      corner: "tr", order: 0, styleMapKey: "type" },
    { field: "basemodel", corner: "tr", order: 1 },
  ],
  cornerStack: { tl: "down", tr: "down", bl: "up", br: "up" },
  badgeGap: 4,
  styleMaps: {
    type: {
      positive: { background: "#2196f3", color: "#fff" },
      negative: { background: "#f44336", color: "#fff" },
    },
  },
  autoplayVideo: true,
  fallbackMedia: "fallback.png",
  tileSize: { width: 300, height: 225 },
};
```

---

## 4) Using TileView

1. Open the note containing the DataviewJS block.
2. The **Filter Bar** appears at the top.
3. Click a **tile** to open the underlying note.
4. Videos autoplay if `autoplayVideo: true`.

---

## 5) Pagination & Performance

TileView includes **infinite scroll pagination** to handle large collections efficiently, especially on mobile devices.

### Configuration

Adjust pagination settings in your CSS snippet:

```css
:root {
  --tv-tiles-per-page: 20;    /* Number of tiles per page (adjust for device) */
  --tv-scroll-threshold: 200; /* Pixels from edge to trigger loading */
}
```

### How It Works

- **Virtual Scrolling**: Only 2 pages maximum are rendered at once (current + 1 adjacent)
- **Infinite Scroll**: Pages load automatically as you scroll
- **Memory Management**: Previous pages are removed when scrolling forward
- **Seamless Experience**: Content appears continuously without pagination buttons

### Device Optimization

**Mobile devices** should use smaller page sizes:
```css
--tv-tiles-per-page: 10;  /* Fewer tiles for mobile */
```

**Desktop** can handle larger pages:
```css
--tv-tiles-per-page: 50;  /* More tiles for desktop */
```

---

## 6) Theming & Styling

In `tileview.css`, adjust variables:

```css
:root {
  --tv-gap: 16px;
  --tv-card-radius: 12px;
  --tv-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
  --tv-footer-bg: rgba(0, 0, 0, 0.7);
  --tv-footer-fg: #fff;
  --tv-footer-muted: #ccc;
  --tv-badge-bg: #444;
  --tv-badge-fg: #fff;
  --tv-accent: #2a7ae2;
}
```

---

## 7) Examples

### A. LoRA Catalog
```yaml
aliases: ["Wan 2.1 FusionX LoRA"]
ecosystem: wan
type: style
basemodel: wan-2.1
version: Image2Video
published: 2025-06-14
image: Visual/LoRAs/images/1678575@1900322.mp4
tags: [lora, i2v, nsfw]
```

### B. Recipes Grid
```yaml
title: "Szechuan Soba Salad"
category: "Salad"
diet: ["Vegetarian", "Dairy-free"]
date: 2025-08-01
image: Recipes/images/SzechuanSobaSalad.png
tags: [recipe, cold]
```

---

## 8) Common Customizations

- New boolean filter:
  ```js
  { id: "hideDraft", label: "Hide Draft", default: true, when: (ctx) => !ctx.tags.has("draft") }
  ```
- Change tile size: `tileSize: { width: 360, height: 240 }`
- Badge coloring: see `styleMaps`

---

## 9) Troubleshooting

- **No tiles show** → check `sourcePath` and YAML fields.
- **Badges not appearing** → ensure field exists or use `label(page)`.
- **Filters odd with lists** → fields are joined by default, tokenize if needed.
- **Video doesn’t autoplay** → some platforms block autoplay; toggle `autoplayVideo: false`.

---

## 10) How It Works

- Groups by `groupBy` → picks latest per group.
- Collects unique values from `filters` → renders checkboxes.
- Applies **facet** filters (datasets) + **boolean** filters (ctx logic).
- **Pagination**: Splits items into pages, renders 2 pages max, infinite scroll.
- Builds tiles with image/video, badges, footer.

---

## 11) Upgrading From Older Versions

- Class names unified as `.tv-*`
- Config centralized in `CONFIG`
- Badges support **corner stacking** and **ordering**
- Boolean filters are declarative
- **NEW**: Pagination system for large collections

### Pagination Migration

If upgrading from a previous version, the pagination system is **automatically enabled**. You can:

1. **Adjust page size** by setting `--tv-tiles-per-page` in your CSS
2. **Disable pagination** by setting `--tv-tiles-per-page: 999999` (render all at once)
3. **Tune performance** with `--tv-scroll-threshold` (lower = more responsive, higher = less CPU usage)

