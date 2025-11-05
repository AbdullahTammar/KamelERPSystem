TITLE: KamelERP Front-End Only — Governance, Micro-Frontends & Folder Layout (for Cursor)
SCOPE: Build ONLY the front-end under KamelERPSystem/front-end. Production-grade, micro-frontends by default. React + Vite + TS.

GOALS
- Deliver a production-ready front-end stack (apps + shared packages) with Module Federation (micro-frontends), RTL/i18n, UI kit, forms, grids, reports, and CI/test tooling.
- Output must be runnable via one command (dev) and buildable for production.
- Provide Storybook, lint/typecheck/tests, and performance budgets.

====================================================
0) TECH STACK (PINNED)
====================================================
- React 18 + TypeScript (strict)
- Vite for dev/build + vite-plugin-federation (Module Federation)
- React Router v6
- TailwindCSS (with RTL plugin)
- Ant Design (UI) + headless patterns where needed
- i18n: react-i18next (languages: ar, en) — RTL default
- Data: TanStack Query 5 (server state)
- State (domain/UI): Zustand (default) or Redux Toolkit (if justified)
- Forms: React Hook Form + Zod (validation)
- Data Grid: AG Grid (Enterprise if available)
- Charts: ECharts (preferred) or Highcharts
- Testing: Vitest + React Testing Library, Playwright (E2E)
- Storybook 8
- Lint/Format: ESLint + @typescript-eslint + eslint-plugin-react + eslint-plugin-import + eslint-plugin-jsx-a11y + Prettier
- Boundaries: eslint-plugin-boundaries (module isolation)
- Bundle analysis: @bundle-stats/cli or rollup-plugin-visualizer

====================================================
1) WORKSPACE & FOLDER LAYOUT (ENFORCED)
====================================================
Create and work ONLY inside: KamelERPSystem/front-end

front-end/
  apps/
    host-shell/         # owns top-level routing, auth layout, shared theme/i18n provider
    admin-portal/       # back-office & configuration (CoA, Taxes, Periods, Roles…)
    user-portal/        # daily operations (P2P, O2C, Inventory, HR), Tasks Inbox, Reports
  packages/
    ui-kit/             # design system components, tokens, theming (no business logic)
    i18n/               # i18n init + ar/en resources, number/date localization
    schemas/            # JSON Schemas for forms & reports (shared models)
    utils/              # cross-app helpers (access guards, date/number, query-keys)
    domain-apis/        # typed API clients (temporarily mocked); wrap fetch with Query
  tooling/
    stories/            # global Storybook configs/addons
    configs/            # shared eslint/ts/tailwind/postcss configs
  .eslintrc.*           # root eslint extends from tooling/configs
  tailwind.config.*     # with RTL plugin and design tokens
  tsconfig.base.json    # strict compiler settings + path aliases
  vite.config.ts        # federation defaults for host only (apps have their own)
  package.json          # scripts (lint,typecheck,test,build,preview,storybook)
  README.md             # how to run dev/build; MFE overview

