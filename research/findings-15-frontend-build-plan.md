# Build Plan 15: Frontend Dashboard — Coding Agent Instructions

## Executive Summary

Step-by-step build plan for Sentinel's Next.js dark-theme intelligence dashboard. The MVP has 3 screens: main dashboard (map + feed + timeline), source management, and event detail. Based on findings-04 (UI/visualization), findings-10 (MVP scope), findings-12 (implementation roadmap).

## Research Scope

Based on: findings-04-ui-visualization.md, findings-10-mvp-scope.md, findings-12-implementation-roadmap.md

---

## 1. Directory Structure

```
src/frontend/
├── package.json
├── tsconfig.json
├── next.config.ts
├── tailwind.config.ts
├── postcss.config.mjs
├── Dockerfile
├── .env.example
├── public/
│   └── favicon.ico
└── src/
    ├── app/
    │   ├── layout.tsx              # Root layout: dark theme, fonts, providers
    │   ├── page.tsx                # Redirect to /dashboard
    │   ├── dashboard/
    │   │   ├── layout.tsx          # Dashboard shell: top bar, nav
    │   │   ├── page.tsx            # Main dashboard: map + feed + timeline
    │   │   ├── sources/
    │   │   │   └── page.tsx        # Source management
    │   │   └── events/
    │   │       └── [id]/
    │   │           └── page.tsx    # Event detail
    │   └── globals.css             # Tailwind base + custom tokens
    ├── components/
    │   ├── ui/                     # Base design system (shadcn-style)
    │   │   ├── button.tsx
    │   │   ├── input.tsx
    │   │   ├── select.tsx
    │   │   ├── badge.tsx
    │   │   ├── card.tsx
    │   │   ├── table.tsx
    │   │   ├── dropdown-menu.tsx
    │   │   └── skeleton.tsx
    │   ├── layout/
    │   │   ├── top-bar.tsx         # App header: logo, search, export
    │   │   ├── filter-bar.tsx      # Date range, event type, country filters
    │   │   └── sidebar.tsx         # Optional nav sidebar
    │   ├── map/
    │   │   ├── conflict-map.tsx    # Mapbox GL JS wrapper
    │   │   ├── map-markers.tsx     # Event markers with color coding
    │   │   └── map-controls.tsx    # Zoom, layer toggle, fullscreen
    │   ├── events/
    │   │   ├── event-feed.tsx      # Scrollable event table
    │   │   ├── event-card.tsx      # Single event row/card
    │   │   ├── event-detail.tsx    # Full event detail view
    │   │   └── event-filters.tsx   # Filter controls
    │   ├── charts/
    │   │   ├── timeline-chart.tsx  # Recharts bar chart
    │   │   └── stats-panel.tsx     # Key metric cards
    │   └── sources/
    │       ├── source-table.tsx    # Source management table
    │       └── add-source-form.tsx # Add new RSS feed form
    ├── lib/
    │   ├── api.ts                  # Fetch wrapper for backend REST API
    │   ├── ws.ts                   # WebSocket client for real-time events
    │   ├── types.ts                # Shared TypeScript interfaces
    │   ├── utils.ts                # Formatting helpers
    │   └── constants.ts            # Event type colors, map config
    └── stores/
        ├── event-store.ts          # Zustand: events, selected event
        └── filter-store.ts         # Zustand: active filters
```

---

## 2. Dependencies (package.json)

```json
{
    "name": "sentinel-frontend",
    "version": "0.1.0",
    "private": true,
    "scripts": {
        "dev": "next dev --turbopack",
        "build": "next build",
        "start": "next start",
        "lint": "next lint",
        "type-check": "tsc --noEmit"
    },
    "dependencies": {
        "next": "^15.1.0",
        "react": "^19.0.0",
        "react-dom": "^19.0.0",
        "zustand": "^5.0.0",
        "@tanstack/react-query": "^5.62.0",
        "mapbox-gl": "^3.9.0",
        "recharts": "^2.15.0",
        "date-fns": "^4.1.0",
        "clsx": "^2.1.0",
        "tailwind-merge": "^2.6.0",
        "@radix-ui/react-dropdown-menu": "^2.1.0",
        "@radix-ui/react-select": "^2.1.0",
        "@radix-ui/react-dialog": "^1.1.0",
        "lucide-react": "^0.460.0"
    },
    "devDependencies": {
        "typescript": "^5.7.0",
        "@types/react": "^19.0.0",
        "@types/react-dom": "^19.0.0",
        "tailwindcss": "^4.0.0",
        "@tailwindcss/postcss": "^4.0.0",
        "postcss": "^8.5.0",
        "eslint": "^9.17.0",
        "eslint-config-next": "^15.1.0",
        "@types/mapbox-gl": "^3.4.0"
    }
}
```

