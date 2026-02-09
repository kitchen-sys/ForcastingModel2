# Research Findings 04: UI/UX Design & Data Visualization

## Executive Summary

This document presents comprehensive research findings for building an intelligence-grade dashboard UI for a conflict forecasting platform. The design targets the visual quality and information density of platforms like Palantir Gotham, Recorded Future, and Bloomberg Terminal, implemented with modern React libraries and a dark-theme design system optimized for extended analyst use.

---

## 1. Intelligence Dashboard Design Analysis

### 1.1 Palantir Gotham Design Patterns

Palantir Gotham is the "Operating System for Defense Decision Making." Key design observations:

- **Dark-theme interface**: Standard across defense/intelligence dashboards for reduced eye strain during extended sessions and low-light environments
- **Configurable Object Views (COVs)**: Widget-based panels showing entity-specific information; customizable per user without coding
- **Multi-modal visualization**: Simultaneous geospatial mapping, network analysis, and mixed-reality views for full situational awareness
- **Granular access controls**: UI reflects classification levels, need-to-know restrictions, and time-bound access windows
- **Edge-ready**: Interfaces function in disconnected, distributed environments with graceful degradation
- **AIP Integration (2024+)**: LLM-powered interfaces with GPT-4 on classified networks via Microsoft partnership

**Key Takeaway**: Gotham's strength is its widget-based, configurable multi-panel layout where analysts compose their own views from reusable visualization components.

### 1.2 Bloomberg Terminal Design Lessons

Bloomberg Terminal represents the gold standard for information density:

- **No wasted whitespace**: Every pixel serves a purpose; crowded screens spanning 4+ panels generate $6.3B/year in revenue
- **Concealed complexity**: Thousands of functions available but hidden until needed; "We're hiding complexity" (Bloomberg CTO Shawn Edwards)
- **Modern evolution**: Moved from fixed 4-panel maximum to dynamic tabbed panel model with arbitrary window counts
- **Color conventions**: Dark background with bright blue, orange, and amber for non-semantic information; red/blue for up/down market status
- **Keyboard-first**: Specialized keyboard commands for speed; power users resist mouse-driven UIs
- **Status symbol**: The "complex" interface is a feature, not a bug; analysts take pride in mastery
- **HTML5/CSS3/JS stack**: Modern web technologies with hardware GPU acceleration
- **Color accessibility**: Specific CVD (Color Vision Deficiency) schemes for Deuteranopia and Protanomaly

**Key Takeaway**: Maximize information density. Do not over-simplify. Provide keyboard shortcuts and command palette for power users. Treat visual complexity as a feature for professional analysts.

---

## 2. Dashboard Layout Wireframes

### Layout A: Map-Centric (Primary View)

```
+------------------------------------------------------------------+
| [Logo] Command Palette (Cmd+K)  [Alerts 3] [User] [Settings]    |
+------------------------------------------------------------------+
| Filters: [Region v] [Conflict Type v] [Date Range] [Severity v]  |
+----------+---------------------------------------+---------------+
|          |                                       |               |
| LEFT     |         CENTER MAP                    | RIGHT         |
| SIDEBAR  |    (Mapbox GL + Deck.gl overlay)      | SIDEBAR       |
| 280px    |                                       | 320px         |
|          |    [Heatmap] [Markers] [Zones]        |               |
| - Nav    |    [Temporal Slider ====o=======]      | - Entity      |
| - Saved  |                                       |   Detail      |
|   Views  |    Zoom: + -  Layers: [S][T][P][M]    | - Related     |
| - Quick  |                                       |   Events      |
|   Stats  |                                       | - Risk        |
| - Tree   |                                       |   Score       |
|   Filter |                                       | - Sparklines  |
|          |                                       |               |
+----------+-------------------+-------------------+---------------+
|    LIVE EVENT FEED           |   TREND CHART (Area/Line)         |
|    (Scrolling ticker)        |   (Conflict intensity over time)  |
|    [New] Attack in Region... |   [1D] [1W] [1M] [3M] [1Y]      |
+------------------------------+-----------------------------------+
```

**Specifications**:
- Left sidebar: 280px fixed, collapsible to icon-only (48px)
- Center map: Flex-grow, minimum 600px width
- Right sidebar: 320px fixed, collapsible, context-sensitive (shows entity detail on selection)
- Bottom panel: 200px height, resizable, split between live feed and trend chart
- Temporal slider: Full-width scrubber for time-based playback of conflict events

### Layout B: Analytics / Dashboard View (Configurable Widgets)

```
+------------------------------------------------------------------+
| [Logo] Command Palette (Cmd+K)  [Alerts 3] [User] [Settings]    |
+------------------------------------------------------------------+
| Tabs: [Overview] [Regional] [Actors] [Forecasts] [Custom +]     |
+------------------------------------------------------------------+
|                                                                  |
|  +------------------+ +------------------+ +------------------+  |
|  | KPI Card         | | KPI Card         | | KPI Card         |  |
|  | Active Conflicts | | Fatalities (30d) | | Risk Score       |  |
|  | 47  (+3)  ^^^^   | | 12,450  (-8%) vv | | 7.8/10  ====o   |  |
|  +------------------+ +------------------+ +------------------+  |
|                                                                  |
|  +---------------------------+ +-------------------------------+ |
|  | CONFLICT TRENDS           | | ACTOR NETWORK GRAPH           | |
|  | (ECharts area chart)      | | (Cytoscape.js force layout)   | |
|  | [Line] [Bar] [Stacked]    | |  O---O    O                   | |
|  |     /\    /\              | |  |   |   /|\                  | |
|  |    /  \  /  \             | |  O   O--O   O                 | |
|  |   /    \/    \___         | |     \|/                       | |
|  +---------------------------+ +-------------------------------+ |
|                                                                  |
|  +---------------------------+ +-------------------------------+ |
|  | REGIONAL BREAKDOWN        | | FORECAST TIMELINE             | |
|  | (Treemap / Nivo)          | | (Gantt-style horizontal bars) | |
|  |  [Africa   ] [M.East]    | | Region A: =====>              | |
|  |  [  Asia  ] [Europe]     | | Region B:   ======>           | |
|  |  [Americas] [Other ]     | | Region C:     ===>            | |
|  +---------------------------+ +-------------------------------+ |
|                                                                  |
+------------------------------------------------------------------+
```

