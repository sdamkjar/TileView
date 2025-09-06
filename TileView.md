```dataviewjs

/**
 * TileView
 */

// ===================== CONFIG =====================
const CONFIG = {
  sourcePath: "example/loras",

  fields: {
    title: ["aliases", "title", "name"],
    subtitle: ["version", "subtitle"],
    media: ["image", "cover", "thumbnail"],
    date: ["published", "date"],
  },

  groupBy: ["title", "basemodel"],
  filters: ["ecosystem", "type", "basemodel"],

  // --- Badge system ---
  // corner: 'tl' | 'tr' | 'bl' | 'br'
  // order: smaller = closer to the corner edge
  // label(page): optional custom text; fallback to the field value
  badges: [
    { field: "type",      corner: "tr", order: 0, styleMapKey: "type" },
    { field: "basemodel", corner: "tr", order: 1 },
    // Example custom label:
    // { field: "ecosystem", corner: "tl", order: 0, label: (p) => `Eco: ${p.ecosystem}` }
  ],

  // How each corner stacks: 'down' (from top corners) or 'up' (from bottom corners)
  cornerStack: { tl: "down", tr: "down", bl: "up", br: "up" },

  // Spacing between badges
  badgeGap: 4,

  // Style maps for value‑based coloring (substring match)
  styleMaps: {
    type: {
      positive: { background: "#2196f3", color: "#fff" },
      negative: { background: "#f44336", color: "#fff" },
    },
  },

  // Boolean filters
  booleanFilters: [
    { id: "latestOnly", label: "Show only latest in group", default: true, when: (ctx) => ctx.isLatest === true },
    { id: "hideNSFW",   label: "Hide NSFW",                  default: true, when: (ctx) => !ctx.tags.has("nsfw") },
  ],

  autoplayVideo: true,
  fallbackMedia: "fallback.png",
  tileSize: { width: 300, height: 225 },

  // Pagination settings
  pagination: {
    enabled: true,
    tilesPerPageCSSVar: "--tv-tiles-per-page", // CSS variable name
    defaultTilesPerPage: 20, // Fallback if CSS var not available
  },
};
// =================== END CONFIG ===================



// ---------- helpers ----------
const getFirstTruthy = (obj, keys) => {
  const list = Array.isArray(keys) ? keys : [keys];
  for (const key of list) {
    const v = obj?.[key];
    if (Array.isArray(v) && v.length) return v[0];
    if (v !== undefined && v !== null && String(v).trim() !== "") return v;
  }
  return undefined;
};
const getFieldValue = (page, logicalKey) => {
  const map = CONFIG.fields?.[logicalKey];
  return map ? getFirstTruthy(page, map) : page[logicalKey];
};
const toDateSafe = (page) => {
  const raw = getFieldValue(page, "date") ?? page.file?.ctime;
  const d = new Date(raw);
  return isNaN(d) ? new Date(page.file?.ctime) : d;
};
const getResource = async (path) => {
  if (!path) return CONFIG.fallbackMedia;
  const f = app.vault.getAbstractFileByPath(path);
  if (!f) return CONFIG.fallbackMedia;
  return await app.vault.adapter.getResourcePath(path);
};
const isVideoPath = (path) => typeof path === "string" && path.toLowerCase().endsWith(".mp4");
const computeGroupKey = (page) => {
  const parts = CONFIG.groupBy.map((k) => (CONFIG.fields[k] ? getFieldValue(page, k) : page[k]) ?? "unknown");
  return parts.join("||");
};
const styleForValue = (styleMap, value) => {
  if (!value || !styleMap) return {};
  const v = String(value).toLowerCase();
  for (const [needle, style] of Object.entries(styleMap)) {
    if (v.includes(needle.toLowerCase())) return style;
  }
  return {};
};
const ensureHtmlSafe = (s) => String(s ?? "").replace(/"/g, "&quot;");
const normalizeTagsToSet = (tags) => {
  if (!tags) return new Set();
  if (Array.isArray(tags)) return new Set(tags.map((t) => String(t).toLowerCase()));
  if (typeof tags === "string") return new Set(tags.split(/[,\s]+/).filter(Boolean).map((t) => t.toLowerCase()));
  return new Set();
};

// ---------- pagination helpers ----------
const getTilesPerPage = () => {
  if (!CONFIG.pagination.enabled) return Number.MAX_SAFE_INTEGER;
  
  // Get value from CSS custom property
  const cssValue = getComputedStyle(document.documentElement)
    .getPropertyValue(CONFIG.pagination.tilesPerPageCSSVar)
    .trim();
  
  const parsed = parseInt(cssValue, 10);
  return isNaN(parsed) || parsed <= 0 
    ? CONFIG.pagination.defaultTilesPerPage 
    : parsed;
};

const createPaginationState = (totalItems) => {
  const tilesPerPage = getTilesPerPage();
  const totalPages = Math.ceil(totalItems / tilesPerPage);
  
  return {
    currentPage: 1,
    tilesPerPage,
    totalPages,
    totalItems,
    getPageItems: (items) => {
      const start = (this.currentPage - 1) * this.tilesPerPage;
      const end = start + this.tilesPerPage;
      return items.slice(start, end);
    },
    setPage: function(page) {
      this.currentPage = Math.max(1, Math.min(page, this.totalPages));
    },
    nextPage: function() { 
      this.setPage(this.currentPage + 1); 
    },
    prevPage: function() { 
      this.setPage(this.currentPage - 1); 
    },
    hasNext: function() { 
      return this.currentPage < this.totalPages; 
    },
    hasPrev: function() { 
      return this.currentPage > 1; 
    },
  };
};



// ---------- load & organize ----------
const allPages = dv.pages(`"${CONFIG.sourcePath}"`);
const groups = {};
for (const page of allPages) (groups[computeGroupKey(page)] ??= []).push(page);
const latestByGroup = {};
for (const key of Object.keys(groups)) {
  latestByGroup[key] = groups[key].slice().sort((a, b) => toDateSafe(b) - toDateSafe(a))[0];
}
const items = Object.values(groups).flat();



// ---------- collect facet values ----------
const filterValues = {};
for (const field of CONFIG.filters) {
  filterValues[field] = [...new Set(items.map(p => {
    const v = p[field];
    return Array.isArray(v) ? v.join(", ") : v;
  }).filter(Boolean))].sort((a, b) => String(a).localeCompare(String(b)));
}



// ---------- UI ----------
dv.container.innerHTML = `
  <div class="tv-filterbar" id="tv-filterbar">
    ${CONFIG.filters.map(field => `
      <div class="tv-filtergroup">
        <strong>${field[0].toUpperCase() + field.slice(1)}:</strong>
        <div class="tv-filteroptions">
          ${filterValues[field].map(value => `
            <label class="tv-filteroption">
              <input type="checkbox" class="tv-filter-${field}" value="${ensureHtmlSafe(value)}" checked>
              ${ensureHtmlSafe(value)}
            </label>
          `).join("")}
        </div>
      </div>
    `).join("")}
    <div class="tv-filtergroup">
      <strong>Options:</strong>
      <div class="tv-filteroptions">
        ${CONFIG.booleanFilters.map(b => `
          <label class="tv-filteroption">
            <input type="checkbox" id="tv-bool-${b.id}" ${b.default ? "checked" : ""}>
            ${ensureHtmlSafe(b.label)}
          </label>
        `).join("")}
      </div>
    </div>
  </div>
  <div class="tv-grid"></div>
  ${CONFIG.pagination.enabled ? `
    <div class="tv-pagination">
      <button class="tv-pagination-btn" id="tv-prev-btn" type="button">Previous</button>
      <div class="tv-pagination-pages" id="tv-pagination-pages"></div>
      <button class="tv-pagination-btn" id="tv-next-btn" type="button">Next</button>
      <div class="tv-pagination-info" id="tv-pagination-info"></div>
    </div>
  ` : ''}