---

## 3. Design System — Dark Intelligence Theme

### 3.1 Color Palette

```css
/* globals.css — CSS custom properties */
:root {
    /* Background layers (darkest to lightest) */
    --bg-primary: #0a0e17;        /* Main background — near black with blue tint */
    --bg-secondary: #111827;      /* Card/panel backgrounds */
    --bg-tertiary: #1a2332;       /* Elevated surfaces, hover states */
    --bg-hover: #1f2b3d;          /* Interactive hover */

    /* Text hierarchy */
    --text-primary: #e2e8f0;      /* Primary text — light gray */
    --text-secondary: #94a3b8;    /* Secondary text — muted */
    --text-tertiary: #64748b;     /* Disabled/hint text */

    /* Borders */
    --border-primary: #1e293b;    /* Subtle dividers */
    --border-hover: #334155;      /* Hover borders */

    /* Event type colors (conflict-specific) */
    --color-battles: #ef4444;          /* Red — armed clashes */
    --color-explosions: #f59e0b;       /* Amber — explosions/remote violence */
    --color-violence-civilians: #f97316; /* Orange — violence against civilians */
    --color-protests: #3b82f6;         /* Blue — protests */
    --color-riots: #8b5cf6;            /* Purple — riots */
    --color-strategic: #06b6d4;        /* Cyan — strategic developments */

    /* Signal colors */
    --color-success: #22c55e;     /* Green — active, healthy */
    --color-warning: #f59e0b;     /* Amber — degraded */
    --color-error: #ef4444;       /* Red — error, critical */
    --color-info: #3b82f6;        /* Blue — informational */

    /* Accent */
    --color-accent: #6366f1;      /* Indigo — primary actions, links */
}
```

### 3.2 Typography

```css
/* Font: Inter for UI, JetBrains Mono for data */
--font-sans: 'Inter', -apple-system, system-ui, sans-serif;
--font-mono: 'JetBrains Mono', 'Fira Code', monospace;

/* Scale */
--text-xs: 0.75rem;    /* 12px — labels, badges */
--text-sm: 0.875rem;   /* 14px — table data, secondary */
--text-base: 1rem;     /* 16px — body text */
--text-lg: 1.125rem;   /* 18px — section headers */
--text-xl: 1.25rem;    /* 20px — page titles */
--text-2xl: 1.5rem;    /* 24px — dashboard title */
```

### 3.3 Event Type Color Map (TypeScript)

```typescript
// lib/constants.ts
export const EVENT_TYPE_COLORS: Record<string, string> = {
    "Battles": "#ef4444",
    "Explosions/Remote violence": "#f59e0b",
    "Violence against civilians": "#f97316",
    "Protests": "#3b82f6",
    "Riots": "#8b5cf6",
    "Strategic developments": "#06b6d4",
};

export const EVENT_TYPE_LABELS: Record<string, string> = {
    "Battles": "Battles",
    "Explosions/Remote violence": "Explosions",
    "Violence against civilians": "Violence vs Civilians",
    "Protests": "Protests",
    "Riots": "Riots",
    "Strategic developments": "Strategic",
};
```

---

## 4. TypeScript Interfaces

```typescript
// lib/types.ts

export interface Event {
    id: string;
    title: string;
    description: string | null;
    event_type: string | null;
    event_date: string;  // ISO date
    latitude: number | null;
    longitude: number | null;
    country: string | null;
    region: string | null;
    actors: string[];
    fatalities: number;
    source_name: string | null;
    source_url: string | null;
    source_type: string | null;
    confidence_score: number;
    ingested_at: string;  // ISO datetime
}

export interface EventCluster {
    cluster_id?: number;
    latitude: number;
    longitude: number;
    count: number;
    dominant_type?: string;
    expansion_zoom?: number;
    event_id?: string;
    event_type?: string;
    title?: string;
}

export interface TimelineBucket {
    date: string;
    count: number;
    fatalities: number;
}

export interface Source {
    id: string;
    name: string;
    url: string;
    source_type: string;
    enabled: boolean;
    poll_interval_minutes: number;
    last_poll_at: string | null;
    last_poll_status: string | null;
    total_items_ingested: number;
    error_count: number;
    consecutive_errors: number;
    reliability_score: number;
    created_at: string;
}

export interface PaginatedResponse<T> {
    data: T[];
    meta: {
        total: number;
        limit: number;
        offset: number;
    };
}

export interface EventFilters {
    date_from?: string;
    date_to?: string;
    event_type?: string[];
    country?: string[];
    search?: string;
    bbox?: string;
}

export interface DashboardStats {
    total_events: number;
    events_today: number;
    active_sources: number;
    countries_covered: number;
}
```