**Specifications**:
- All widgets are draggable and resizable via react-grid-layout
- 12-column grid, responsive breakpoints at 1200px, 996px, 768px
- Widget minimum sizes enforced (e.g., KPI card: 2x1, chart: 4x3)
- Layout persistence to localStorage and server-side per user
- "Add Widget" button opens a catalog of available widget types
- Tab system for multiple saved dashboard configurations

### Layout C: Investigation / Entity Focus View

```
+------------------------------------------------------------------+
| [Logo] Command Palette (Cmd+K)  [Alerts 3] [User] [Settings]    |
+------------------------------------------------------------------+
| << Back to Dashboard  |  Entity: "Wagner Group"  [Bookmark] [Share]|
+------------------------------------------------------------------+
|                        |                                          |
|  ENTITY PROFILE        |  NETWORK GRAPH                          |
|  +-----------------+   |  (Cytoscape.js - full panel)             |
|  | Type: Armed Grp |   |                                         |
|  | Status: Active  |   |     O---O        O                      |
|  | Region: Multi   |   |     |   |       /|\                     |
|  | Risk: 9.2/10    |   |     O   O------O   O---O                |
|  | Aliases: [...]   |   |       \   |   /                        |
|  +-----------------+   |        O--O--O                           |
|                        |                                          |
|  TIMELINE              |  [1-hop] [2-hop] [All] [Filter: v]      |
|  (D3.js horizontal)    +------------------------------------------+
|  2022 --|---|--|--      |                                          |
|  2023 ----|-----|--     |  DATA TABLE (AG Grid)                   |
|  2024 ---|---|----      |  [Events] [Relations] [Sources] [Intel]  |
|  2025 --|----|--        |  +------+--------+--------+--------+    |
|                        |  | Date | Event  | Source | Conf.  |    |
|  RELATED EVENTS MAP    |  | 2/1  | Attack | OSINT  | High   |    |
|  (Small Mapbox inset)  |  | 1/28 | Move   | SAT    | Med    |    |
|  [....map....]         |  | 1/15 | Supply | HUMINT | Low    |    |
|                        |  +------+--------+--------+--------+    |
+------------------------------------------------------------------+
```

**Specifications**:
- Two-column layout: Entity profile/timeline (35%) and visualization (65%)
- Network graph takes primary focus with interactive exploration
- Data table with virtual scrolling for event history
- Small inset map for geographic context
- Breadcrumb navigation back to main dashboard
- Entity cross-referencing: click any node in the network to navigate to its profile

---

## 3. Map Visualization Strategy

### 3.1 Library Comparison

| Feature | Mapbox GL JS | Deck.gl | CesiumJS | MapLibre GL JS |
|---------|-------------|---------|----------|----------------|
| **Primary Strength** | Styled 2D/2.5D vector maps | GPU-powered large datasets | True 3D globe & terrain | Open-source Mapbox fork |
| **npm Downloads/wk** | ~1.87M | ~119K | ~84K | ~600K+ |
| **GitHub Stars** | ~12K | ~13.7K | ~14.7K | ~7K+ |
| **License** | Commercial (free tier) | MIT (OpenJS Foundation) | Apache 2.0 (Ion is commercial) | BSD-3-Clause |
| **3D Capability** | Extrusions & terrain | Layered 3D views | Full 3D globe | Extrusions & terrain |
| **Rendering** | WebGL vector tiles | WebGL2 GPU layers | WebGL 3D Tiles | WebGL vector tiles |
| **React Integration** | react-map-gl | Native React support | resium | react-map-gl |
| **Clustering** | Built-in | Custom layers | Built-in | Built-in |
| **Heatmaps** | Built-in layer | HeatmapLayer | Built-in | Built-in layer |
| **Offline/Edge** | Limited | Yes (with tiles) | Limited | Yes (self-hosted tiles) |
| **Cost at Scale** | Pay-as-you-go after 50K tiles/mo | Free | Ion subscription | Free |

### 3.2 Recommended Architecture: Hybrid Mapbox GL + Deck.gl

**Strategy**: Use **Mapbox GL JS** (or MapLibre GL JS for cost savings) as the base map renderer, with **Deck.gl** layered on top for GPU-accelerated data visualization overlays.

```
Layer Stack (bottom to top):
1. Mapbox GL JS / MapLibre GL JS - Base map (vector tiles, terrain, satellite)
2. Deck.gl GeoJsonLayer - Conflict zones, areas of control (polygons)
3. Deck.gl ScatterplotLayer - Event markers with clustering
4. Deck.gl HeatmapLayer - Conflict intensity heatmap
5. Deck.gl ArcLayer - Arms flows, migration routes
6. Deck.gl IconLayer + milsymbol - Military symbology overlay
7. Deck.gl PathLayer - Troop movements, animated paths
8. UI Overlay - Tooltips, popups, legend, controls
```

