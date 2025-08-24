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

## 5) Theming & Styling

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

## 6) Examples

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

## 7) Common Customizations

- New boolean filter:
  ```js
  { id: "hideDraft", label: "Hide Draft", default: true, when: (ctx) => !ctx.tags.has("draft") }
  ```
- Change tile size: `tileSize: { width: 360, height: 240 }`
- Badge coloring: see `styleMaps`

---

## 8) Troubleshooting

- **No tiles show** → check `sourcePath` and YAML fields.
- **Badges not appearing** → ensure field exists or use `label(page)`.
- **Filters odd with lists** → fields are joined by default, tokenize if needed.
- **Video doesn’t autoplay** → some platforms block autoplay; toggle `autoplayVideo: false`.

---

## 9) How It Works

- Groups by `groupBy` → picks latest per group.
- Collects unique values from `filters` → renders checkboxes.
- Applies **facet** filters (datasets) + **boolean** filters (ctx logic).
- Builds tiles with image/video, badges, footer.

---

## 10) Upgrading From Older Versions

- Class names unified as `.tv-*`
- Config centralized in `CONFIG`
- Badges support **corner stacking** and **ordering**
- Boolean filters are declarative

