# Blok 1B — Briefing voor Claude Code

**Datum:** 2026-05-11
**Status:** Klaar om te starten
**Doel:** Database-migratie in TEST-Supabase — `team`-tabel koppelen aan `auth.users` en `assigned_to`-kolom omzetten naar `auth_user_id`-array (B-013).

> **Hoe te gebruiken:** open Claude Code in `~/praktijkhub/`, plak de prompt onderaan dit document, en wacht op de eerste bevestiging voordat je SQL laat draaien.

---

## Uitgangspositie

Wat staat er al (uit blok 0B + 1A):

- TEST-Supabase project ref: `orojitrzpapjyrkbebdy` (eu-west-1 Ierland)
- Werkportaal-data geïmporteerd: `team=9`, `apps=33`, `categories=15`, `announcements=1`
- 8 `auth.users`-accounts aangemaakt voor de werkmail-collega's (Yvonne/stagiaire geparkeerd tot blok 1E)
- Email provider aan, signup/confirm uit, magic-link template in NL
- Werkportaal-repo lokaal in `~/praktijkhub/werkportaal/`, geen feature-branch aanwezig

Wat NIET aangeraakt wordt in dit blok:

- Productie-Supabase (Werkportaal én FysioPro)
- Werkportaal-frontend (`app.js`, `js/supabase-api.js`, `sw.js`) — komt in blok 1C
- RLS-policies omschrijven — komt in blok 1D
- Yvonne/stagiaire-mailbox — blok 1E

---

## Scope blok 1B

| # | Sub-stap | Resultaat |
|---|---|---|
| 1 | `team`-tabel: voeg `auth_user_id uuid references auth.users(id)` toe (nullable) | Kolom bestaat |
| 2 | Index `idx_team_auth_user_id` op `team(auth_user_id)` | Snellere lookups |
| 3 | Unique constraint op `team.email` (nodig voor mapping én voorkomen duplicate accounts) | Constraint actief |
| 4 | Email-match: `update team t set auth_user_id = au.id from auth.users au where t.email = au.email` | 8 rijen gekoppeld |
| 5 | Verificatie: lijst teamleden zonder `auth_user_id` | Verwacht: alleen Yvonne |
| 6 | B-013 voorbereiden: maak mapping-view of helper-query `team.id` → `team.auth_user_id` | Mapping zichtbaar |
| 7 | B-013 uitvoeren: herschrijf `apps.assigned_to` van `team.id`-strings naar `auth_user_id`-strings | Alle 33 apps geüpdatet |
| 8 | Verificatie B-013: geen `apps.assigned_to`-waarde meer die naar oude `team.id` wijst | 0 mismatches |
| 9 | Snapshot-dump van TEST-Supabase nemen (`pg_dump`) als tussenbackup vóór blok 1C | `.sql`-bestand in `backups/` |

---

## SQL-skeletten (Claude Code mag aanvullen/verbeteren)

```sql
-- Stap 1 t/m 3
alter table team add column auth_user_id uuid references auth.users(id);
create index idx_team_auth_user_id on team(auth_user_id);
alter table team add constraint team_email_unique unique(email);

-- Stap 4 — email-match
update team t
set auth_user_id = au.id
from auth.users au
where t.email = au.email;

-- Stap 5 — verificatie
select name, email, auth_user_id
from team
where auth_user_id is null;
-- Verwacht: 1 rij (Yvonne)

-- Stap 6 — mapping (handig als view)
create or replace view team_auth_map as
select id as team_id, auth_user_id, email, name
from team;

-- Stap 7 — apps.assigned_to migratie (B-013)
-- LET OP: assigned_to is text[] met team.id als strings.
-- We zetten elke string om naar de overeenkomstige auth_user_id-string.
update apps a
set assigned_to = (
  select coalesce(array_agg(t.auth_user_id::text), '{}')
  from unnest(a.assigned_to) as old_id
  left join team t on t.id::text = old_id
  where t.auth_user_id is not null
);

-- Stap 8 — verificatie B-013
-- Zoek apps waarin nog een oude team.id-waarde voorkomt
select a.id, a.name, a.assigned_to
from apps a
where exists (
  select 1
  from unnest(a.assigned_to) as val
  where val::uuid in (select id from team)
);
-- Verwacht: 0 rijen
```