---

## 5. API Client

```typescript
// lib/api.ts
const API_BASE = process.env.NEXT_PUBLIC_API_URL || "http://localhost:8000/api/v1";

async function fetchAPI<T>(path: string, options?: RequestInit): Promise<T> {
    const res = await fetch(`${API_BASE}${path}`, {
        headers: { "Content-Type": "application/json" },
        ...options,
    });
    if (!res.ok) throw new Error(`API error: ${res.status}`);
    return res.json();
}

export const api = {
    events: {
        list: (filters: EventFilters & { limit?: number; offset?: number }) =>
            fetchAPI<PaginatedResponse<Event>>(`/events?${buildParams(filters)}`),
        get: (id: string) => fetchAPI<{ data: Event }>(`/events/${id}`),
        clusters: (params: { bbox: string; zoom: number } & EventFilters) =>
            fetchAPI<{ data: EventCluster[] }>(`/events/clusters?${buildParams(params)}`),
        timeline: (params: EventFilters & { interval?: string }) =>
            fetchAPI<{ data: TimelineBucket[] }>(`/events/timeline?${buildParams(params)}`),
    },
    sources: {
        list: () => fetchAPI<{ data: Source[] }>("/sources"),
        create: (source: Partial<Source>) =>
            fetchAPI<{ data: Source }>("/sources", { method: "POST", body: JSON.stringify(source) }),
        update: (id: string, data: Partial<Source>) =>
            fetchAPI<{ data: Source }>(`/sources/${id}`, { method: "PATCH", body: JSON.stringify(data) }),
        delete: (id: string) => fetchAPI(`/sources/${id}`, { method: "DELETE" }),
        poll: (id: string) => fetchAPI(`/sources/${id}/poll`, { method: "POST" }),
    },
    health: () => fetchAPI<{ status: string }>("/health"),
    stats: () => fetchAPI<DashboardStats>("/stats"),
};
```

---

## 6. Zustand Stores

```typescript
// stores/filter-store.ts
import { create } from "zustand";
import type { EventFilters } from "@/lib/types";

interface FilterState {
    filters: EventFilters;
    setFilter: (key: keyof EventFilters, value: unknown) => void;
    clearFilters: () => void;
    setDateRange: (from: string, to: string) => void;
}

export const useFilterStore = create<FilterState>((set) => ({
    filters: {},
    setFilter: (key, value) => set((s) => ({ filters: { ...s.filters, [key]: value } })),
    clearFilters: () => set({ filters: {} }),
    setDateRange: (from, to) => set((s) => ({
        filters: { ...s.filters, date_from: from, date_to: to }
    })),
}));
```

```typescript
// stores/event-store.ts
import { create } from "zustand";
import type { Event } from "@/lib/types";

interface EventState {
    selectedEvent: Event | null;
    selectEvent: (event: Event | null) => void;
    hoveredEventId: string | null;
    setHoveredEvent: (id: string | null) => void;
}

export const useEventStore = create<EventState>((set) => ({
    selectedEvent: null,
    selectEvent: (event) => set({ selectedEvent: event }),
    hoveredEventId: null,
    setHoveredEvent: (id) => set({ hoveredEventId: id }),
}));
```

---

## 7. Map Implementation

```typescript
// components/map/conflict-map.tsx
// Key implementation details:

// 1. Initialize Mapbox with dark style
const map = new mapboxgl.Map({
    container: mapRef.current,
    style: "mapbox://styles/mapbox/dark-v11",
    center: [30, 20],  // Centered roughly on conflict hotspots
    zoom: 2.5,
    projection: "mercator",
});

// 2. Add GeoJSON source for events
map.addSource("events", {
    type: "geojson",
    data: { type: "FeatureCollection", features: [] },
    cluster: true,
    clusterMaxZoom: 14,
    clusterRadius: 50,
});

// 3. Cluster layer (circles sized by point_count)
map.addLayer({
    id: "clusters",
    type: "circle",
    source: "events",
    filter: ["has", "point_count"],
    paint: {
        "circle-color": ["step", ["get", "point_count"],
            "#6366f1", 10, "#f59e0b", 50, "#ef4444"],
        "circle-radius": ["step", ["get", "point_count"],
            15, 10, 25, 50, 35],
    },
});

// 4. Individual event markers (colored by event_type)
map.addLayer({
    id: "unclustered-point",
    type: "circle",
    source: "events",
    filter: ["!", ["has", "point_count"]],
    paint: {
        "circle-color": ["match", ["get", "event_type"],
            "Battles", "#ef4444",
            "Explosions/Remote violence", "#f59e0b",
            "Violence against civilians", "#f97316",
            "Protests", "#3b82f6",
            "Riots", "#8b5cf6",
            "Strategic developments", "#06b6d4",
            "#94a3b8"],  // default gray
        "circle-radius": 6,
        "circle-stroke-width": 1,
        "circle-stroke-color": "#0a0e17",
    },
});

// 5. Click handler → select event
map.on("click", "unclustered-point", (e) => {
    const feature = e.features[0];
    useEventStore.getState().selectEvent(feature.properties);
});
```