**Integration via react-map-gl**:
```jsx
import Map from 'react-map-gl';
import DeckGL from '@deck.gl/react';

<DeckGL layers={[conflictZones, eventMarkers, heatmap, armsFlows]}>
  <Map mapStyle="mapbox://styles/mapbox/dark-v11" />
</DeckGL>
```

### 3.3 Military Symbology Implementation

**Primary Library**: `milsymbol` (MIT license)

- Supports MIL-STD-2525C/D/E and STANAG APP-6 B/D/E
- Generates SVG symbols in < 20ms per 1000 symbols
- No images or fonts required; pure code-generated symbols
- Integration with Mapbox GL JS, Leaflet, Cesium, OpenLayers

**Usage Pattern**:
```javascript
import { Symbol } from 'milsymbol';

// Create a hostile infantry unit symbol
const symbol = new Symbol('SHGPUCI----E***', {
  size: 35,
  quantity: 200,
  staffComments: 'Hostile force',
  additionalInformation: 'Wagner Group',
  type: 'Infantry',
  dtg: '271600ZJAN25',
  location: '38.9072N 77.0369W'
});

// Get as image for map marker
const canvas = symbol.asCanvas();
const svg = symbol.asSVG();
```

**Alternative for Full MIL-STD-2525D/E**: `mil-sym-ts` (TypeScript, supports multi-point tactical graphics)

### 3.4 Map Features Checklist

| Feature | Implementation |
|---------|---------------|
| Heatmap overlay | Deck.gl HeatmapLayer with temporal filtering |
| Marker clustering | Supercluster + Deck.gl IconLayer |
| Animated temporal playback | Custom animation loop updating Deck.gl layer data by timestamp |
| Conflict zone polygons | Deck.gl GeoJsonLayer with fill and stroke |
| Areas of control shading | Deck.gl PolygonLayer with semi-transparent fills |
| Military symbology | milsymbol SVG -> Deck.gl IconLayer |
| Layer toggling | React state controlling layer visibility |
| Satellite/terrain/political | Mapbox style switching (dark-v11, satellite-v9, etc.) |
| 3D globe (optional) | CesiumJS with resium wrapper for dedicated globe view |
| Drawing tools | @mapbox/mapbox-gl-draw or nebula.gl for user annotations |

---

## 4. Data Visualization Library Recommendations

### 4.1 Chart Library Comparison

| Criteria | Recharts | Nivo | Apache ECharts | Tremor |
|----------|----------|------|----------------|--------|
| **Best For** | Quick setup, React-idiomatic | Beautiful pre-styled, theming | Large datasets, real-time | Dashboard-specific components |
| **Rendering** | SVG only | SVG, Canvas, SSR | Canvas + WebGL | SVG (via Recharts) |
| **Bundle Size** | ~45KB | ~80KB+ (larger) | ~40KB (core) | ~30KB (components) |
| **Max Data Points** | ~10K | ~10K | 10M+ (streaming) | ~10K |
| **Learning Curve** | Low | Medium | High (config-based API) | Very Low |
| **Animations** | Basic | Rich built-in | Rich + GPU-accelerated | Basic (via Recharts) |
| **Dark Theme** | Manual styling | Built-in theming | Built-in theme support | Tailwind dark mode |
| **SSR Support** | Limited | Yes (native) | Node canvas | Yes (Next.js) |
| **TypeScript** | Yes | Yes | Yes | Yes |
| **License** | MIT | MIT | Apache 2.0 | Apache 2.0 |

### 4.2 Recommended Library Per Visualization Type

| Visualization Type | Primary Library | Why |
|-------------------|----------------|-----|
| **Line/Area charts** (conflict trends) | Apache ECharts | Handles large time-series, real-time streaming, zoom/brush |
| **Bar charts** (comparative stats) | Recharts or Tremor | Simple, declarative, fast to implement |
| **KPI cards / Sparklines** | Tremor | Purpose-built dashboard components with dark mode |
| **Treemaps** (categorical breakdown) | Nivo | Beautiful defaults, responsive, rich interactivity |
| **Sankey diagrams** (arms flows) | Apache ECharts | Best Sankey implementation with tooltips and animation |
| **Network graphs** (actor relationships) | Cytoscape.js | Purpose-built for graph analysis; compound nodes, layouts, WebGL |
| **Timelines** (event progression) | D3.js (custom) or vis-timeline | Maximum control for custom conflict timelines |
| **Heatmap grids** (correlation matrices) | Nivo HeatMap | Clean API, responsive, built-in theming |
| **Donut/Pie charts** | Nivo or Tremor | Visually polished defaults |
| **Geographic visualizations** | Deck.gl + Mapbox GL | GPU-accelerated, handles millions of points |
| **Gauges / Radial progress** | Apache ECharts | Rich gauge components with custom styling |

### 4.3 Network Graph Visualization

**Primary**: Cytoscape.js

| Feature | Cytoscape.js | D3.js (force) | Sigma.js |
|---------|-------------|---------------|----------|
| **Focus** | Graph-specific | General-purpose | Large graph rendering |
| **Built-in Layouts** | 15+ (force, hierarchical, circular, grid, concentric, etc.) | Manual (d3-force only) | ForceAtlas2 |
| **Interactivity** | Pinch-zoom, box-select, pan built-in | Must build manually | Built-in |
| **Graph Algorithms** | BFS, DFS, Dijkstra, PageRank, etc. | None built-in | None built-in |
| **Compound Nodes** | Yes (nested groups) | No | No |
| **Performance** | Canvas/WebGL, handles 10K+ nodes | SVG, struggles at 1K+ nodes | WebGL, handles 100K+ edges |
| **Learning Curve** | Moderate | Steep | Moderate |
| **License** | MIT | BSD | MIT |

