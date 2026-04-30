---
trigger: glob
globs: admin/**
---

# Admin Panel Conventions (Next.js 16)

## Tech Stack

- **Framework:** Next.js 16 (App Router)
- **React:** 19.2
- **State:** Redux Toolkit + react-redux
- **Styling:** TailwindCSS v4
- **Icons:** Lucide React
- **Rich Text:** TipTap editor (for CMS pages)
- **Drag & Drop:** @dnd-kit/core + @dnd-kit/sortable
- **Monitoring:** Sentry (`@sentry/nextjs`)

## Directory Layout

```text
admin/src/
├── app/                         → Next.js App Router pages
│   ├── gallery/                 → Image & video gallery management
│   ├── talents/                 → Talent management CRUD
│   ├── talent-applications/     → Talent application review
│   ├── talent-categories/       → Category management
│   ├── talent-category-usages/  → Category usage rules
│   ├── talent-featured-images/  → Featured image management
│   ├── talent-product-use-case/ → Product use case management
│   ├── pro-refinements/         → Pro refinement management
│   ├── pricing/                 → Point cost management
│   ├── settings/                → App settings
│   ├── static-pages/            → CMS pages (ToS, Privacy)
│   ├── tips/                    → Tips management
│   ├── users/                   → User management
│   ├── vouchers/                → Voucher management
│   ├── clothes-gallery/         → Clothes gallery management
│   └── login/                   → Admin login
├── components/                  → Shared UI components (AuthGuard, ReduxProvider, etc.)
└── services/                    → API service modules (one per domain entity)
```

## Patterns

### API Services

- Each domain has a service file in `admin/src/services/` (e.g., `talents.ts`, `vouchers.ts`).
- Services define TypeScript interfaces for API entities and export fetch/create/update/delete functions.
- API base URL from `NEXT_PUBLIC_API_URL` env var.
- Admin routes hit `/api/admin/*` endpoints on the backend.

### Auth

- Uses `AuthGuard` component wrapping all routes in `layout.tsx`.
- Admin users must have `is_admin: true` or `is_superuser: true` on the backend.

### Layout

- Server-side layout (`layout.tsx` is NOT `"use client"`), uses `Metadata` export.
- Font: Geist + Geist Mono (from `next/font/google`).

### Differences from Frontend

- Admin uses **Next.js 16** (frontend uses 15).
- Admin uses **Lucide icons** (frontend uses Phosphor + MUI icons).
- Admin has **no canvas/Konva** editing.
- Admin has **no redux-persist** (no client-side persistence).
- Admin pages are simpler CRUD interfaces focused on data management.
