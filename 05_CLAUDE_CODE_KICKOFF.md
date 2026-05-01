# Claude Code Kick-off Prompt

> **Hoe te gebruiken:**
> 1. Open Claude Code in de terminal
> 2. Ga naar je dev-map: `cd ~/dev/`
> 3. Maak een nieuwe map: `mkdir praktijkhub-ecosystem && cd praktijkhub-ecosystem`
> 4. Start Claude Code: `claude`
> 5. Plak onderstaande prompt in het eerste bericht

---

## Prompt om te plakken in Claude Code

```
Hi Claude,

Ik ga een grootschalig migratie- en bouwproject doen voor mijn fysiotherapie-
praktijk. Lees eerst zorgvuldig deze context voor je iets doet.

## Context

Ik ben Marc de Jong, fysiotherapeut, eigenaar Fysiopraktijk Zeist. Ik bouw 
een ecosysteem van 3 webapps op één Supabase-project:

1. **Werkportaal** (bestaand) — vanilla HTML/JS, eigen PIN-auth → MOET 
   gemigreerd naar Supabase Auth
   - GitHub: [vul jouw repo-URL in]
   - Productie: werkportaal.fysiopraktijkzeist.nl
   
2. **FysioPro Kennisbank** (bestaand) — React + Vite + Supabase Auth + RLS, 
   blijft als referentie-architectuur
   - GitHub: https://github.com/fpz2018/fysiopro-kennisbank
   - Productie: [vul URL in]
   
3. **PraktijkHub** (NIEUW) — Slack-achtige team-samenwerking in NL
   - Stack: React 18 + Vite + Tailwind + shadcn/ui + Supabase
   - Gebruikt zelfde Supabase-project en Supabase Auth als FysioPro

## Gebruikers

9 teamleden — 8 met @fysiopraktijkzeist.nl mail, 1 stagiaire met 
stage@fysiopraktijkzeist.nl (rolgebaseerde mailbox).

## Taal & UX

- Alle UI in Nederlands
- Mobile-first
- Maximaal 5 hoofdfuncties zichtbaar — geen verstopte menu's
- Heldere zorg-taal: Onderwerpen, Berichten, Gesprekken, Praktijk, Projecten
  (NIET: channels, threads, workspace, DMs, projects)
- Voor niet-technici: intuïtief boven feature-rijk

## Wat ik wil dat je nu doet

1. **STOP** voor je code schrijft. Bevestig dat je deze context begrepen hebt.

2. Stel mij eerst voor: welke fase pakken we als eerste?
   - Fase 0: Voorbereiding & backup
   - Fase 1: Werkportaal-auth migratie naar Supabase Auth (2-3 weken)
   - Fase 2: Supabase-projecten samenvoegen (1 week)
   - Fase 3: PraktijkHub v1 bouwen (6-8 weken)
   - Fase 4: PraktijkHub v2 (4-6 weken)

3. Vraag me om de relevante repo's te clonen voor de fase die we kiezen, 
   voordat je gaat lezen of bouwen.

4. Stel altijd 1-3 verduidelijkende vragen voor je een grote stap zet. 
   Ik werk in dialoog, niet via long-form briefings.

## Mijn werkstijl

- Korte zinnen, stelling eerst
- Bullet points en tabellen boven proza
- Geen academisch vertoon, geen managementtaal, geen metaforen
- Selector, geen specificeerder — bied opties zodat ik kan kiezen
- "Hoi" als opening, "Vriendelijke groet" als afsluiting

## Veiligheid — niet-onderhandelbaar

- Geen RLS-policies met `using (true)`
- Geen hardcoded API-keys, alles via .env
- Service-role-key nooit in client-code, alleen in Edge Functions
- Patiëntdata komt NIET in PraktijkHub-berichten

## Documentatie

Ik heb voor dit project een Claude.ai Project genaamd "PraktijkHub" met 
6 documenten als project-knowledge. Het meest relevante voor jou nu is 
04_FASE_1_WERKPORTAAL_AUTH_MIGRATIE.md (als we met fase 1 starten).

Begin nu met stap 1: bevestigen dat je dit begrepen hebt + vraag welke 
fase we starten.
```

---

## Tips voor het werken in Claude Code

### Per fase een aparte sub-map

```
~/dev/praktijkhub-ecosystem/
├── werkportaal/          (bestaande repo, gecloned)
├── fysiopro-kennisbank/  (bestaande repo, gecloned)
├── praktijkhub/          (nieuwe repo, na fase 3)
├── docs/                 (kopie van Claude.ai Project-bestanden)
└── migrations/           (SQL-scripts voor de migratie-stappen)
```

### Belangrijke commands om Claude Code te geven

**Bij start van elke sessie:**
```
Lees /docs/01_PROJECT_INSTRUCTIES.md en /docs/03_ARCHITECTUUR.md voor context.
We werken vandaag aan: [fase X, stap Y].
```

**Voor je commit:**
```
Voor je commit: laat me eerst de diff zien en wacht op mijn akkoord.
```

**Voor productie-deploy:**
```
NIET deployen naar productie zonder mijn expliciete "go". Test in 
test-omgeving eerst.
```

### Backup-discipline

Voor ELKE fase:
```bash
# Maak feature-branch
git checkout -b fase-1-auth-migratie

# Werk in branch
# ... commits ...

# Push branch (geen merge naar main!)
git push origin fase-1-auth-migratie
```

Pas mergen naar `main` als de fase echt klaar is en getest.

### Wanneer terug naar Claude.ai (deze chat)

Voor Claude Code: bouwen, debuggen, migreren, testen.
Voor Claude.ai (Project): strategie, architectuur-beslissingen, herzien plannen, copywriting voor handleidingen.

---

## Eerste sessie — voorbeeldverloop

**Jij:** plakt bovenstaande prompt
**Claude Code:** bevestigt context + vraagt welke fase
**Jij:** "Fase 1 — Werkportaal auth-migratie"
**Claude Code:** "Clone eerst de Werkportaal-repo zodat ik de huidige code kan zien. Welke URL?"
**Jij:** geeft URL
**Claude Code:** kloont, leest code, vat samen wat hij ziet, vraagt of het klopt
**Jij:** bevestigt of corrigeert
**Claude Code:** stelt voor: "Laten we beginnen met stap 1.1: Supabase Auth inschakelen in een test-project. Ik kan de SQL en config voor je voorbereiden — akkoord?"
**Jij:** "ja"
**Claude Code:** levert SQL + instructies voor Supabase Dashboard

Zo bouw je stap voor stap, in dialoog, zonder dat Claude Code 6 uur achter elkaar dingen doet die je niet bedoelde.