`;



// ---------- render ----------
const grid = dv.container.querySelector(".tv-grid");
const vault = app.vault.getName();
const cardCtx = new WeakMap();
const allCards = []; // Store all cards for pagination

// Initialize pagination state
let state = createPaginationState(items.length);

for (const page of items) {
  const title = getFieldValue(page, "title") ?? page.file.name;
  const subtitle = getFieldValue(page, "subtitle") ?? "";
  const mediaPath = getFieldValue(page, "media") ?? CONFIG.fallbackMedia;
  const mediaUrl = await getResource(mediaPath);
  const video = isVideoPath(mediaPath);
  const groupKey = computeGroupKey(page);
  const isLatest = latestByGroup[groupKey]?.file?.path === page.file.path;
  const tagsSet = normalizeTagsToSet(page.tags);
  const hasMedia = !!mediaPath;

  const fileUrl = `obsidian://open?vault=${encodeURIComponent(vault)}&file=${encodeURIComponent(page.file.path)}`;
  const a = document.createElement("a");
  a.className = "tv-cardlink";
  a.href = fileUrl;

  // facet dataset
  for (const field of CONFIG.filters) {
    const raw = page[field];
    const val = Array.isArray(raw) ? raw.join(", ") : (raw ?? "unknown");
    a.dataset[field] = String(val).toLowerCase();
  }
  cardCtx.set(a, { isLatest, tags: tagsSet, hasMedia });

  // --- build badges grouped by corner ---
  const byCorner = { tl: [], tr: [], bl: [], br: [] };
  for (const b of CONFIG.badges) {
    const rawVal = (b.field in page) ? page[b.field] : getFieldValue(page, b.field);
    if (rawVal == null || rawVal === "") continue;
    const val = Array.isArray(rawVal) ? rawVal.join(", ") : rawVal;
    const label = b.label ? b.label(page) : String(val).toUpperCase();
    const map = CONFIG.styleMaps?.[b.styleMapKey ?? b.field];
    const style = styleForValue(map, val);
    byCorner[b.corner].push({ order: b.order ?? 0, label, style });
  }
  // sort each corner by order (asc: nearer edge first)
  for (const k of Object.keys(byCorner)) byCorner[k].sort((a, b) => (a.order - b.order));

  const cornerHtml = (corner) => {
    const items = byCorner[corner];
    if (!items.length) return "";
    // stacking direction: 'down' => column; 'up' => column-reverse
    const direction = CONFIG.cornerStack[corner] === "up" ? "column-reverse" : "column";
    return `
      <div class="tv-badges-corner tv-badges-${corner}" style="flex-direction:${direction};row-gap:${CONFIG.badgeGap}px;">
        ${items.map(it => `
          <div class="tv-badge" style="${it.style.background ? `background:${it.style.background};` : ""}${it.style.color ? `color:${it.style.color};` : ""}">${ensureHtmlSafe(it.label)}</div>
        `).join("")}
      </div>
    `;
  };

  a.innerHTML = `
    <div class="tv-card" style="width:${CONFIG.tileSize.width}px;height:${CONFIG.tileSize.height}px;">
      <div class="tv-media" style="background-image:url('${mediaUrl}');">
        ${video
          ? `<video src="${mediaUrl}" ${CONFIG.autoplayVideo ? "autoplay" : ""} loop muted playsinline></video>`
          : ""}
        <div class="tv-badges">
          ${cornerHtml("tl")}
          ${cornerHtml("tr")}
          ${cornerHtml("bl")}
          ${cornerHtml("br")}
        </div>
      </div>
      <div class="tv-footer">
        <span class="tv-title">${ensureHtmlSafe(title)}</span>
        ${subtitle ? `<span class="tv-subtitle">${ensureHtmlSafe(subtitle)}</span>` : ""}
      </div>
    </div>
  `;
  
  // Store card instead of immediately appending
  allCards.push(a);
}



