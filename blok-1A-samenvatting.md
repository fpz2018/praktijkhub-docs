# Blok 1A — Samenvatting

**Datum:** 2026-05-01
**Status:** Afgerond
**Doel:** Supabase Auth voorbereiden in TEST-project + auth-accounts aanmaken voor 8 teamleden.

---

## Uitgevoerde stappen

| # | Stap | Resultaat |
|---|---|---|
| 1 | Email provider in TEST-Supabase: enabled, signup uit, confirm uit | ✅ |
| 2 | URL Configuration: Site URL + Redirect URL op `localhost:3000` voor TEST | ✅ |
| 3 | Magic Link email-template aangepast naar Nederlands | ✅ |
| 4 | 8 auth-accounts aangemaakt in TEST-project | ✅ |
| 5 | Verificatie SQL: 8 rijen, allemaal `email_confirmed_at = true` | ✅ |

---

## Auth-accounts in TEST-project

| Naam | Email |
|---|---|
| Arie | arie@fysiopraktijkzeist.nl |
| Cristina | cristina@fysiopraktijkzeist.nl |
| Gerry | gerry@fysiopraktijkzeist.nl |
| Gillian | gillian@fysiopraktijkzeist.nl |
| Giovanni | giovanni@fysiopraktijkzeist.nl |
| Iris | iris@fysiopraktijkzeist.nl |
| Marc | marc@fysiopraktijkzeist.nl |
| Robin | robin@fysiopraktijkzeist.nl |

**Niet aangemaakt:** Yvonne (stagiaire) — geparkeerd tot blok 1E (zie notities).

---

## Belangrijke notities

- **Email-template:** gebruikt `{{ .ConfirmationURL }}`-variabele voor de inloglink. Onderwerp en body in NL, conform fase 1-plan.
- **Site URL = `localhost:3000`** in TEST-project. Wordt bij productie-cutover vervangen door `werkportaal.fysiopraktijkzeist.nl`.
- **Signup uit, confirm uit:** users worden alleen door admins aangemaakt; geen self-service registratie. Confirm-mail uit omdat we accounts handmatig maken met `email_confirm: true`.
- **Stagiaire (Yvonne):** geparkeerd voor blok 1E. Vereist eerst aanmaak van rolgebaseerde mailbox `stage@fysiopraktijkzeist.nl` (zie B-005).
- **Dummy-wachtwoord `TempPassword2026!`:** alleen voor TEST. NIET gebruiken in productie. In productie loggen gebruikers in via magic link, geen wachtwoord.

---

## Wat dit oplevert voor blok 1B

- 8 `auth.users`-rijen in TEST staan klaar voor mapping met `team`-tabel via `email`
- Magic-link-flow kan getest worden zodra frontend wordt omgebouwd (blok 1C)
- Site URL en redirects staan correct voor lokale ontwikkeling

---

## Status

**Blok 1A afgerond.**

**Volgende stap:** Blok 1B — database-migratie:
- `team`-tabel uitbreiden met `auth_user_id`-kolom
- Mapping `team.email` ↔ `auth.users.email`
- Verificatie: 8 van 9 teamleden gekoppeld (Yvonne nog open)
- `assigned_to`-migratie van `team.id[]` naar `auth_user_id[]` voorbereiden (B-013)
