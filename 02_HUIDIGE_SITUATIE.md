# Huidige Situatie — Werkportaal & FysioPro Kennisbank

> Analyse van de twee bestaande zelfgebouwde apps van Marc, op basis waarvan PraktijkHub wordt gebouwd.

---

## Werkportaal Fysiopraktijk Zeist

**Versie:** 11.0 (Supabase Realtime) — 18 maart 2026
**Doel:** Centraal startscherm/dashboard voor medewerkers met snelle toegang tot werkapplicaties.
**Repo:** GitHub (privé)

### Stack

| Component | Technologie |
|---|---|
| Frontend | Vanilla HTML5 + JavaScript (geen framework) |
| Styling | CSS3 met custom properties, dark mode |
| Hosting | Netlify (gratis tier) |
| Database | Supabase (PostgreSQL, gratis tier) |
| Realtime | Supabase Realtime (WebSocket) |
| Email | Supabase Edge Functions + Resend |
| PWA | manifest.json + service worker (cache v15) |
| CDN-libs | Font Awesome 6.5, Google Fonts (Montserrat/Roboto), SortableJS 1.15.3, @supabase/supabase-js v2 |
| Weer | Open-Meteo API (gratis, geen key) |

### Architectuur

Single-Page Application met PWA. Geen router. State-gedreven schermen in `index.html` (≈654 regels):
Setup-screen (DB-config) → Gate-screen (poortcode) → User-select (teamlid) → Main app

```
├── index.html              # Single entry point
├── css/style.css
├── js/
│   ├── supabase-api.js     # REST-wrappers, config-management
│   └── app.js              # UI-events, auth, realtime (~2000 regels)
├── supabase/functions/send-invite/index.ts   # Edge Function (Resend)
├── supabase-schema.sql     # Basis-schema
└── supabase-migration-v9..v13-taken.sql      # Migraties
```

### Authenticatie (HUIDIG — wordt vervangen)

Tweelaags, volledig **PIN-gebaseerd**. Geen OAuth/magic-link/Supabase Auth.

1. **Poortcode (Gate)** — 6 cijfers, SHA-256-hash in `settings`. Max 3 pogingen → exponentiële lockout (60s × 2ⁿ). Sessie via `localStorage.fpz_gate_verified`.
2. **PIN per teamlid** — 6 cijfers, SHA-256 + vaste salt `fpz_salt_2026`. Brute-force-tracking per apparaat in `localStorage.fpz_pin_lockout`. Max 3 pogingen → 30s × 2ⁿ.

Rolbeheer via `team.is_admin` (boolean) + `roles`-tabel (Eigenaar, Fysiotherapeut, Admin, Stagiaire). E-mailuitnodigingen via eenmalige `invite_tokens` (7 dagen TTL).

### KRITIEK BEVEILIGINGSPROBLEEM

- Alle RLS-policies in `supabase-schema.sql` zijn `"Allow all" (true)`
- Supabase anon-key staat **hardcoded** in `js/supabase-api.js`
- Effectieve beveiliging is **alleen client-side** (poortcode + PIN-lockout)
- Iedereen met de anon-key kan via Postman direct alle data lezen/schrijven

**Dit moet opgelost worden in fase 1 — en is een primair argument voor de migratie.**

### Database-schema Werkportaal

| Tabel | Belangrijkste kolommen | Opmerking |
|---|---|---|
| `team` | id (UUID), name, role, email, avatar_color, is_admin, pin_hash, invite_sent_at, birthday | Trigger `updated_at` |
| `invite_tokens` | token (UNIQUE), member_id → team.id, email, used, used_at, expires_at | Indexen op token + member_id |
| `categories` | id, name, icon (FA), color, sort_order | |
| `apps` | id, name, url, icon, color, category, description, notes, sort_order, assigned_to (TEXT[]) | `assigned_to` = array van team-IDs |
| `announcements` | id, title, content, author, priority (normaal/belangrijk/urgent), active | |
| `roles` (v10) | id, name (UNIQUE), sort_order | Seed: 4 standaardrollen |
| `taken` (v13) | takenlijst | Toegevoegd in laatste migratie |

**Triggers:** gemeenschappelijke `update_updated_at()` op team/categories/apps/announcements/roles.
**Seed-data:** eigenaar "Marc" als admin, ~8 standaardcategorieën (EPD, Microsoft, Communicatie, etc.).

### Aantal gebruikers

**9 totaal:**
- 8 met werkmail `@fysiopraktijkzeist.nl`
- 1 stagiaire zonder werkmail (oplossing nodig in migratieplan)

---

## FysioPro Kennisbank

**Repo:** https://github.com/fpz2018/fysiopro-kennisbank (privé)
**Doel:** Kennisbank, AI-assistent, PDCA, planning, bestellingen, scholingen voor de praktijk.

### Stack

**Frontend & build:**
- React 18.2 + Vite 6.1 (SPA)
- React Router 6.26
- Tailwind CSS 3.4 + shadcn/ui (New York style) op Radix UI (~30 components)
- TanStack React Query 5.84
- React Hook Form 7.54 + Zod 3.24
- Framer Motion, Recharts, Lucide, React Markdown
- Documentverwerking: Mammoth (Word), pdf-parse, jsPDF, xlsx