// ---------- pagination functions ----------
function updatePaginationUI() {
  if (!CONFIG.pagination.enabled) return;
  
  const prevBtn = document.getElementById("tv-prev-btn");
  const nextBtn = document.getElementById("tv-next-btn");
  const pagesContainer = document.getElementById("tv-pagination-pages");
  const infoElement = document.getElementById("tv-pagination-info");
  
  if (!prevBtn || !nextBtn || !pagesContainer || !infoElement) return;
  
  // Update buttons
  prevBtn.disabled = !state.hasPrev();
  nextBtn.disabled = !state.hasNext();
  
  // Update page numbers (show max 5 pages around current)
  const maxPageButtons = 5;
  const startPage = Math.max(1, state.currentPage - Math.floor(maxPageButtons / 2));
  const endPage = Math.min(state.totalPages, startPage + maxPageButtons - 1);
  
  pagesContainer.innerHTML = '';
  
  for (let i = startPage; i <= endPage; i++) {
    const pageBtn = document.createElement('button');
    pageBtn.className = `tv-pagination-page ${i === state.currentPage ? 'active' : ''}`;
    pageBtn.textContent = i;
    pageBtn.onclick = () => goToPage(i);
    pagesContainer.appendChild(pageBtn);
  }
  
  // Update info
  const startItem = (state.currentPage - 1) * state.tilesPerPage + 1;
  const endItem = Math.min(state.currentPage * state.tilesPerPage, state.totalItems);
  infoElement.textContent = `${startItem}-${endItem} of ${state.totalItems} items`;
}