**Recommendation**: Use Cytoscape.js for actor relationship networks (compound node support for grouping factions, built-in graph analysis). For very large graphs (100K+ edges), consider Sigma.js v2 with WebGL rendering.

### 4.4 Data Table Comparison

| Feature | AG Grid | TanStack Table |
|---------|---------|----------------|
| **Architecture** | Batteries-included | Headless (logic-only) |
| **Bundle Size** | ~200KB+ (Enterprise) | ~15KB (+ UI code) |
| **Max Rows** | 100K+ (native virtualization) | ~10K (with react-window) |
| **Built-in Features** | Sorting, filtering, grouping, pivoting, Excel export | Sorting, filtering (UI must be built) |
| **Server-side** | Built-in server-side row model | Manual implementation |
| **Customization** | Configurable but opinionated | Fully custom UI |
| **License** | Community (MIT) / Enterprise (commercial) | MIT |
| **Dev Effort** | Low | High |

**Recommendation**: Use **TanStack Table + Shadcn UI table components** for most dashboard tables (event lists, intel feeds). Use **AG Grid Community** for the Investigation view's detailed data table where advanced sorting, filtering, and column pinning are critical. This avoids the Enterprise license while covering 90% of needs.

---

## 5. Dark Theme Design System

### 5.1 Color Palette

The palette is designed for intelligence/defense dashboards, optimized for extended screen time, WCAG 2.1 AA compliance, and color-blind accessibility.

#### Base Colors

| Token | Hex Code | Usage |
|-------|----------|-------|
| `--bg-primary` | `#0A0E14` | Main background (near-black with blue undertone) |
| `--bg-secondary` | `#111822` | Card/panel backgrounds |
| `--bg-tertiary` | `#1A2332` | Elevated surfaces, hover states |
| `--bg-quaternary` | `#243044` | Active states, selected items |
| `--border-subtle` | `#1E2D3D` | Subtle dividers between panels |
| `--border-default` | `#2A3A4E` | Default borders |
| `--border-strong` | `#3D5068` | Emphasized borders, focus rings |

#### Text Colors

| Token | Hex Code | Usage |
|-------|----------|-------|
| `--text-primary` | `#E6EDF3` | Primary text, headings |
| `--text-secondary` | `#8B9BB4` | Secondary text, labels |
| `--text-tertiary` | `#5A6B82` | Disabled text, placeholders |
| `--text-inverse` | `#0A0E14` | Text on light/accent backgrounds |

#### Semantic / Status Colors

| Token | Hex Code | Usage |
|-------|----------|-------|
| `--status-critical` | `#F85149` | Critical alerts, hostile forces, high risk |
| `--status-critical-bg` | `#3D1214` | Critical alert background |
| `--status-warning` | `#D29922` | Warnings, elevated risk, caution |
| `--status-warning-bg` | `#2E2111` | Warning background |
| `--status-success` | `#3FB950` | Stable, positive change, friendly forces |
| `--status-success-bg` | `#122117` | Success background |
| `--status-info` | `#58A6FF` | Informational, neutral forces, links |
| `--status-info-bg` | `#0D1D32` | Info background |
| `--status-unknown` | `#8B8B8B` | Unknown status, pending verification |

#### Accent / Data Visualization Colors

| Token | Hex Code | Usage |
|-------|----------|-------|
| `--accent-primary` | `#3B82F6` | Primary accent (buttons, active states) |
| `--accent-secondary` | `#8B5CF6` | Secondary accent (charts, alt series) |
| `--accent-tertiary` | `#06B6D4` | Tertiary accent (charts, third series) |
| `--chart-1` | `#3B82F6` | Chart series 1 (Blue) |
| `--chart-2` | `#F59E0B` | Chart series 2 (Amber) |
| `--chart-3` | `#10B981` | Chart series 3 (Emerald) |
| `--chart-4` | `#8B5CF6` | Chart series 4 (Violet) |
| `--chart-5` | `#EC4899` | Chart series 5 (Pink) |
| `--chart-6` | `#06B6D4` | Chart series 6 (Cyan) |
| `--chart-7` | `#F97316` | Chart series 7 (Orange) |
| `--chart-8` | `#84CC16` | Chart series 8 (Lime) |

#### Military Symbology Colors (MIL-STD-2525)

| Token | Hex Code | Affiliation |
|-------|----------|-------------|
| `--mil-friendly` | `#80E0FF` | Friendly forces (light cyan/blue) |
| `--mil-hostile` | `#FF8080` | Hostile forces (light red) |
| `--mil-neutral` | `#AAFFAA` | Neutral forces (light green) |
| `--mil-unknown` | `#FFFF80` | Unknown forces (light yellow) |

### 5.2 Typography System

