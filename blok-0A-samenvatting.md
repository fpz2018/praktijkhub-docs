# Blok 0A — Samenvatting

**Datum:** 2026-04-30
**Status:** Afgerond
**Doel:** Werkmap opzetten, repo's verplaatsen, docs op de juiste plek, eerste verkenning.

---

## Uitgevoerd

| # | Stap | Resultaat |
|---|---|---|
| 1 | Werkmap `C:\Users\marc\praktijkhub\` aangemaakt met submap `docs\` | ✅ |
| 2 | `Werkportaal` verplaatst van OneDrive naar `praktijkhub\werkportaal\` | ✅ |
| 3 | `fysiopro-kennisbank` verplaatst naar `praktijkhub\fysiopro-kennisbank\` | ✅ |
| 4 | Beide repo's gevalideerd: branch `main`, working tree clean, upstream intact | ✅ |
| 5 | 5 docs-bestanden door Marc in `docs/` geplaatst | ✅ |
| 6 | Docs gelezen: `02_HUIDIGE_SITUATIE`, `03_ARCHITECTUUR`, `04_FASE_1_WERKPORTAAL_AUTH_MIGRATIE`, `06_BESLISSINGEN` | ✅ |
| 7 | README's gelezen van beide repo's | ✅ |

---

## Samenvatting Claude — wat ik zie

| Element | Status |
|---|---|
| Doelarchitectuur | Helder vastgelegd (één Supabase-project, FysioPro = hoofd, drie frontends) |
| Fase 1-plan | Concreet: backup → test-Supabase → `auth_user_id`-kolom → magic link → echte RLS |
| Werkportaal | Vanilla JS, ±2000 regels in `app.js`, hardcoded anon-key, RLS `using (true)`, PIN-flow |
| FysioPro | React + Vite + Supabase Auth + RLS — gezonde basis, dient als model |
| Beslissingen vastgelegd | B-001 t/m B-010, helder en consistent |

### Zaken die opvielen

1. **README Werkportaal = v9, docs zeggen v11.** README mogelijk verouderd. Migratiebestanden v10-v13 staan wel in de repo-root.
2. **FysioPro `README.md` is nog de Base44-template.** De echte staat in `README.new.md`.
3. **FysioPro repo bevat `node_modules/`, `dist/`, `fysiopraktijk-zeist-v3.zip`, `import_sql.zip`** — horen niet in git.
4. **`team.assigned_to` is `TEXT[]` van team-IDs.** Werkt, maar wankel na migratie: nieuwe code moet weten dat dit géén `auth.users.id` is. Risico op verwarring bij PraktijkHub-bouw.

### Open beslissingen die fase 1 raken

- Welk Supabase-project wordt hoofdproject?
- Early adopter voor fase 1-test?
- Stagiaire-mailbox `stage@fysiopraktijkzeist.nl` aanmaken?

### Openstaande vragen

1. Werkportaal Supabase-wachtwoord beschikbaar voor `pg_dump`?
2. Bestaat er al een test-Supabase-project?
3. RLS-performance: subquery `auth.uid()` ↔ `team.auth_user_id` of optimalisatie via JWT-claim?

---

## Antwoorden + beslissingen Marc

### Op observaties

- README Werkportaal verouderd: klopt, geen blocker.
- FysioPro README + repo-rommel: opruim voor later, geen prio.
- `assigned_to TEXT[]`: belangrijk punt. Zie beslissing B-013 hieronder.

### Op vragen

1. **Werkportaal-wachtwoord:** wordt gecheckt in 1Password tijdens blok 0B.
2. **Test-Supabase:** bestaat nog niet. Wordt aangemaakt in blok 0B.
3. **RLS-performance:** SIMPEL houden. Subquery is prima bij 9 users. Geen JWT-claim-optimalisatie.

### Nieuwe beslissingen (toe te voegen aan `06_BESLISSINGEN.md`)

#### B-011: Hoofdproject = FysioPro Supabase
**Datum:** 2026-04-30
**Beslissing:** Het FysioPro Kennisbank Supabase-project wordt het hoofdproject voor het ecosysteem. Werkportaal-data migreert hier naartoe in fase 2.
**Reden:** FysioPro heeft al echte Supabase Auth + RLS-policies; Werkportaal moet zich daarnaar conformeren.
**Consequentie:** Alle nieuwe migraties en PraktijkHub-tabellen worden in dit project gemaakt.

#### B-012: Marc is early adopter voor fase 1-test
**Datum:** 2026-04-30
**Beslissing:** Marc de Jong (marc@fysiopraktijkzeist.nl) test als eerste de magic link-flow voordat het team wordt uitgerold.
**Reden:** Eigenaar + technisch onderlegd; kan pijnpunten herkennen vóór uitrol naar 8 anderen.
**Consequentie:** Cutover-volgorde: Marc test → fix → 1 collega test → fix → volledige uitrol.

#### B-013: `assigned_to` migreren van `team.id[]` naar `auth_user_id[]` tijdens fase 1
**Datum:** 2026-04-30
**Context:** Huidige `assigned_to` in `apps`-tabel is `TEXT[]` van `team.id`-waarden. Na auth-migratie blijft dit werken, maar levert verwarring op bij PraktijkHub-bouw (twee soorten user-IDs naast elkaar).
**Beslissing:** Eenmalige migratie tijdens fase 1: `assigned_to` herschrijven naar array van `auth_user_id`-waarden via mapping-tabel.
**Reden:** Eén bron van waarheid voor user-identiteit door het hele ecosysteem.
**Consequentie:** Sub-stap toevoegen aan fase 1-plan. Concrete uitwerking in blok 1A.

### Stagiaire-mailbox

`stage@fysiopraktijkzeist.nl` moet nog aangemaakt worden — gepland in blok 1A.

---

## Status

**Blok 0A afgerond.**

Volgende stap: blok 0B — backups maken + test-Supabase aanmaken (op Marc's prompt).
