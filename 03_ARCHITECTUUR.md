# Architectuur — PraktijkHub-ecosysteem

> Doelbeeld na alle 4 fasen, gevolgd door fase-voor-fase plan.

---

## Eindbeeld (na ±3 maanden)

```
┌────────────────────────────────────────────────────────────────┐
│              ÉÉN SUPABASE-PROJECT                              │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐     │
│  │  auth.users (Supabase Auth)                          │     │
│  │  └─ public.users (profiel, role, is_active, ...)    │     │
│  └──────────────────────────────────────────────────────┘     │
│                                                                │
│  Werkportaal-tabellen   FysioPro-tabellen   PraktijkHub-tabellen│
│  ├─ team*               ├─ documents        ├─ onderwerpen     │
│  ├─ apps                ├─ chat_history     ├─ berichten       │
│  ├─ categories          ├─ meldingen        ├─ gesprekken      │
│  ├─ announcements       ├─ sop_flows        ├─ projecten       │
│  ├─ taken               ├─ bbsc_*           ├─ leden           │
│  └─ ...                 └─ ...              └─ snelkoppelingen │
│                                                                │
│  *team wordt gemerged met public.users in fase 1               │
│                                                                │
│  Echte RLS-policies op ALLE tabellen                          │
└────────────────────────────────────────────────────────────────┘
        ▲                  ▲                    ▲
        │                  │                    │
   ┌────┴──────┐    ┌──────┴────┐       ┌──────┴───────┐
   │Werkportaal│    │ FysioPro  │       │  PraktijkHub │
   │vanilla JS │    │  React    │       │    React     │
   │(gemigreerd│    │ (bestaand)│       │   (nieuw)    │
   │  in fase 1)│   │           │       │              │
   └───────────┘    └───────────┘       └──────────────┘
        │                  │                    │
        └──────────────────┴────────────────────┘
                           │
                    Supabase Auth
                  (gedeelde sessie)
                  (Single Sign-On)

   Alle drie op Netlify, eigen subdomein:
   • werkportaal.fysiopraktijkzeist.nl
   • kennisbank.fysiopraktijkzeist.nl
   • hub.fysiopraktijkzeist.nl
```

## Wat dit oplevert

- **Eén keer inloggen** → toegang tot alle drie via gedeelde sessie
- **Werkportaal** kan in PraktijkHub berichten "FysioPro-artikel X is bijgewerkt" via realtime
- **PraktijkHub** kan in zijn snelkoppelingen-paneel direct linken naar FysioPro & Werkportaal
- **Geen code samenvoegen** — drie kleine repo's blijven klein, gescheiden onderhoud
- **Echte AVG-compliance** — geen open RLS meer in Werkportaal

## Trade-offs van deze architectuur

**Voordelen:**
- Eén bron van waarheid voor users
- Gedeelde sessie = geen drie keer inloggen
- Cross-app realtime mogelijk
- SaaS-klaar (multi-tenant kan later via `practice_id` toegevoegd worden)

**Nadelen:**
- Eén Supabase-project = single point of failure
- Migratie van Werkportaal-data heeft risico (mitigatie: backup + rollback-strategie)
- Drie repo's vereisen drie keer deploy bij grote wijzigingen
- Stagiaire zonder `@fysiopraktijkzeist.nl` mail vereist aparte oplossing

---

## Fasering — overzicht

| Fase | Wat | Doorlooptijd | Outcome |
|---|---|---|---|
| **0** | Voorbereiding & backup | 2-3 dagen | Veilig kunnen starten |
| **1** | Werkportaal-auth migreren naar Supabase Auth | 2-3 weken | Werkportaal werkt met Supabase Auth ipv PIN |
| **2** | Supabase-projecten samenvoegen | 1 week | Alles in één Supabase-project |
| **3** | PraktijkHub v1 bouwen | 6-8 weken | Chat + onderwerpen + berichten + bestanden live |
| **4** | PraktijkHub v2 | 4-6 weken | Projecten + snelkoppelingen + integraties |

**Totaal: ±13-18 weken naast lopende projecten.**

---

## Fase 0 — Voorbereiding (2-3 dagen)

**Doel:** veilig kunnen starten zonder iets kapot te maken.