```
Font Stack:
  Primary:    'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif
  Monospace:  'JetBrains Mono', 'Fira Code', 'SF Mono', 'Consolas', monospace
  Map Labels: 'DIN Pro', 'Inter', sans-serif (matches Mapbox default)

Type Scale (1.200 - Minor Third):
  --text-2xs:     0.625rem  (10px)  — Sparkline labels, map annotations
  --text-xs:      0.75rem   (12px)  — Table cells, metadata, timestamps
  --text-sm:      0.875rem  (14px)  — Body text, form labels, sidebar items
  --text-base:    1rem      (16px)  — Default body, card content
  --text-lg:      1.125rem  (18px)  — Card titles, section headings
  --text-xl:      1.25rem   (20px)  — Page subtitles
  --text-2xl:     1.5rem    (24px)  — Page titles
  --text-3xl:     1.875rem  (30px)  — KPI numbers, hero stats
  --text-4xl:     2.25rem   (36px)  — Dashboard overview numbers

Font Weights:
  --font-normal:    400  — Body text
  --font-medium:    500  — Labels, buttons, table headers
  --font-semibold:  600  — Card titles, section headings
  --font-bold:      700  — KPI numbers, page titles

Line Heights:
  --leading-tight:   1.25  — Headings, KPI numbers
  --leading-snug:    1.375 — Card content
  --leading-normal:  1.5   — Body text
  --leading-relaxed: 1.625 — Long-form content

Letter Spacing:
  --tracking-tighter: -0.02em  — Large headings
  --tracking-tight:   -0.01em  — KPI numbers
  --tracking-normal:   0em     — Body text
  --tracking-wide:     0.025em — Labels, all-caps metadata
  --tracking-wider:    0.05em  — Small caps, status badges
```

### 5.3 Spacing & Layout Tokens

```
Spacing Scale (4px base unit):
  --space-0:    0px
  --space-1:    4px    — Tight inline spacing
  --space-2:    8px    — Icon-to-text gap, dense lists
  --space-3:    12px   — Default element padding
  --space-4:    16px   — Card internal padding
  --space-5:    20px   — Section spacing
  --space-6:    24px   — Panel padding
  --space-8:    32px   — Major section gaps
  --space-10:   40px   — Page margins
  --space-12:   48px   — Large section breaks

Border Radius:
  --radius-sm:    4px   — Buttons, inputs, badges
  --radius-md:    6px   — Cards, panels
  --radius-lg:    8px   — Modals, dialogs
  --radius-xl:    12px  — Large containers
  --radius-full:  9999px — Avatars, circular indicators

Shadows (dark theme - uses opacity, not color):
  --shadow-sm:    0 1px 2px rgba(0,0,0,0.3)
  --shadow-md:    0 4px 6px rgba(0,0,0,0.4)
  --shadow-lg:    0 10px 15px rgba(0,0,0,0.5)
  --shadow-xl:    0 20px 25px rgba(0,0,0,0.6)
```

---

## 6. UI Component Library Stack

### 6.1 Recommended Stack

| Layer | Library | Purpose |
|-------|---------|---------|
| **Base Components** | Shadcn/UI + Radix UI | Buttons, inputs, dialogs, dropdowns, popovers, tooltips |
| **Dashboard Components** | Tremor | KPI cards, sparklines, pre-built chart wrappers |
| **Styling** | Tailwind CSS v4 | Utility-first CSS, dark mode via `class` strategy |
| **Chart - General** | Recharts (via Tremor) | Simple bar, line, area charts |
| **Chart - Advanced** | Apache ECharts | Large datasets, Sankey, real-time, streaming |
| **Chart - Decorative** | Nivo | Treemaps, heatmap grids, chord diagrams |
| **Network Graph** | Cytoscape.js | Actor relationship networks |
| **Map** | Mapbox GL JS + Deck.gl | Geospatial with GPU-accelerated overlays |
| **Military Symbols** | milsymbol | MIL-STD-2525 symbol rendering |
| **Data Table** | TanStack Table + AG Grid Community | Headless tables + advanced grid |
| **Grid Layout** | react-grid-layout | Drag-and-drop configurable widget layout |
| **Command Palette** | cmdk | Cmd+K navigation for power users |
| **Date Picker** | react-day-picker (via Shadcn) | Date range selection |
| **Icons** | Lucide React | Consistent icon set |
| **Animations** | Framer Motion | Panel transitions, widget animations |
| **State Management** | Zustand or Jotai | Lightweight reactive state for dashboard config |

### 6.2 Shadcn/UI Components to Use

Core components needed for the dashboard:
- `Button`, `Badge`, `Avatar` - Basic UI
- `Card`, `CardHeader`, `CardContent` - Widget containers
- `Dialog`, `Sheet`, `Drawer` - Modal interactions
- `DropdownMenu`, `ContextMenu` - Right-click menus
- `Command` (cmdk) - Command palette
- `Tabs`, `NavigationMenu` - Navigation
- `Select`, `Combobox`, `DatePicker` - Form inputs
- `Table` - Basic tables (for small datasets)
- `Tooltip`, `Popover`, `HoverCard` - Information overlays
- `Skeleton` - Loading states
- `Toast` / `Sonner` - Notifications
- `ScrollArea` - Virtual scrolling containers
- `Separator`, `Collapsible`, `Accordion` - Layout helpers
- `Toggle`, `Switch`, `Checkbox` - Settings and filters
- `Slider` - Temporal playback control

### 6.3 Tremor Dashboard Components

Tremor provides purpose-built dashboard components:
- `Card` with built-in metric display
- `AreaChart`, `BarChart`, `LineChart` - Chart wrappers over Recharts
- `DonutChart`, `BarList` - Compact visualizations
- `Tracker` - Status timeline indicators
- `SparkAreaChart`, `SparkBarChart`, `SparkLineChart` - Inline sparklines
- `NumberTicker` - Animated KPI numbers
- `Badge`, `CategoryBar` - Status indicators
- Dark/light mode toggle with global theming

---

## 7. React-Grid-Layout Implementation

### 7.1 Configuration

