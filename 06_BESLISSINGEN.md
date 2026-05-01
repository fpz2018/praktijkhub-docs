# Beslissingen-logboek

> Chronologisch logboek van alle architecturele en strategische keuzes voor PraktijkHub.
> **Update dit bestand bij elke nieuwe beslissing** zodat ik (en toekomstige Claude-sessies) altijd weten waarom iets is zoals het is.

---

## 2026-04-30 — Initiële beslissingen

### B-001: Optie 2 gekozen — Werkportaal harmoniseren naar FysioPro-model

**Context:** Drie scenario's overwogen voor PraktijkHub-bouw:
- Optie 1: PraktijkHub als losse derde app (snelste, maar drie aparte logins)
- Optie 2: Werkportaal-auth migreren + PraktijkHub erbij op gedeelde basis
- Optie 3: Alles samenvoegen in één React-codebase (3-6 maanden, te groot)

**Beslissing:** Optie 2.

**Reden:** Marc wil écht "één app"-gevoel met één login over drie tools, maar wil niet alles vanaf nul herbouwen. Werkportaal heeft geen veilige RLS-architectuur en moet sowieso gemoderniseerd worden — dit is een goed moment.

**Consequentie:** ±13-18 weken werk verdeeld over 4 fasen.

---

### B-002: Eén Supabase-project (geen drie gekoppelde projecten)

**Context:** Keuze tussen één Supabase-project met alle tabellen versus drie aparte projecten met FDW-koppelingen.

**Beslissing:** Eén Supabase-project.

**Reden:** Simpeler, sneller, goedkoper. Drie projecten zijn alleen nodig als apps later los aan andere praktijken verkocht worden — voor nu niet relevant. Multi-tenancy kan later toegevoegd via `practice_id`-kolom als SaaS-route doorzet.

**Consequentie:** Werkportaal-data moet in fase 2 gemigreerd worden naar het FysioPro-Supabase-project.

---

### B-003: Werkportaal-data behouden, niet vers starten

**Context:** Bij auth-migratie keuze tussen behouden van bestaande data of vers beginnen.

**Beslissing:** Behouden.

**Reden:** Apps, categorieën, taken, aankondigingen zijn waardevol en duur om opnieuw in te voeren. Migratie is technisch niet moeilijk (UUID-mapping via email).

**Consequentie:** `team`-tabel krijgt `auth_user_id` kolom, mapping via email-match.

---

### B-004: 9 gebruikers, ASAP uitrollen

**Context:** Hoeveel gebruikers en welke tijdsdruk.

**Beslissing:** 9 gebruikers, fase 1 starten zodra mogelijk.

**Reden:** Werkportaal heeft acuut beveiligingsprobleem (open RLS + hardcoded anon-key). Hoe sneller dat opgelost is, hoe beter. Ook: PraktijkHub vervangt Signal — hoe eerder live, hoe meer waarde.

**Consequentie:** Fase 0 (voorbereiding) moet binnen 1 week kunnen starten.

---

### B-005: Stagiaire krijgt rolgebaseerde mailbox `stage@fysiopraktijkzeist.nl`

**Context:** 1 van de 9 gebruikers (stagiaire) heeft geen werkmail. Drie opties: persoonlijke mail (AVG-zwak), rolgebaseerde mailbox, of custom username-flow.

**Beslissing:** Rolgebaseerde mailbox `stage@fysiopraktijkzeist.nl`.

**Reden:** AVG-net, eenmalig setup, makkelijk overdraagbaar bij stagewissel (wachtwoord resetten i.p.v. account migreren).

**Consequentie:** Eenmalige setup van 15 minuten in mail-omgeving voor cutover van fase 1.

---

### B-006: PraktijkHub-stack = React + Vite + Tailwind + shadcn/ui

**Context:** Stackkeuze voor de nieuwe app.

**Beslissing:** Zelfde stack als FysioPro Kennisbank.

**Reden:**
- Codepatronen direct herbruikbaar
- shadcn/ui-componenten al beschikbaar
- Marc werkt al met deze stack — minder cognitieve overhead
- Realtime via Supabase WebSocket goed ondersteund in React

**Consequentie:** Geen Base44-prototype meer (eerder overwogen). Direct Claude Code → React.

---

### B-007: Magic link login (geen wachtwoorden)

**Context:** Login-mechanisme voor de niet-technische gebruikers.

**Beslissing:** Magic link via email.

**Reden:**
- Geen wachtwoord onthouden
- Geen "wachtwoord vergeten"-flow nodig
- Geen wachtwoordmanager nodig
- Veilig genoeg voor zorgcontext mits AVG-vriendelijk emailverkeer

**Consequentie:** Supabase Auth-config: alleen `signInWithOtp` enabled, geen `signInWithPassword` als primaire flow. Email templates in NL.

---

### B-008: Terminologie vastgelegd

**Context:** Kerndoel van de app is heldere zorg-taal i.p.v. techjargon.

**Beslissing:**

| Slack/Teams-term | PraktijkHub-term |
|---|---|
| Workspace | Praktijk |
| Channel | Onderwerp |
| Direct Message | Bericht |
| Thread | Gesprek |
| Project | Project |