- [ ] Volledige backup van Werkportaal Supabase-project (`pg_dump`)
- [ ] Volledige backup van FysioPro Supabase-project (`pg_dump`)
- [ ] Beslissen welk Supabase-project het "hoofdproject" wordt (advies: **FysioPro**, want het heeft al echte auth + RLS)
- [ ] Stagiaire-oplossing kiezen (zie BESLISSINGEN.md)
- [ ] Lokale dev-omgeving voor beide bestaande apps werkend
- [ ] Test-Supabase-project aanmaken voor experimenten

---

## Fase 1 — Werkportaal-auth migreren (2-3 weken)

**Doel:** Werkportaal werkt met Supabase Auth in plaats van PIN, met behoud van data.

Detail in `04_FASE_1_WERKPORTAAL_AUTH_MIGRATIE.md`.

**Hoofdstappen:**
1. Nieuwe `auth.users`-accounts aanmaken voor de 9 teamleden (8x werkmail + 1x stagiaire-oplossing)
2. `team`-tabel uitbreiden met `auth_user_id` kolom (FK naar `auth.users`)
3. Mapping maken: bestaande `team.id` ↔ nieuwe `auth.users.id`
4. RLS-policies herschrijven van `true` naar echte rule-based policies
5. PIN-flow in `app.js` vervangen door Supabase Auth-flow
6. Hardcoded anon-key netjes via env-variabelen
7. Service worker updaten (cache versie ophogen)
8. Magic-link login implementeren (geen wachtwoord nodig — beter UX voor niet-technici)
9. Test met testaccounts → uitrol naar team

---

## Fase 2 — Supabase-projecten samenvoegen (1 week)

**Doel:** Werkportaal-tabellen migreren naar het FysioPro-project zodat alles in één database staat.

**Hoofdstappen:**
1. Schema-export van Werkportaal-project (`pg_dump --schema-only`)
2. Tabellen prefixen of in eigen schema (`werkportaal_team`, `werkportaal_apps`, etc.) om naamconflicten te voorkomen
3. Data-export van Werkportaal-project (`pg_dump --data-only`)
4. Schema importeren in FysioPro-project
5. Data importeren met user-ID-mapping uit fase 1
6. RLS-policies overzetten en aanpassen
7. Werkportaal-frontend `.env` updaten naar nieuwe Supabase-URL + anon-key
8. Test alle Werkportaal-functies in nieuwe omgeving
9. Cutover (oude project op `read-only` → nieuwe project live)
10. Oude Werkportaal-Supabase-project archiveren (niet meteen verwijderen — 30 dagen wachten)

---

## Fase 3 — PraktijkHub v1 bouwen (6-8 weken)

**Doel:** Werkende chat-app voor het team, mobile-first, in NL.

**Stack:** React 18 + Vite + Tailwind + shadcn/ui + Supabase Realtime.

### Database-schema PraktijkHub v1