```javascript
// Dashboard grid configuration
const gridConfig = {
  className: 'dashboard-grid',
  cols: { lg: 12, md: 10, sm: 6, xs: 4, xxs: 2 },
  rowHeight: 60,
  margin: [12, 12],        // gap between widgets
  containerPadding: [16, 16],
  isDraggable: true,
  isResizable: true,
  compactType: 'vertical', // or 'horizontal' or null for free-form
  preventCollision: false,
  useCSSTransforms: true,
};

// Widget definitions with min/max constraints
const widgetCatalog = {
  kpiCard:       { minW: 2, minH: 1, maxW: 4,  maxH: 2  },
  lineChart:     { minW: 4, minH: 3, maxW: 12, maxH: 6  },
  barChart:      { minW: 3, minH: 3, maxW: 8,  maxH: 6  },
  mapWidget:     { minW: 6, minH: 4, maxW: 12, maxH: 10 },
  networkGraph:  { minW: 4, minH: 4, maxW: 12, maxH: 10 },
  dataTable:     { minW: 4, minH: 3, maxW: 12, maxH: 8  },
  eventFeed:     { minW: 3, minH: 3, maxW: 6,  maxH: 10 },
  treemap:       { minW: 3, minH: 3, maxW: 8,  maxH: 6  },
  sankeyDiagram: { minW: 6, minH: 4, maxW: 12, maxH: 8  },
  timeline:      { minW: 6, minH: 2, maxW: 12, maxH: 4  },
};
```

### 7.2 Layout Persistence

```javascript
// Save layout on change
const handleLayoutChange = (layout, layouts) => {
  localStorage.setItem('dashboardLayouts', JSON.stringify(layouts));
  // Also persist to server for cross-device sync
  api.saveDashboardLayout(userId, dashboardId, layouts);
};

// Load saved layout
const loadSavedLayout = () => {
  const saved = localStorage.getItem('dashboardLayouts');
  return saved ? JSON.parse(saved) : defaultLayouts;
};
```

### 7.3 Widget Catalog & Dynamic Addition

Widgets can be added from a catalog panel. Each widget registers itself with a type, default dimensions, and the React component to render. The layout system handles positioning, and users can freely rearrange and resize.

---

## 8. Real-Time Update Patterns

### 8.1 Data Flow Architecture

```
WebSocket Server (conflict events)
        |
        v
  WebSocket Client (reconnecting-websocket)
        |
        v
  Event Dispatcher (Zustand store)
        |
        +---> Map Store ----> Map markers update (Deck.gl layer data swap)
        |
        +---> Feed Store ---> Live event feed prepend (virtualized list)
        |
        +---> Chart Store --> Chart data append (ECharts setOption merge)
        |
        +---> Alert Store --> Toast notification + badge count
        |
        +---> Table Store --> Table row insertion (TanStack query invalidation)
```

### 8.2 Update Strategy Per Component

| Component | Update Method | Frequency | Technique |
|-----------|--------------|-----------|-----------|
| **Map markers** | Deck.gl data prop swap | On event | Replace layer data array; Deck.gl diffs internally |
| **Heatmap** | Deck.gl data prop swap | Every 30s batch | Batch weight recalculation, smooth transition |
| **Live event feed** | Prepend to virtualized list | On event | react-virtuoso with `firstItemIndex` shifting |
| **Trend charts** | ECharts `setOption` merge | Every 60s | Append new data point, shift window if needed |
| **KPI numbers** | Animated counter | Every 30s | Tremor NumberTicker with spring animation |
| **Network graph** | Cytoscape `add()`/`remove()` | On event | Incremental graph mutation, re-run layout on batch |
| **Data table** | TanStack Query invalidation | Every 60s | Background refetch, optimistic row insertion |
| **Sparklines** | Direct data prop update | Every 60s | Lightweight re-render, SVG path transition |
| **Alerts/Toasts** | Push to notification store | On event | Sonner toast with severity-based styling and sound |
| **Connection status** | WebSocket readyState | Continuous | Status indicator in header (green/yellow/red dot) |

### 8.3 WebSocket Reconnection Strategy

```javascript
// reconnecting-websocket with exponential backoff
const ws = new ReconnectingWebSocket(WS_URL, [], {
  maxReconnectionDelay: 30000,     // max 30s between retries
  minReconnectionDelay: 1000,      // start at 1s
  reconnectionDelayGrowFactor: 1.5, // exponential backoff
  connectionTimeout: 5000,
  maxRetries: Infinity,            // never stop trying
});

// Connection status indicator
ws.onopen = () => setConnectionStatus('connected');
ws.onclose = () => setConnectionStatus('reconnecting');
ws.onerror = () => setConnectionStatus('error');
```

### 8.4 Optimistic Updates & Batching

- **Batch incoming events**: Accumulate events over 100ms windows before dispatching to avoid excessive re-renders
- **Requestanimationframe throttling**: Use `requestAnimationFrame` to throttle map and chart updates to 60fps
- **Stale-while-revalidate**: Show cached data immediately while fetching fresh data in background (React Query / TanStack Query pattern)
- **Skeleton screens**: Show animated placeholder content during initial load (Shadcn Skeleton component)
- **Progressive loading**: Load critical widgets first (KPI cards, map), then secondary (charts, tables), then tertiary (network graphs)

---

## 9. Accessibility & Performance Guidelines

### 9.1 Accessibility (WCAG 2.1 AA Minimum)

