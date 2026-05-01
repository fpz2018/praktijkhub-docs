# Blok 0B — Samenvatting

**Datum:** 2026-05-01
**Status:** Afgerond — fase 0 voltooid
**Doel:** Backups van beide productie-databases + werkende test-Supabase voor fase 1.

---

## Uitgevoerde stappen

| # | Stap | Resultaat |
|---|---|---|
| 1 | `praktijkhub/` is geen git-repo bevestigd, `pg_dump` 17.9 aanwezig | ✅ |
| 2 | `.gitignore` aangemaakt (`backups/` uitgesloten) + `backups/`-map | ✅ |
| 3 | Project-refs ontvangen (Werkportaal, FysioPro) | ✅ |
| 4 | `pg_dump` Werkportaal-database → `backups/werkportaal_backup_20260501.sql` | ✅ |
| 5 | `pg_dump` FysioPro-database → `backups/fysiopro_backup_20260501.sql` | ✅ |
| 6 | Test-Supabase-project aangemaakt (`orojitrzpapjyrkbebdy`, eu-west-1 Ierland) | ✅ |
| 7 | Werkportaal-backup geïmporteerd in test-Supabase | ✅ |
| 8 | Verificatie data-import: `team=9`, `apps=33`, `categories=15`, `announcements=1` | ✅ |

---

## Backup-bestanden

| Bestand | Grootte | Tabellen |
|---|---|---|
| `backups/werkportaal_backup_20260501.sql` | 32 KB | 8 |
| `backups/fysiopro_backup_20260501.sql` | 4.8 MB | 33 |

**Locatie:** `C:\Users\marc\praktijkhub\backups\` — staat in `.gitignore`, wordt niet gecommit.

**Bewaartermijn:** minimaal 90 dagen (zie Definition of Done fase 1).

---

## Connection-info (geen wachtwoorden — uit 1Password)

| Omgeving | Pooler-host | Region | User-format |
|---|---|---|---|
| Werkportaal-prod | `aws-1-eu-north-1.pooler.supabase.com` | Stockholm | `postgres.nmkxcenoyhevphonfjra` |
| FysioPro-prod | `aws-1-eu-west-2.pooler.supabase.com` | London | `postgres.kunrbjdljqhxpfuwxqdb` |
| Test-Supabase | `aws-0-eu-west-1.pooler.supabase.com` | Ierland | `postgres.orojitrzpapjyrkbebdy` |

**Project-refs:**
- Werkportaal: `nmkxcenoyhevphonfjra`
- FysioPro: `kunrbjdljqhxpfuwxqdb`
- Test: `orojitrzpapjyrkbebdy`

Wachtwoorden staan in 1Password, niet in dit document.

---

## Wat dit oplevert voor fase 1

- ✅ Veilige rollback-positie: backups van beide productie-databases vóór elke wijziging
- ✅ Test-omgeving met Werkportaal-data om migratie te oefenen zonder productie-risico
- ✅ Connection-info gedocumenteerd voor toekomstige scripts/migraties

---

## Status

**Blok 0B afgerond.**
**Fase 0 (voorbereiding) voltooid.**

**Volgende stap:** Fase 1 — Werkportaal auth-migratie. Begint met blok 1A (zie `04_FASE_1_WERKPORTAAL_AUTH_MIGRATIE.md`).

Eerste blok 1A-onderwerpen verwacht:
- Stagiaire-mailbox `stage@fysiopraktijkzeist.nl` aanmaken
- Supabase Auth inschakelen op test-project
- `team`-tabel uitbreiden met `auth_user_id`-kolom in test-omgeving
- Mapping `team.email` ↔ `auth.users.email` opzetten
- `assigned_to`-migratie van `team.id[]` naar `auth_user_id[]` voorbereiden (B-013)