```sql
-- Onderwerpen (= Slack channels)
create table praktijkhub_onderwerpen (
  id uuid primary key default gen_random_uuid(),
  naam text not null,
  beschrijving text,
  type text check (type in ('open', 'besloten')) default 'open',
  aangemaakt_door uuid references auth.users(id),
  aangemaakt_op timestamptz default now(),
  gearchiveerd boolean default false
);

-- Lidmaatschap onderwerpen (wie zit in welk onderwerp)
create table praktijkhub_onderwerp_leden (
  onderwerp_id uuid references praktijkhub_onderwerpen(id) on delete cascade,
  user_id uuid references auth.users(id) on delete cascade,
  rol text check (rol in ('lid', 'beheerder')) default 'lid',
  toegevoegd_op timestamptz default now(),
  primary key (onderwerp_id, user_id)
);

-- Berichten (in onderwerpen, of als DM)
create table praktijkhub_berichten (
  id uuid primary key default gen_random_uuid(),
  onderwerp_id uuid references praktijkhub_onderwerpen(id) on delete cascade,
  dm_kanaal_id uuid references praktijkhub_dm_kanalen(id) on delete cascade,
  ouder_bericht_id uuid references praktijkhub_berichten(id), -- voor gesprekken (threads)
  auteur_id uuid references auth.users(id) not null,
  inhoud text not null,
  bewerkt_op timestamptz,
  verwijderd_op timestamptz,
  aangemaakt_op timestamptz default now(),
  check (
    (onderwerp_id is not null and dm_kanaal_id is null) or
    (onderwerp_id is null and dm_kanaal_id is not null)
  )
);

-- Direct Message kanalen (= Slack DMs)
create table praktijkhub_dm_kanalen (
  id uuid primary key default gen_random_uuid(),
  type text check (type in ('1op1', 'groep')) not null,
  naam text, -- alleen voor groeps-DMs
  aangemaakt_op timestamptz default now()
);

create table praktijkhub_dm_leden (
  dm_kanaal_id uuid references praktijkhub_dm_kanalen(id) on delete cascade,
  user_id uuid references auth.users(id) on delete cascade,
  primary key (dm_kanaal_id, user_id)
);

-- Bestanden bij berichten
create table praktijkhub_bijlagen (
  id uuid primary key default gen_random_uuid(),
  bericht_id uuid references praktijkhub_berichten(id) on delete cascade,
  bestandsnaam text not null,
  storage_pad text not null, -- pad in Supabase Storage
  bestandstype text,
  grootte_bytes int,
  aangemaakt_op timestamptz default now()
);

-- Reacties (emoji-reactions op berichten)
create table praktijkhub_reacties (
  bericht_id uuid references praktijkhub_berichten(id) on delete cascade,
  user_id uuid references auth.users(id) on delete cascade,
  emoji text not null,
  primary key (bericht_id, user_id, emoji)
);

-- Gelezen-status per gebruiker per onderwerp
create table praktijkhub_gelezen (
  user_id uuid references auth.users(id) on delete cascade,
  onderwerp_id uuid references praktijkhub_onderwerpen(id) on delete cascade,
  laatst_gelezen_op timestamptz default now(),
  primary key (user_id, onderwerp_id)
);
```

### Schermen v1

1. **Inlog-scherm** — magic link of wachtwoord
2. **Hoofdscherm (mobiel)** — bottom-tabs: Onderwerpen | Berichten | Zoeken | Profiel
3. **Onderwerpen-lijst** — alle onderwerpen waar ik in zit, ongelezen-badges
4. **Onderwerp-detail** — berichten chronologisch, input onderaan, +-knop voor bijlage
5. **Berichten-lijst (DMs)** — overzicht 1-op-1 en groeps-DMs
6. **DM-detail** — zelfde als onderwerp-detail
7. **Nieuw onderwerp aanmaken** — naam, beschrijving, type, leden uitnodigen
8. **Profiel & instellingen** — naam, avatar, notificatievoorkeuren

### Wat NIET in v1

- Projecten (komt in v2)
- Snelkoppelingen-paneel (komt in v2)
- Integraties met FysioPro/Werkportaal (komt in v2)
- Spraakberichten
- Video-calls
- Externe gasten

---

## Fase 4 — PraktijkHub v2 (4-6 weken)

**Doel:** "Eén app voor alles"-gevoel realiseren.

**Toegevoegde features:**
- **Projecten** — aparte sectie, takenlijsten, betrokkenen, deadlines, gekoppeld aan onderwerpen
- **Snelkoppelingen-paneel** — directe links naar Werkportaal, FysioPro, FysioRoadmap, etc. met SSO
- **Cross-app notificaties** — Werkportaal kan via Supabase Realtime een bericht in PraktijkHub plaatsen
- **Zoeken** — globaal zoeken in berichten, onderwerpen, bestanden
- **Push-notificaties** — via Web Push API (PWA)
- **Vermeldingen (@naam)** — met notificatie

---

## Veiligheids- & AVG-overwegingen (gelden voor alle fasen)

- Geen open RLS-policies (`true`) toegestaan
- Anon-key alleen via env-variabelen, nooit in code
- Service-role-key alleen in Edge Functions, nooit in client
- Patiëntdata komt NIET in PraktijkHub-berichten (verwijzing via FysioPro-artikel-ID is OK; vrije tekst over patiënten niet)
- Audit-log voor admin-acties (nieuwe gebruikers, verwijderde berichten)
- Bewaartermijn berichten: configureerbaar per onderwerp (default: oneindig, verwijderbaar door beheerder)
- Storage-buckets: alleen toegankelijk voor leden van het onderwerp/DM