function renderCurrentPage() {
  // Clear grid
  grid.innerHTML = '';
  
  // Get filtered cards
  const visibleCards = getFilteredCards();
  
  // Update pagination state with current filtered count
  state = createPaginationState(visibleCards.length);
  state.currentPage = Math.min(state.currentPage, state.totalPages || 1);
  
  // Get cards for current page
  const pageCards = state.getPageItems(visibleCards);
  
  // Append cards to grid
  pageCards.forEach(card => grid.appendChild(card));
  
  // Update pagination UI
  updatePaginationUI();
}

function getFilteredCards() {
  const activeFacets = {};
  for (const f of CONFIG.filters) {
    activeFacets[f] = Array.from(document.querySelectorAll(`.tv-filter-${f}:checked`))
      .map(cb => cb.value.toLowerCase());
  }
  const activeBools = CONFIG.booleanFilters.filter(b => document.getElementById(`tv-bool-${b.id}`)?.checked);
  
  return allCards.filter(card => {
    const facetPass = CONFIG.filters.every(f => activeFacets[f].includes(card.dataset[f]));
    const ctx = cardCtx.get(card) || {};
    const boolPass = activeBools.every(b => { try { return b.when(ctx); } catch { return true; } });
    return facetPass && boolPass;
  });
}

function goToPage(page) {
  state.setPage(page);
  renderCurrentPage();
}

function setupPaginationEventListeners() {
  if (!CONFIG.pagination.enabled) return;
  
  const prevBtn = document.getElementById("tv-prev-btn");
  const nextBtn = document.getElementById("tv-next-btn");
  
  if (prevBtn) prevBtn.addEventListener("click", () => {
    state.prevPage();
    renderCurrentPage();
  });
  
  if (nextBtn) nextBtn.addEventListener("click", () => {
    state.nextPage();
    renderCurrentPage();
  });
}

// ---------- filtering ----------
function applyFilters() {
  // Reset to page 1 when filters change
  state.currentPage = 1;
  renderCurrentPage();
}

CONFIG.filters.forEach(f => document.querySelectorAll(`.tv-filter-${f}`).forEach(cb => cb.addEventListener("change", applyFilters)));
CONFIG.booleanFilters.forEach(b => document.getElementById(`tv-bool-${b.id}`)?.addEventListener("change", applyFilters));

// Initial setup
setupPaginationEventListeners();
renderCurrentPage();



```