PATH ALIASES (tsconfig.base.json)
- @ui/* -> packages/ui-kit/src/*
- @i18n/* -> packages/i18n/src/*
- @schemas/* -> packages/schemas/src/*
- @utils/* -> packages/utils/src/*
- @api/* -> packages/domain-apis/src/*
- @domain/<module>/* -> apps/*/src/domain/<module>/* (local per app)

BOUNDARIES RULES (eslint-plugin-boundaries)
- apps/* can import from packages/* and their own local code ONLY.
- packages/* MUST NOT import from apps/*.
- ui-kit is presentation-only (no fetch, no TanStack Query).
- domain-apis MUST expose typed hooks that internally use TanStack Query.

====================================================
2) MICRO-FRONTENDS (MODULE FEDERATION)
====================================================
Approach: vite-plugin-federation with 1 host and 2 remotes. Independent builds & deploys.

- host-shell (HOST)
  exposes: none
  consumes: admin-portal, user-portal
  responsibilities:
    - top-level routing (/admin/*, /app/*)
    - auth bootstrap (load token/user/permissions)
    - provide i18n/theme/query-client contexts
    - cross-app navbar/sidebar (slots)

- admin-portal (REMOTE)
  exposed modules:
    - './AdminApp' (root mount for /admin/*)
  features:
    - configuration screens (CoA, Taxes, Fiscal Periods, Roles/Policies)
    - master data editors (items, warehouses, employees, partners)
    - posting engine rules viewer (read-only for now)

- user-portal (REMOTE)
  exposed modules:
    - './UserApp' (root mount for /app/*)
  features:
    - operational flows: P2P, O2C, Inventory, HR (lite)
    - Tasks Inbox (claim/approve/reject, SLAs)
    - Reports viewer (reads Report Spec JSON, tables + charts, export CSV/PDF)

SHARED DEPENDENCIES (singletons/versions aligned)
- react, react-dom, react-router-dom, i18next, react-i18next, @tanstack/react-query,
  zustand, antd, tailwindcss, zod, react-hook-form, ag-grid-react, echarts

ROUTING
- host-shell owns routes:
  - /admin/* -> loads remote admin-portal
  - /app/* -> loads remote user-portal
  - /login, /forbidden -> local
- Code-split per route (React.lazy + Suspense + Skeletons)

AUTH & ACCESS
- host-shell fetches currentUser & policies on load; passes via context
- route guards in host-shell; component/field guards in apps via @utils/access

====================================================
3) UI/UX STANDARDS
====================================================
- RTL default; font: Tajawal; logical CSS (ps/pe/ms/me)
- Skeletons (not spinners) for loading lists & forms
- Empty/Failure states standardized (ui-kit components)
- Data Grid (AG Grid):
  - server-side row model for large lists
  - column state persistence per user
  - inline edit where safe, controlled via RBAC
- Forms:
  - React Hook Form + Zod; JSON Schema adapter in @schemas
  - field-level RBAC props: { canRead, canEdit, mask? }
  - masked rendering for sensitive fields (IBAN, salary)
  - conditional visibility via schema ui:conditions
- Reports viewer:
  - reads Report Spec JSON, renders table + chart
  - export CSV/PDF (client-side for CSV; stub PDF util)
  - schedule button disabled (back-end not in scope now)

====================================================
4) STATE & DATA RULES
====================================================
- Server state ONLY in TanStack Query:
  - key helper: @utils/queryKeys (e.g., qk.finance.journals(params))
  - per-domain defaults for staleTime/cacheTime documented
- Domain/UI state in Zustand:
  - one store per domain (finance/p2p/o2c/inventory/hr)
  - persist only preferences (grid columns, density, theme)
- DO NOT duplicate server data in Zustand.

====================================================
5) SECURITY & ACCESS
====================================================
- Auth context comes from host-shell (mock provider for now)
- Route guards in host-shell; component/field guards via @utils/access
- Prevent rendering of disallowed fields/actions (not just disabled)
- No secrets in FE; env vars prefixed VITE_ only at build time

====================================================
6) i18n & LOCALIZATION
====================================================
- Languages: ar (default), en
- Folder: packages/i18n/src/locales/{ar,en}/domain.json
- Keys format: "<domain>.<page>.<field>.<label|placeholder|hint>"
- Number/date formatting via Intl; currency from context

====================================================
7) DESIGN SYSTEM (ui-kit)
====================================================
- Tokens (colors, spacing, shadows) + Tailwind config
- Components:
  - Layout: AppShell, PageHeader, Toolbar
  - Data: DataTable (AG Grid wrapper), KPIStat, EmptyState, ErrorState
  - Forms: Form, FormField, SelectField, DateField, MoneyField, RefPicker
  - Feedback: Skeleton, Toast, Modal, Drawer
- Storybook stories for each component (default/loading/empty/error)

====================================================
8) TESTING, LINTING, PERFORMANCE
====================================================
- Vitest + RTL for unit; Playwright for E2E (smoke):
  - P2P: create requisition → approve (mock) → see in list
  - O2C: create SO → deliver (mock) → invoice (mock)
  - HR: create leave request → approve (mock)
- ESLint (no warnings), Prettier, TS strict
- Performance budgets (CI gated):
  - gz bundle per remote ≤ 250KB (warn 300KB, fail 350KB)
- Web Vitals reporting (console or optional Sentry)

====================================================
9) SCRIPTS (ROOT package.json under front-end/)
====================================================
- "dev": start host-shell and remotes concurrently (pnpm or npm-run-all)
- "build": build remotes then host
- "preview": serve host build
- "lint", "typecheck", "test", "test:e2e", "storybook", "analyze"

====================================================
10) DELIVERABLE SCREENS (MVP)
====================================================
- host-shell:
  - /login (mock), /forbidden, /admin/*, /app/*
  - Top nav with tenant switch (mock), user menu, language switch (ar/en)
- admin-portal:
  - Finance Config: CoA list + create/edit (client-side mock data)
  - Taxes & Periods pages (forms + lists)
  - Roles & Permissions (read-only matrix demo)
- user-portal:
  - P2P: Requisitions list + create; Approvals list (mock)
  - O2C: Sales Orders list + create; Deliveries list (mock)
  - Inventory: Items list; Stock On Hand (mock)
  - HR: Leave Requests list + create
  - Tasks Inbox: unified tasks (mock data)
  - Reports: viewer reading JSON Spec from @schemas

====================================================
11) ACCEPTANCE CRITERIA (FRONT-END ONLY)
====================================================
- Vite + Module Federation working: host consumes remotes at runtime
- All apps build & run; routes resolve; i18n/RTL functional
- Data grid, forms, and reports viewer operate with mock APIs
- Guards work: route guard + component/field-level guards
- Storybook renders ui-kit components with documented states
- CI passes: lint, typecheck, unit tests; Playwright smoke runs locally
- Bundle budgets respected; analyze report generated

====================================================
12) STEP-BY-STEP (WHAT TO OUTPUT NOW)
====================================================
1) Create the folder structure above under KamelERPSystem/front-end.
2) Initialize package.json, tsconfig.base.json, Tailwind config, ESLint/Prettier.
3) Scaffold apps (host-shell, admin-portal, user-portal) with vite-plugin-federation:
   - host exposes nothing; remotes expose ./AdminApp and ./UserApp
   - host routes /admin/* and /app/* to the respective remotes
4) Create packages (ui-kit, i18n, schemas, utils, domain-apis) with exports and examples.
5) Implement sample screens listed in Section 10 using mocks in domain-apis.
6) Add Storybook config and sample stories for ui-kit and at least one page per app.
7) Provide root README with:
   - how to run dev: `pnpm i && pnpm dev` (or npm)
   - how to build: `pnpm build` then `pnpm preview`
   - federation diagram & contracts
8) Print:
   - final folder tree
   - federation config snippets (host/remotes)
   - dev/build commands
   - how to switch language and where i18n files live

NOTES
- Keep business logic out of ui-kit.
- Do not fetch directly from components; use domain-apis + TanStack Query hooks.
- Prefer skeletons over spinners; empty/error standardized components.
- Arabic is default; ensure proper RTL layout in all screens.

END OF FILE
