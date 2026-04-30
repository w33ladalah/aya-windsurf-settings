---
trigger: glob
globs: frontend/**
---

# Frontend Conventions (Next.js + React)

## Tech Stack

- **Framework:** Next.js 15 (App Router, typed routes enabled)
- **React:** 19.2
- **State:** Redux Toolkit + redux-persist (persisted: `generator`, `auth`, `ui`)
- **Styling:** TailwindCSS v4 + custom CSS (`globals.css`, `styles.css`, `responsive.css`)
- **UI Components:** MUI 7 (Material UI) + Phosphor Icons + react-icons
- **Canvas/Editor:** Konva + react-konva (with custom webpack aliases in `next.config.ts`)
- **Animation:** Framer Motion + GSAP
- **HTTP:** Axios with interceptors (`src/utils/api.ts`)
- **Auth:** JWT Bearer tokens in `localStorage`, Google OAuth
- **Monitoring:** Sentry (`@sentry/nextjs`)
- **Font:** Nunito (variable weight, preloaded)

## Directory Layout

```text
frontend/src/
├── app/                 → Next.js App Router pages (route-per-folder)
│   ├── generate/        → Image generation page
│   ├── edit/            → Image editing page
│   ├── enhance/         → Image enhancement page
│   ├── compose/         → Image composition page
│   ├── gallery/         → Gallery pages
│   ├── video/           → Video generation page
│   ├── talking-video/   → Talking video page
│   ├── ai-builder/      → AI Builder page
│   ├── quick-tools/     → Quick tools (bg remove, upscale, etc.)
│   └── user/            → User profile page
├── components/          → Shared React components
├── constants/           → API endpoints, static config
├── helpers/             → Helper functions
├── hooks/               → Custom React hooks (useImageProcessing, useImageStatusWebSocket, etc.)
├── providers/           → Context providers (Redux, Error, MUI)
├── services/            → WebSocket service modules
├── store/
│   ├── slices/          → Redux slices (one per domain)
│   ├── thunks/          → Async thunks (one per domain)
│   ├── hooks.ts         → Typed useAppDispatch / useAppSelector
│   └── store.ts         → Store config, RootState, persistor
├── styles/              → Additional stylesheets
├── types/               → TypeScript type definitions
└── utils/               → Axios instance, error handling, image utils
```

## Patterns & Conventions

### Redux

- One slice + one thunk file per domain (e.g., `generatorSlice.ts` + `generatorThunks.ts`).
- Always use typed hooks: `useAppDispatch()` and `useAppSelector()` from `store/hooks.ts`.
- Thunks use `createAsyncThunk` with `AppThunkConfig` for proper typing.
- Persisted slices: `generator`, `auth`, `ui` — others reset on page reload.
- `RESET_STATE` action clears everything; `RESET_EXCEPT_AUTH` preserves auth.

### API Layer

- Centralized Axios instance in `src/utils/api.ts` with interceptors.
- All endpoints defined in `src/constants/apiEndpoints.ts` using `API_ENDPOINTS` object.
- Base URL from `NEXT_PUBLIC_API_BASE_URL` env var, defaults to `/api` (proxied to backend via Next.js rewrites).
- Auto token refresh on 401 responses.
- 10-minute default timeout (AI operations).

### Components

- Layout: `src/app/layout.tsx` wraps with `ReduxProvider` → `ErrorProvider`.
- Root layout is a `"use client"` component.
- Feature components live directly in `src/components/` (flat structure, not deeply nested).
- Feature-specific sub-components in subdirectories (e.g., `components/edit/`, `components/talent/`).

### Styling

- TailwindCSS v4 as the primary styling approach.
- MUI components for complex UI elements (modals, sliders, tabs).
- Custom CSS in `app/styles.css` and `app/responsive.css` for non-Tailwind styles.
- Font: Nunito (`font-sans` in Tailwind config).

### Image Handling

- Next.js images are `unoptimized: true` (external image proxy handles optimization).
- Remote patterns configured for: Replicate, S3, CloudFront, imgproxy, ibb.co.
- Konva/react-konva used for canvas-based image editing (custom webpack config required).

### API Proxy

- `next.config.ts` rewrites `/api/*` → `http://127.0.0.1:8000/api/*` for local dev.
- In production, the frontend hits the API directly via `NEXT_PUBLIC_API_BASE_URL`.

### Dev Server

- Run: `npm run dev` (port 3000 by default).
- Turbopack available: `npm run dev:turbo`.
