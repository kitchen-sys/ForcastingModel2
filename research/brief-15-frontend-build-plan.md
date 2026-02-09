# Build Plan Brief 15: Frontend Dashboard — Agent Blueprint

## Objective
Create a COMPLETE build plan that a coding agent can follow to build the Next.js frontend dashboard from scratch. Every page, every component, every style — spelled out.

## Context — Read These First
- /home/user/ForcastingModel2/research/findings-04-ui-visualization.md
- /home/user/ForcastingModel2/research/findings-10-mvp-scope.md
- /home/user/ForcastingModel2/research/findings-12-implementation-roadmap.md

## What This Document Must Contain

### 1. Directory Structure
- Exact file tree for `/src/frontend/`
- Every page, component, hook, utility, and style file
- Next.js App Router structure (app/ directory layout)

### 2. Dependencies
- Complete `package.json` with pinned versions
- All libraries: Next.js, React, Mapbox GL JS, Recharts, Zustand, TanStack Query, Tailwind, Shadcn/ui, etc.
- Why each dependency is needed

### 3. Design System Specification
- Color palette: exact hex codes for dark theme (background, surface, text, borders, accent, alert colors)
- Typography: font family, sizes, weights for headings, body, labels, data
- Spacing system: padding/margin scale
- Border radius, shadows
- Component tokens (button colors, input styles, card styles)

### 4. Page Specifications
For EACH page/view:
- Route path
- Layout description (what components, where positioned)
- Data requirements (what API calls, what state)
- User interactions (clicks, filters, searches)
- Loading/error states

MVP Pages:
- Dashboard (main view: map + event feed + stats)
- Event Detail view
- Sources Management view
- Settings (optional for MVP)

### 5. Component Specifications
For EACH component:
- Props interface (TypeScript types)
- What it renders
- What data it needs
- Events it emits/handles
- Responsive behavior

Key components:
- ConflictMap (Mapbox GL JS)
- EventFeed (real-time scrolling list)
- EventCard (single event display)
- StatsPanel (key metrics)
- SourceTable (source management)
- FilterBar (date range, category, region)
- AlertBanner (breaking events)
- Timeline (temporal view)

### 6. State Management
- Zustand store structure (slices, actions)
- React Query setup (queries, mutations, cache config)
- WebSocket integration for real-time updates
- How state flows between components

### 7. Map Implementation
- Mapbox GL JS setup and configuration
- Map layers: conflict event markers, heatmap, clusters
- Interaction handlers: click, hover, zoom
- How map syncs with event feed and filters

### 8. Real-Time Updates
- WebSocket connection setup
- How new events appear on map and feed without page refresh
- Notification/toast for breaking events
- Connection status indicator

### 9. API Integration
- API client setup (fetch wrapper or axios)
- Every API call mapped to the backend endpoints
- Error handling patterns
- Loading state management

### 10. Agent Instructions
- Step-by-step build order for a coding agent
- "Create the Next.js scaffold first, then the design system, then layout, then components"
- Testing checkpoints: "After building the map, verify it renders with mock data"
- Mock data files for development before backend is ready

## Deliverable
Write to: /home/user/ForcastingModel2/research/findings-15-frontend-build-plan.md
