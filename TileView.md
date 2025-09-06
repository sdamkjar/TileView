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
};
// =================== END CONFIG ===================

// ---------- pagination state ----------
const PAGINATION = {
  tilesPerPage: 20, // default, will be read from CSS
  currentPage: 0,
  visiblePages: new Set([0]), // track which pages are currently rendered
  scrollThreshold: 200, // pixels from edge to trigger loading
  allItems: [], // all filtered items
  pageItems: new Map(), // page number -> items for that page
};

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
const readCSSVariable = (variable) => {
  const value = getComputedStyle(document.documentElement).getPropertyValue(variable).trim();
  return parseFloat(value) || 0;
};

const splitIntoPages = (items, tilesPerPage) => {
  const pages = new Map();
  for (let i = 0; i < items.length; i += tilesPerPage) {
    const pageNum = Math.floor(i / tilesPerPage);
    pages.set(pageNum, items.slice(i, i + tilesPerPage));
  }
  return pages;
};

const createTileElement = async (page) => {
  const title = getFieldValue(page, "title") ?? page.file.name;
  const subtitle = getFieldValue(page, "subtitle") ?? "";
  const mediaPath = getFieldValue(page, "media") ?? CONFIG.fallbackMedia;
  const mediaUrl = await getResource(mediaPath);
  const video = isVideoPath(mediaPath);
  const groupKey = computeGroupKey(page);
  const isLatest = latestByGroup[groupKey]?.file?.path === page.file.path;
  const tagsSet = normalizeTagsToSet(page.tags);
  const hasMedia = !!mediaPath;

  const vault = app.vault.getName();
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
  
  // Store context for filtering
  a._tileContext = { isLatest, tags: tagsSet, hasMedia };

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
  
  return a;
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
  <div class="tv-grid" id="tv-grid"></div>
  <div class="tv-loading" id="tv-loading" style="display:none;text-align:center;padding:1rem;">Loading...</div>
`;



// ---------- pagination setup ----------
// Read pagination config from CSS
PAGINATION.tilesPerPage = readCSSVariable('--tv-tiles-per-page') || 20;
PAGINATION.scrollThreshold = readCSSVariable('--tv-scroll-threshold') || 200;

// Store all items and build page mapping
PAGINATION.allItems = items;

// Initialize with first page
const grid = dv.container.querySelector("#tv-grid");
const loadingEl = dv.container.querySelector("#tv-loading");

// ---------- pagination functions ----------
const renderPage = async (pageNum) => {
  if (PAGINATION.visiblePages.has(pageNum)) return; // already rendered
  
  const pageItems = PAGINATION.pageItems.get(pageNum);
  if (!pageItems || pageItems.length === 0) return;
  
  // Show loading indicator
  loadingEl.style.display = "block";
  
  try {
    // Create page container
    const pageContainer = document.createElement("div");
    pageContainer.className = "tv-page";
    pageContainer.dataset.page = pageNum;
    
    // Render tiles for this page
    const tilePromises = pageItems.map(page => createTileElement(page));
    const tileElements = await Promise.all(tilePromises);
    
    tileElements.forEach(tileElement => {
      pageContainer.appendChild(tileElement);
    });
    
    // Insert page in correct position
    const existingPages = Array.from(grid.querySelectorAll('.tv-page'));
    const insertIndex = existingPages.findIndex(p => parseInt(p.dataset.page) > pageNum);
    
    if (insertIndex === -1) {
      grid.appendChild(pageContainer);
    } else {
      grid.insertBefore(pageContainer, existingPages[insertIndex]);
    }
    
    PAGINATION.visiblePages.add(pageNum);
    console.log(`TileView: Rendered page ${pageNum} with ${pageItems.length} items`);
    
  } catch (error) {
    console.error(`TileView: Error rendering page ${pageNum}:`, error);
  } finally {
    // Hide loading indicator
    loadingEl.style.display = "none";
  }
};

const removePage = (pageNum) => {
  const pageEl = grid.querySelector(`[data-page="${pageNum}"]`);
  if (pageEl) {
    pageEl.remove();
    PAGINATION.visiblePages.delete(pageNum);
  }
};

const updateVisiblePages = async () => {
  const currentPage = PAGINATION.currentPage;
  const maxPage = Math.max(0, PAGINATION.pageItems.size - 1);
  
  // Always keep current page and one adjacent
  const targetPages = new Set([currentPage]);
  if (currentPage > 0) targetPages.add(currentPage - 1);
  if (currentPage < maxPage) targetPages.add(currentPage + 1);
  
  // Remove pages that shouldn't be visible
  for (const visiblePage of PAGINATION.visiblePages) {
    if (!targetPages.has(visiblePage)) {
      removePage(visiblePage);
    }
  }
  
  // Add pages that should be visible
  for (const targetPage of targetPages) {
    if (!PAGINATION.visiblePages.has(targetPage)) {
      await renderPage(targetPage);
    }
  }
};

const handleScroll = (() => {
  let isHandling = false;
  let timeoutId = null;
  
  return () => {
    // Debounce scroll events
    if (timeoutId) clearTimeout(timeoutId);
    
    timeoutId = setTimeout(() => {
      if (isHandling) return;
      isHandling = true;
      
      try {
        // Get the scroll container - could be the preview pane or window
        const scrollContainer = grid.closest('.markdown-preview-view') || grid.closest('.markdown-rendered') || document.documentElement;
        const scrollTop = scrollContainer.scrollTop || 0;
        const scrollHeight = scrollContainer.scrollHeight || 0;
        const clientHeight = scrollContainer.clientHeight || 0;
        
        // Validate scroll values
        if (scrollHeight === 0 || clientHeight === 0) {
          isHandling = false;
          return;
        }
        
        const distanceFromBottom = scrollHeight - (scrollTop + clientHeight);
        const distanceFromTop = scrollTop;
        
        const maxPage = Math.max(0, PAGINATION.pageItems.size - 1);
        const currentPage = PAGINATION.currentPage;
        
        // Check if we need to load next page
        if (distanceFromBottom < PAGINATION.scrollThreshold && currentPage < maxPage) {
          console.log(`TileView: Loading next page (${currentPage + 1}), distance from bottom: ${distanceFromBottom}`);
          PAGINATION.currentPage++;
          updateVisiblePages();
        }
        
        // Check if we need to load previous page  
        else if (distanceFromTop < PAGINATION.scrollThreshold && currentPage > 0) {
          console.log(`TileView: Loading previous page (${currentPage - 1}), distance from top: ${distanceFromTop}`);
          PAGINATION.currentPage--;
          updateVisiblePages();
        }
        
      } catch (error) {
        console.error('TileView: Scroll handling error:', error);
      } finally {
        isHandling = false;
      }
    }, 100); // 100ms debounce
  };
})();

// ---------- filtering with pagination ----------
const applyFiltersWithPagination = () => {
  // Get currently active filters
  const activeFacets = {};
  for (const f of CONFIG.filters) {
    activeFacets[f] = Array.from(document.querySelectorAll(`.tv-filter-${f}:checked`))
      .map(cb => cb.value.toLowerCase());
  }
  const activeBools = CONFIG.booleanFilters.filter(b => document.getElementById(`tv-bool-${b.id}`)?.checked);
  
  // Filter all items
  const filteredItems = PAGINATION.allItems.filter(page => {
    // Check facet filters
    const facetPass = CONFIG.filters.every(f => {
      const raw = page[f];
      const val = Array.isArray(raw) ? raw.join(", ") : (raw ?? "unknown");
      return activeFacets[f].includes(String(val).toLowerCase());
    });
    
    if (!facetPass) return false;
    
    // Check boolean filters
    const groupKey = computeGroupKey(page);
    const isLatest = latestByGroup[groupKey]?.file?.path === page.file.path;
    const tagsSet = normalizeTagsToSet(page.tags);
    const ctx = { isLatest, tags: tagsSet, hasMedia: !!getFieldValue(page, "media") };
    
    const boolPass = activeBools.every(b => {
      try { return b.when(ctx); } catch { return true; }
    });
    
    return boolPass;
  });
  
  // Rebuild pagination with filtered items
  PAGINATION.pageItems = splitIntoPages(filteredItems, PAGINATION.tilesPerPage);
  PAGINATION.currentPage = 0;
  PAGINATION.visiblePages.clear();
  
  // Clear grid and render first page
  grid.innerHTML = "";
  updateVisiblePages();
};

// ---------- initial render ----------
const initializePagination = async () => {
  // Split all items into pages
  PAGINATION.pageItems = splitIntoPages(PAGINATION.allItems, PAGINATION.tilesPerPage);
  
  console.log(`TileView Pagination: ${PAGINATION.allItems.length} items, ${PAGINATION.pageItems.size} pages, ${PAGINATION.tilesPerPage} tiles per page`);
  
  // Render initial pages
  await updateVisiblePages();
  
  // Set up scroll listener - try multiple possible containers
  const scrollContainers = [
    grid.closest('.markdown-preview-view'),
    grid.closest('.markdown-rendered'), 
    grid.closest('.workspace-leaf-content'),
    document.documentElement,
    window
  ].filter(Boolean);
  
  // Add scroll listener to the first valid container
  if (scrollContainers.length > 0) {
    const container = scrollContainers[0];
    container.addEventListener('scroll', handleScroll, { passive: true });
    console.log(`TileView: Scroll listener attached to`, container.className || 'document');
  }
  
  // Set up filter listeners
  CONFIG.filters.forEach(f => 
    document.querySelectorAll(`.tv-filter-${f}`)
      .forEach(cb => cb.addEventListener("change", applyFiltersWithPagination))
  );
  CONFIG.booleanFilters.forEach(b => 
    document.getElementById(`tv-bool-${b.id}`)
      ?.addEventListener("change", applyFiltersWithPagination)
  );
};

// Start the pagination system
initializePagination();



```