**Aandachtspunt B-013:** als een `apps.assigned_to`-rij verwijst naar Yvonne (zonder `auth_user_id`), valt die uit de array. Maak die rij zichtbaar vóór de update en parkeer of behandel apart.

```sql
-- Pre-check: welke apps verwijzen naar leden zonder auth_user_id?
select a.id, a.name, a.assigned_to
from apps a
where exists (
  select 1 from unnest(a.assigned_to) as old_id
  join team t on t.id::text = old_id
  where t.auth_user_id is null
);
```

---

## Veiligheids- en procesregels

- Werk alléén op TEST-project `orojitrzpapjyrkbebdy`.
- Géén SQL op productie deze sessie.
- Voor elke `update`/`alter`: laat Claude Code éérst de SQL tonen en wacht op Marc's "ga".
- Voor stap 9: backup met `pg_dump` naar `~/praktijkhub/backups/test_blok1B_YYYYMMDD.sql` (zit in `.gitignore`).
- Connection-info uit 1Password. Geen wachtwoorden in chat of in code.

---

## Definition of Done — blok 1B

- [ ] `team.auth_user_id` kolom + index + unique constraint op email aanwezig
- [ ] 8 van 9 teamleden gekoppeld via email-match (Yvonne `null`, conform plan)
- [ ] `apps.assigned_to` bevat alleen `auth_user_id`-waarden of lege arrays
- [ ] Verificatie-queries leveren 0 mismatches
- [ ] Backup `test_blok1B_<datum>.sql` aanwezig in `backups/`
- [ ] `blok-1B-samenvatting.md` geschreven (Marc commit/pusht via GH Desktop)
- [ ] `06_BESLISSINGEN.md` aangevuld als er nieuwe beslissingen ontstaan tijdens uitvoering

---

## Prompt om in Claude Code te plakken

```
Hoi Claude,

We werken vandaag aan blok 1B van het PraktijkHub-project. Lees eerst:
- docs/04_FASE_1_WERKPORTAAL_AUTH_MIGRATIE.md (stap 1.2–1.4)
- docs/06_BESLISSINGEN.md (B-013 specifiek)
- docs/blok-1A-samenvatting.md (uitgangspositie)
- docs/blok-1B-briefing.md (deze briefing — scope, sql-skeletten, DoD)

Doel blok 1B: database-migratie in TEST-Supabase (orojitrzpapjyrkbebdy).
- team-tabel koppelen aan auth.users via email-match
- apps.assigned_to omzetten van team.id[] naar auth_user_id[] (B-013)
- Backup nemen na afloop

Werkwijze:
1. STOP voor je SQL uitvoert. Vat eerst samen wat je gaat doen.
2. Stel 1–3 verduidelijkingsvragen als iets onduidelijk is.
3. Voor elke SQL-statement: toon de query, wacht op mijn "ga", dan pas uitvoeren.
4. Werk uitsluitend tegen TEST-project (orojitrzpapjyrkbebdy). NIET tegen productie.
5. Verificatiequeries na elke stap. Toon resultaat aan mij.
6. Aan het einde: schrijf docs/blok-1B-samenvatting.md in hetzelfde format als blok-1A.

Begin met: bevestigen dat je de briefing begrepen hebt + samenvatten welke 9 stappen je gaat doen + vragen of de connection-string voor TEST klaar staat.
```

---

## Volgende stappen (na blok 1B)

- **Blok 1C** — Werkportaal-frontend ombouwen: env-vars + magic-link login + sessie-handling + service worker bumpen.
- **Blok 1D** — RLS-policies herschrijven (geen `using (true)` meer).
- **Blok 1E** — Stagiaire-mailbox `stage@fysiopraktijkzeist.nl` aanmaken + Yvonne koppelen.
- **Blok 1F** — Acceptatietest + productie-cutover.