**Reden:** Marc bevestigde deze keuze direct. Nederlands, intuïtief, zorg-vriendelijk.

**Consequentie:** Alle UI-tekst, database-tabelnamen waar mogelijk, en documentatie volgen deze termen.

---

### B-009: Snelkoppelingen-paneel in v2 (niet v1)

**Context:** Wens om vanuit PraktijkHub direct naar Werkportaal en FysioPro te kunnen.

**Beslissing:** Wel doen, maar pas in fase 4 (v2). Niet in v1.

**Reden:** v1 moet zich focussen op de kernfunctie (chat). Snelkoppelingen-paneel is eenvoudig (1 dag werk) maar vereist SSO-tests met de andere twee apps — beter na de samenvoeging in fase 2.

**Consequentie:** v1-MVP-scope blijft scherp.

---

### B-010: Claude.ai Project + Claude Code beide gebruiken

**Context:** Waar de bouwwerkzaamheden plaatsvinden.

**Beslissing:**
- **Claude.ai Project "PraktijkHub"** voor strategie, architectuur, copywriting, plan-iteratie
- **Claude Code** voor daadwerkelijk schrijven van code, migraties, deploys

**Reden:** Verschillende sterke punten. Claude.ai (mobiel) = denkwerk onderweg. Claude Code = code-execution op laptop.

**Consequentie:** Documentatie wordt hier in Claude.ai onderhouden en als markdown geëxporteerd naar `/docs/` in de Claude Code-werkmap.

---

## 2026-05-01 — Beslissingen tijdens blok 0A/0B

### B-011: FysioPro Supabase = hoofdproject voor het ecosysteem

**Context:** Twee bestaande Supabase-projecten (Werkportaal en FysioPro). Eén moet hoofdproject worden waar alle data uiteindelijk samenkomt.

**Beslissing:** FysioPro Kennisbank Supabase-project (`kunrbjdljqhxpfuwxqdb`) wordt het hoofdproject.

**Reden:** FysioPro heeft al echte Supabase Auth + RLS-policies. Werkportaal moet zich daarnaar conformeren — andersom zou betekenen dat we een gezond systeem op een onveilig systeem stapelen.

**Consequentie:**
- Alle nieuwe migraties en PraktijkHub-tabellen worden in dit project gemaakt.
- Werkportaal-data migreert in fase 2 hier naartoe.
- Het oude Werkportaal-Supabase-project gaat na fase 2 in archief (30 dagen wachten voor verwijderen).

---

### B-012: Marc de Jong = early adopter voor fase 1-test

**Context:** Wie test als eerste de magic link-flow vóór uitrol naar het 9-koppige team.

**Beslissing:** Marc de Jong (marc@fysiopraktijkzeist.nl) test als eerste.

**Reden:** Eigenaar + technisch onderlegd; kan pijnpunten herkennen en oplossen vóór bredere uitrol.

**Consequentie:** Cutover-volgorde fase 1:
1. Marc test op test-Supabase → fix
2. Marc test op productie → fix
3. 1 collega test (least-technical of meest-betrokken) → fix
4. Volledige uitrol naar overige 7 teamleden

---

### B-013: `assigned_to` migreren van `team.id[]` naar `auth_user_id[]` in fase 1

**Context:** Werkportaal-tabel `apps` heeft kolom `assigned_to TEXT[]` met daarin `team.id`-waarden. Na auth-migratie blijven die werken, maar het levert verwarring op bij PraktijkHub-bouw — daar wordt overal `auth.users.id` gebruikt. Twee soorten user-IDs naast elkaar = brug naar bugs.

**Beslissing:** Eenmalige migratie tijdens fase 1: `apps.assigned_to` herschrijven naar array van `auth_user_id`-waarden via mapping-tabel (`team.id` → `team.auth_user_id`).

**Reden:** Eén bron van waarheid voor user-identiteit door het hele ecosysteem. Voorkomt langetermijnverwarring.

**Consequentie:**
- Sub-stap toevoegen aan fase 1-plan (concrete uitwerking in blok 1A).
- RLS-policies op `apps` kunnen dan rechtstreeks `auth.uid() = ANY(assigned_to)` doen — geen subquery nodig.

---

## Template voor toekomstige beslissingen

```markdown
### B-XXX: [Korte titel beslissing]

**Datum:** YYYY-MM-DD
**Context:** [Wat was de vraag/keuze?]
**Beslissing:** [Wat is gekozen?]
**Reden:** [Waarom deze keuze?]
**Consequentie:** [Wat moet er nu gebeuren of wat is hierdoor uitgesloten?]
```

---

## Open punten — nog te beslissen

- [ ] **B-?:** Productie-domeinen: subdomeinen of paden? (`hub.fysiopraktijkzeist.nl` vs `fysiopraktijkzeist.nl/hub`)
- [ ] **B-?:** Bewaartermijn berichten in PraktijkHub (default oneindig of bv. 2 jaar?)
- [ ] **B-?:** Push-notificaties via Web Push of via email-fallback?
- [ ] **B-?:** Dark mode in v1 of v2?

## Afgesloten in deze ronde
- ✅ B-011: hoofdproject = FysioPro
- ✅ B-012: early adopter fase 1 = Marc