---

## 8. Page Layouts

### Main Dashboard (`/dashboard`)
- **Top bar**: App name (left), search input (center), export button (right)
- **Filter bar**: Sticky row — date range picker, event type multi-select, country dropdown, clear button
- **Content**: CSS Grid — map 60% left, event feed 40% right
- **Timeline**: Full-width bar chart at bottom, 200px height

### Source Management (`/dashboard/sources`)
- **Table**: AG-Grid-style table with columns: Name, Type, Status, Last Poll, Items, Errors, Actions
- **Add Feed**: Modal form with URL input, name, poll interval
- **Ingestion Log**: Scrollable log below table

### Event Detail (`/dashboard/events/[id]`)
- **Mini-map**: 300x250px Mapbox centered on event
- **Metadata**: All fields displayed in card layout
- **Raw data**: Collapsible JSON viewer

---

## 9. Agent Build Instructions — Step by Step

### Step 1: Scaffold Next.js project
```bash
npx create-next-app@latest sentinel-frontend --typescript --tailwind --app --src-dir
```

### Step 2: Install all dependencies
Install Mapbox, Recharts, Zustand, TanStack Query, Radix UI, Lucide icons.

### Step 3: Set up globals.css
Add CSS custom properties (color palette, typography).

### Step 4: Create root layout
Dark background, Inter font, React Query provider, Zustand.

### Step 5: Build lib/ layer
- `types.ts` — All TypeScript interfaces
- `constants.ts` — Event colors, map config
- `api.ts` — API client
- `utils.ts` — Formatting helpers

### Step 6: Build stores
- `filter-store.ts`
- `event-store.ts`

### Step 7: Build UI base components
Button, Input, Select, Badge, Card, Table, Skeleton.

### Step 8: Build layout components
TopBar, FilterBar.

### Step 9: Build map component
ConflictMap with Mapbox GL JS, dark style, clusters, color-coded markers.

### Step 10: Build event feed
EventFeed table, EventCard rows, click-to-select.

### Step 11: Build timeline chart
Recharts BarChart with event counts by day.

### Step 12: Wire up main dashboard page
Combine map + feed + timeline with filter integration.

### Step 13: Build source management page
Source table + add feed form.

### Step 14: Build event detail page
Dynamic route with mini-map + full metadata.

### Verification Checkpoint
```bash
npm run dev
# Visit http://localhost:3000/dashboard
# Verify: dark theme, map renders, layout matches spec
# Test with mock data before backend is ready
```

---

## Comparison Tables

### Map Library

| Criteria | Mapbox GL JS | Deck.gl | Leaflet | CesiumJS |
|----------|-------------|---------|---------|----------|
| Dark style | Built-in | Manual | Plugin | Built-in |
| Clustering | Built-in | Manual | Plugin | Manual |
| 3D support | Terrain | Full 3D | None | Full 3D |
| Bundle size | 210KB | 350KB+ | 40KB | 800KB+ |
| Free tier | 50K loads/mo | Open source | Free | Free |
| **Recommendation** | **YES** | V2 option | No | No |

### Chart Library

| Criteria | Recharts | Nivo | ECharts | Victory |
|----------|----------|------|---------|---------|
| React native | Yes | Yes | Wrapper | Yes |
| Bundle size | 45KB | 80KB+ | 400KB+ | 60KB |
| Customization | Good | Excellent | Excellent | Good |
| Learning curve | Low | Medium | High | Low |
| **Recommendation** | **YES** | V2 option | No | No |

---

## Priority Implementation Order

1. **Scaffold + design system** — Foundation
2. **Types + API client + stores** — Data layer
3. **Map component** — Primary analyst view
4. **Event feed** — Secondary analyst view
5. **Filter bar** — Connects map + feed
6. **Timeline chart** — Trend awareness
7. **Main dashboard page** — Wire it all up
8. **Source management** — Admin view
9. **Event detail** — Drill-down view

---

## Open Questions

1. **Mapbox API key**: Need NEXT_PUBLIC_MAPBOX_TOKEN — free tier is 50K loads/month
2. **Server components vs client**: Map and interactive components must be client — how much can be server?
3. **Mock data**: Need sample GeoJSON for development before backend is ready
4. **Mobile responsive**: Is mobile a requirement for MVP or desktop-only?