- **Color contrast**: All text meets 4.5:1 contrast ratio against backgrounds; large text meets 3:1
- **Color-blind safe**: Never rely on color alone to convey meaning; always pair with icons, patterns, or text labels
- **Keyboard navigation**: All interactive elements focusable via Tab; Escape to close modals; arrow keys in lists/tables
- **Screen reader**: ARIA labels on all charts, maps, and interactive widgets; live regions for real-time updates
- **Reduced motion**: Respect `prefers-reduced-motion` media query; disable animations for users who request it
- **Focus indicators**: Visible focus rings (`--border-strong` #3D5068 with 2px outline offset)
- **CVD (Color Vision Deficiency) mode**: Offer alternative palettes following Bloomberg's approach (blue/red scheme for deuteranopia; separate scheme for protanomaly)

### 9.2 Performance Targets

| Metric | Target | Strategy |
|--------|--------|----------|
| **First Contentful Paint** | < 1.5s | Code splitting, critical CSS inlining |
| **Largest Contentful Paint** | < 2.5s | Lazy load charts/maps below fold |
| **Time to Interactive** | < 3.5s | Defer non-critical JS, Web Workers for data processing |
| **Map frame rate** | 60fps | Deck.gl GPU rendering, avoid DOM manipulation |
| **Chart render (10K points)** | < 200ms | ECharts Canvas mode, data downsampling |
| **Table render (10K rows)** | < 100ms | Virtual scrolling (TanStack Virtual / AG Grid virtualization) |
| **WebSocket latency** | < 500ms e2e | Binary protocol (MessagePack), server-side filtering |
| **Bundle size (initial)** | < 300KB gzip | Tree shaking, dynamic imports, route-based splitting |
| **Memory (sustained)** | < 200MB | Object pooling for map markers, garbage collection awareness |

### 9.3 Performance Optimization Techniques

1. **Code splitting**: Each major view (Map, Analytics, Investigation) is a separate lazy-loaded route
2. **Virtual scrolling**: All lists and tables with > 50 items use virtualization
3. **Web Workers**: Data processing (filtering, aggregation, statistical calculations) offloaded to Web Workers
4. **Canvas over SVG**: For charts with > 1000 data points, prefer Canvas rendering (ECharts) over SVG (Recharts)
5. **Debounced filters**: Search and filter inputs debounced at 300ms
6. **Memoization**: `React.memo` + `useMemo` for expensive chart computations; Zustand selectors for minimal re-renders
7. **Image optimization**: Map tiles cached aggressively; sprite sheets for custom markers
8. **Data windowing**: For time-series, only load visible time window + buffer; fetch more on scroll/zoom

---

## 10. Navigation & Interaction Patterns

### 10.1 Command Palette (Cmd+K)

Powered by `cmdk` library, providing:
- Quick navigation to any view/dashboard
- Entity search across all data
- Recent items and bookmarks
- Filter commands (e.g., "show conflicts in Africa")
- Settings and keyboard shortcut reference

### 10.2 Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Cmd+K` / `Ctrl+K` | Open command palette |
| `Cmd+/` | Toggle left sidebar |
| `Cmd+\` | Toggle right sidebar |
| `Cmd+F` | Focus search/filter |
| `Cmd+1-9` | Switch dashboard tabs |
| `Escape` | Close modals, deselect, exit full-screen |
| `F` | Toggle fullscreen on focused widget |
| `Space` | Play/pause temporal playback |
| `←` / `→` | Step backward/forward in timeline |
| `+` / `-` | Zoom in/out on map |
| `R` | Reset map view to default extent |
| `N` | Focus next alert |
| `?` | Show keyboard shortcut reference |

### 10.3 Filter & Search UX

- **Global filter bar**: Persistent across all views; filters apply to map, charts, and tables simultaneously
- **Faceted search**: Region, conflict type, date range, severity, actors, data source
- **Saved filters**: Users can save and name filter combinations
- **Filter chips**: Active filters shown as removable chips below filter bar
- **Date range picker**: Preset ranges (24h, 7d, 30d, 90d, 1y, All) + custom date picker
- **Temporal slider**: Dedicated slider on map view for animated time-based playback

---

## 11. Reference Implementation Notes

### 11.1 Folder Structure (Proposed)

```
src/
  components/
    ui/              # Shadcn/UI base components
    dashboard/       # Dashboard-specific components
      widgets/       # Individual widget components
        KPICard.tsx
        ConflictMap.tsx
        TrendChart.tsx
        NetworkGraph.tsx
        EventFeed.tsx
        DataTable.tsx
        SankeyDiagram.tsx
        Treemap.tsx
        Timeline.tsx
      WidgetCatalog.tsx
      DashboardGrid.tsx
    map/             # Map-specific components
      layers/        # Deck.gl layer definitions
      controls/      # Map controls (zoom, layer toggle, draw)
      symbols/       # milsymbol integration
    charts/          # Chart wrapper components
    navigation/      # Sidebar, header, command palette
  hooks/
    useWebSocket.ts
    useDashboardLayout.ts
    useMapLayers.ts
    useRealTimeData.ts
  stores/
    dashboardStore.ts
    mapStore.ts
    eventStore.ts
    filterStore.ts
  styles/
    tokens.css       # Design tokens (CSS custom properties)
    theme.ts         # Tailwind theme configuration
  lib/
    milsymbol.ts     # Military symbology utilities
    websocket.ts     # WebSocket client
    dataProcessing.ts # Web Worker scripts
```

### 11.2 Key Dependencies (package.json excerpt)

```json
{
  "dependencies": {
    "react": "^18.3.0",
    "next": "^14.2.0",
    "@radix-ui/react-*": "latest",
    "tailwindcss": "^4.0.0",
    "class-variance-authority": "^0.7.0",
    "clsx": "^2.1.0",
    "tailwind-merge": "^2.2.0",

    "mapbox-gl": "^3.4.0",
    "react-map-gl": "^7.1.0",
    "@deck.gl/react": "^9.0.0",
    "@deck.gl/layers": "^9.0.0",
    "@deck.gl/geo-layers": "^9.0.0",
    "milsymbol": "^2.2.0",
    "supercluster": "^8.0.0",

    "echarts": "^5.5.0",
    "echarts-for-react": "^3.0.0",
    "recharts": "^2.12.0",
    "@tremor/react": "^3.18.0",
    "@nivo/treemap": "^0.87.0",
    "@nivo/heatmap": "^0.87.0",
    "@nivo/sankey": "^0.87.0",

    "cytoscape": "^3.30.0",
    "react-cytoscapejs": "^2.0.0",

    "react-grid-layout": "^1.4.0",
    "@tanstack/react-table": "^8.17.0",
    "@tanstack/react-virtual": "^3.5.0",
    "ag-grid-react": "^32.0.0",
    "ag-grid-community": "^32.0.0",

    "cmdk": "^1.0.0",
    "framer-motion": "^11.0.0",
    "lucide-react": "^0.400.0",
    "sonner": "^1.5.0",
    "zustand": "^4.5.0",
    "@tanstack/react-query": "^5.45.0",
    "reconnecting-websocket": "^4.4.0",
    "date-fns": "^3.6.0"
  }
}
```

---

## 12. Summary of Recommendations

### Critical Path (Build First)
1. **Design system**: Set up Tailwind config with dark theme tokens, Shadcn/UI components, and Inter font
2. **Dashboard shell**: Header, sidebars, command palette, navigation — the "chrome" around content
3. **Map view**: Mapbox GL JS + Deck.gl with basic marker layer and heatmap
4. **Live event feed**: WebSocket connection + virtualized event list
5. **KPI cards**: Tremor cards with key metrics

### Second Phase
6. **Configurable grid**: react-grid-layout with widget catalog
7. **Trend charts**: ECharts for time-series, Recharts/Tremor for simple charts
8. **Data tables**: TanStack Table for event lists, AG Grid for investigation view
9. **Temporal playback**: Animated slider controlling map and chart data

### Third Phase
10. **Network graph**: Cytoscape.js for actor relationship visualization
11. **Military symbology**: milsymbol integration with map markers
12. **Advanced charts**: Sankey (ECharts), Treemap (Nivo), Timeline (D3.js custom)
13. **Saved dashboards**: Layout persistence, user preferences, shared views

### Final Polish
14. **Accessibility audit**: WCAG 2.1 AA compliance, CVD color schemes
15. **Performance optimization**: Code splitting, Web Workers, virtual scrolling
16. **Keyboard shortcuts**: Full shortcut system with reference panel
17. **Notification system**: Toast notifications with sound, severity levels, badge counts

---

## Sources

- [Palantir Gotham Platform](https://www.palantir.com/platforms/gotham/)
- [Jenny Fan - Palantir Gotham Design](https://jennyfan.com/projects/gotham)
- [Inside Palantir Gotham - ProDefence](https://prodefence.io/news/palantir-gotham-reviews-features)
- [Mapping Libraries: A Practical Comparison - GISCarta](https://giscarta.com/blog/mapping-libraries-a-practical-comparison)
- [Cesium vs Deck.gl - MATOM.AI](https://matom.ai/insights/cesium-vs-deck-gl/)
- [npm trends: cesium vs deck.gl vs mapbox-gl vs maplibre-gl](https://npmtrends.com/cesium-vs-deck.gl-vs-leaflet-vs-mapbox-gl-vs-maplibre-gl-vs-openlayers)
- [milsymbol - Military Symbols in JavaScript](https://github.com/spatialillusions/milsymbol)
- [mil-sym-ts - MIL-STD-2525 TypeScript Renderer](https://github.com/missioncommand/mil-sym-ts)
- [Mission Command Open Source](https://missioncommand.github.io/)
- [Network Graph Visualization Libraries Comparison](https://www.cylynx.io/blog/a-comparison-of-javascript-graph-network-visualisation-libraries/)
- [Cytoscape.js Documentation](https://js.cytoscape.org/)
- [Best React Chart Libraries 2025 - LogRocket](https://blog.logrocket.com/best-react-chart-libraries-2025/)
- [Nivo vs Recharts - Speakeasy](https://www.speakeasy.com/blog/nivo-vs-recharts)
- [8 Best React Chart Libraries 2025 - Embeddable](https://embeddable.com/blog/react-chart-libraries)
- [react-grid-layout GitHub](https://github.com/react-grid-layout/react-grid-layout)
- [Building Interactive Dashboards with React-Grid-Layout - ilert](https://www.ilert.com/blog/building-interactive-dashboards-why-react-grid-layout-was-our-best-choice)
- [Shadcn/UI](https://www.shadcn.io)
- [Tremor - Dashboard Components](https://www.tremor.so/)
- [TanStack Table vs AG Grid Comparison 2025](https://www.simple-table.com/blog/tanstack-table-vs-ag-grid-comparison)
- [AG Grid Alternatives 2026 - TFC](https://www.thefrontendcompany.com/posts/ag-grid-alternatives)
- [Bloomberg Terminal Color Accessibility](https://www.bloomberg.com/company/stories/designing-the-terminal-for-color-accessibility/)
- [How Bloomberg Terminal UX Designers Conceal Complexity](https://www.bloomberg.com/company/stories/how-bloomberg-terminal-ux-designers-conceal-complexity/)
- [The Impossible Bloomberg Makeover - UX Magazine](https://uxmag.com/articles/the-impossible-bloomberg-makeover)
- [Dark Dashboard Color Palettes - ColorsWall](https://colorswall.com/palette/1459)
- [Ultimate Dashboard Colour Palette - DEV Community](https://dev.to/info_generalhazedawn_a3d/design-matters-1-the-ultimate-dashboard-colour-palette-ma3)