**Backend & deployment:**
- Supabase 2.95 (Postgres + Auth + Storage + Edge Functions)
- Deno edge functions in `/functions` en `/supabase/functions`
- Netlify (netlify.toml met SPA-rewrites)
- LLM: Base44 SDK 0.8.6 (legacy RAG) + Claude/multi-LLM

### Architectuur

```
src/
├── pages/        Home, Login, Assistent, Kennisbank, PDCA, Planning,
│                 Bestellingen, Scholingen, Beheer, MijnZaken, …
├── components/   ui/ (shadcn), shared, admin, planning, todos, chat,
│                 audit, bestellingen, pdca, account, bbsc
├── api/          supabaseApi.js (CRUD abstractie)
├── lib/          AuthContext.jsx, supabaseClient.js, ErrorBoundary
├── hooks/        custom hooks
└── utils/
functions/        Deno edge: askQuestion, importFromGoogleDrive,
                  importFromOneDrive, checkDeadlineHerinneringen
supabase/migrations/  20260318_*, 20260324_*, 20260429_*
```

App.jsx → AuthProvider → Router met LoginGuard voor protected routes.
Frontend praat direct met Supabase via JS-client (RLS enforces); zware/asynchrone taken via Edge Functions.

### Authenticatie (BEHOUDEN — wordt model voor Werkportaal)

- **Supabase Auth** (email + wachtwoord)
- Sessie persistent in `localStorage` onder key `fysiopraktijk-auth`, auto-refresh
- Password-reset flow met hash-based recovery
- `public.users` spiegelt `auth.users` met `role` (admin/user) en `is_active`
- Gedeactiveerde users worden geweigerd
- AuthContext levert: `user`, `userProfile`, `isAuthenticated`, `authError`, `isLoadingAuth`
- Toegang via combinatie van LoginGuard (frontend) + RLS-policies (backend)

**Belangrijke bestanden:** `src/lib/AuthContext.jsx`, `src/lib/supabaseClient.js`.

### Database-schema FysioPro (kerntabellen, alle met RLS)

| Tabel | Belangrijkste kolommen |
|---|---|
| `users` | id (FK auth.users), email, full_name, role, is_active |
| `documents` | title, content, tags[], access_level (all/admin_only), file_url, uploaded_by, is_active |
| `chat_history` | user_id, question, answer, sources[], suggested_questions[] |
| `meldingen` (PDCA) | type (klacht/incident/datalek/verbetering/feedback), urgency, status, assigned_to, reported_by, deadline, resolution |
| `sop_flows` / `sop_completions` | SOP-stappen (JSONB) + voortgang per user |
| `voorbeeld_vragen` | vraag, categorie, display_order |
| `bbsc_doelen` / `bbsc_evaluaties` | Balanced Scorecard: perspectief, KPI, target/current, verantwoordelijke |
| `planning_items` | titel, type, datum, deadline, actiepunten (JSONB), gerelateerd_document |
| `bestelbare_items` / `bestellingen` | catalogus + aanvragen (status: aangevraagd/goedgekeurd/afgewezen/geleverd) |
| `inventaris_items` / `inventaris_uitgiftes` | uitgifte/inlevering, akkoord_medewerker |
| `scholingen`, `bezit_checks` | training- en compliance-tracking |
| `app_settings` | configuratie key/value |
| `sync_sources`, `sync_folders`, `sync_filename_patterns` | OneDrive/GDrive sync-config |

**RLS-patroon:** users zien hun eigen rijen; admins zien/beheren alles; documents zichtbaar afhankelijk van access_level.

**Storage:** Supabase Storage; bucket-fixes in `SQL_FIX_STORAGE_BUCKET.sql` en `SQL_FIX_DOCUMENTS_DELETE_STORAGE.sql`.

**Belangrijk om te raadplegen:** `supabase/migrations/20260318_fix_all_rls_policies.sql`, `SQL_QUERY_1_TABELLEN.sql` t/m `SQL_QUERY_5_KENNISTOEGANG.sql`, `functions/askQuestion.ts`.

---

## Vergelijking & gevolg voor migratie

| Aspect | Werkportaal (huidig) | FysioPro (huidig) | PraktijkHub (doel) |
|---|---|---|---|
| Frontend | Vanilla HTML/JS | React + Vite + Tailwind + shadcn | React + Vite + Tailwind + shadcn |
| Auth | Eigen PIN | Supabase Auth | Supabase Auth (gedeeld met FysioPro) |
| Users-tabel | `team` (eigen UUID) | `users` (FK `auth.users`) | `users` (gedeeld met FysioPro) |
| RLS | Open (`true`) — onveilig | Echte policies | Echte policies |
| Beveiliging | Client-side PIN — onveilig | Server-side RLS + Auth | Server-side RLS + Auth |
| Supabase-project | Eigen project | Eigen project | **GEDEELD project (na fase 2)** |

**Conclusie:** FysioPro is de gezonde basis. Werkportaal moet zich daarnaar conformeren. Daarna kan PraktijkHub als derde frontend ernaast gebouwd worden.
