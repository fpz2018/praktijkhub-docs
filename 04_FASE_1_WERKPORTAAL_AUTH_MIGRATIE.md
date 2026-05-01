# Fase 1 — Werkportaal Auth-Migratie

> Concreet stappenplan om Werkportaal van eigen PIN-auth naar Supabase Auth te migreren, met behoud van alle bestaande data.
> **Doorlooptijd:** 2-3 weken
> **Risico:** middel (mitigatie: backup + test-omgeving + rollback-strategie)

---

## Doel

Aan het einde van deze fase:
- ✅ Alle 9 teamleden loggen in via Supabase Auth (geen PIN meer)
- ✅ Bestaande Werkportaal-data (apps, categorieën, taken, aankondigingen) blijft volledig behouden
- ✅ RLS-policies zijn omgezet van `true` naar echte rule-based policies
- ✅ Anon-key staat netjes in env-variabelen
- ✅ Service worker is geüpdatet (cache versie ophogen)
- ✅ Stagiaire heeft een werkbare login-oplossing

---

## Voorbereiding (voor je begint)

### 1. Backup maken

```bash
# Verbinden met Werkportaal Supabase
pg_dump "postgres://postgres:[PASSWORD]@db.[PROJECT-REF].supabase.co:5432/postgres" \
  --schema=public \
  --no-owner \
  -f werkportaal_backup_$(date +%Y%m%d).sql
```

Check dat de backup ±100KB+ groot is en alle tabellen bevat (`grep "CREATE TABLE" werkportaal_backup_*.sql`).

### 2. Test-Supabase-project aanmaken

- Ga naar supabase.com → New project
- Naam: `werkportaal-migration-test`
- Region: Frankfurt (eu-central-1)
- Database password: bewaar in 1Password
- Importeer backup: `psql "[CONNECTION_STRING]" < werkportaal_backup_*.sql`

### 3. Lokale dev-omgeving

```bash
git clone [werkportaal-repo]
cd werkportaal
# Maak .env.local aan met TEST-Supabase credentials (NIET productie)
echo "SUPABASE_URL=https://[test-project-ref].supabase.co" > .env.local
echo "SUPABASE_ANON_KEY=[test-anon-key]" >> .env.local
```

---

## Beslismoment vooraf — Stagiaire-oplossing

De stagiaire heeft geen `@fysiopraktijkzeist.nl` mail. Drie opties:

| Optie | Hoe | Voor- en nadelen |
|---|---|---|
| **A. Persoonlijke mail** | Stagiaire gebruikt eigen Gmail/Outlook | ✅ Snel, geen extra werk. ❌ Toegang tot praktijkdata via privémail = AVG-zwak |
| **B. Stagiaire-mail aanmaken** | Maak `stage@fysiopraktijkzeist.nl` aan | ✅ AVG-net. ❌ Extra mailbox onderhouden, wisselt per stagiaire |
| **C. Magic link zonder mail** | Custom Supabase Auth flow met username + admin-goedkeuring | ✅ Geen mail nodig. ❌ Veel custom werk, niet standaard |

**Mijn advies: Optie B** — `stage@fysiopraktijkzeist.nl` als rolgebaseerde mailbox. Eenmalig 15 minuten setup, daarna nooit meer omkijken. Bij stagewissel: wachtwoord resetten.

---

## Stap-voor-stap plan

### Week 1 — Database & Auth voorbereiden

#### Stap 1.1 — Supabase Auth inschakelen

In test-Supabase-dashboard:
- Authentication → Providers → Email: enable
- Authentication → Settings → Site URL: `http://localhost:3000` (voor test)
- Email templates: aanpassen naar NL (zie sectie "Email templates" onderaan)

#### Stap 1.2 — `team`-tabel uitbreiden

```sql
-- Voeg auth_user_id toe als nullable FK
alter table team add column auth_user_id uuid references auth.users(id);

-- Index voor snelle lookup
create index idx_team_auth_user_id on team(auth_user_id);

-- Maak email uniek (nodig voor Supabase Auth match)
alter table team add constraint team_email_unique unique(email);
```

#### Stap 1.3 — Auth-accounts aanmaken voor de 9 teamleden

**Optie 1 — Handmatig via Supabase Dashboard** (aanbevolen voor 9 users):
- Authentication → Users → Add user → Send invitation
- Voor elk teamlid: email invullen, "Auto-confirm user" aanvinken (geen mailbevestiging nodig)
- Resultaat: 9 nieuwe rijen in `auth.users` zonder wachtwoord

**Optie 2 — Via SQL (sneller voor 9, schaalbaar):**

```sql
-- Voer dit uit per teamlid (vervang waardes)
select supabase_auth.create_user(
  '{"email":"marc@fysiopraktijkzeist.nl","email_confirm":true}'::jsonb
);
```

#### Stap 1.4 — Mapping leggen tussen `team` en `auth.users`

```sql
-- Voor elk teamlid: koppel team.id aan auth.users.id via email
update team t
set auth_user_id = au.id
from auth.users au
where t.email = au.email;

-- Verifieer: alle 9 moeten gekoppeld zijn
select name, email, auth_user_id from team where auth_user_id is null;
-- Verwacht: 0 rijen
```

**Voor de stagiaire (optie B):**
- Maak `stage@fysiopraktijkzeist.nl` aan als auth-user
- Koppel handmatig: `update team set auth_user_id = '[uuid]' where name = 'Stagiaire'`

---

### Week 2 — Frontend ombouwen

#### Stap 2.1 — Env-variabelen invoeren

In Werkportaal heeft de Supabase-config nu hardcoded de anon-key. Verander dit:

**Voor (huidig in `js/supabase-api.js`):**
```javascript
const SUPABASE_URL = 'https://[ref].supabase.co';
const SUPABASE_ANON_KEY = 'eyJhbG...'; // hardcoded — UNVEILIG
```

**Na:**
```javascript
// In Netlify: Site settings → Environment variables
// SUPABASE_URL en SUPABASE_ANON_KEY toevoegen

// In code (vanilla JS, dus via build-tijd injectie of runtime config):
// Optie A: Netlify Snippet Injection in <head>:
//   <script>
//     window.ENV = {
//       SUPABASE_URL: '%SUPABASE_URL%',
//       SUPABASE_ANON_KEY: '%SUPABASE_ANON_KEY%'
//     };
//   </script>

const SUPABASE_URL = window.ENV?.SUPABASE_URL;
const SUPABASE_ANON_KEY = window.ENV?.SUPABASE_ANON_KEY;
```

#### Stap 2.2 — PIN-flow vervangen door Supabase Auth

In `app.js` (de ~2000 regels):

**Verwijderen:**
- Hele Gate-screen (poortcode-flow)
- Hele User-select-screen (PIN-per-teamlid)
- `localStorage.fpz_gate_verified` logica
- `localStorage.fpz_pin_lockout` logica
- SHA-256 PIN-hashing functies

**Toevoegen:**
- Eén login-scherm: "Voer je werkmail in, je krijgt een magic link"
- `supabase.auth.signInWithOtp({ email })` voor magic link
- `supabase.auth.onAuthStateChange()` listener
- Bij `SIGNED_IN`: haal teamlid-profiel op via `team.auth_user_id = auth.user.id`
- Logout-knop: `supabase.auth.signOut()`

**Pseudocode voor nieuwe loginflow:**

```javascript
// Login-scherm
async function login(email) {
  const { error } = await supabase.auth.signInWithOtp({
    email,
    options: { emailRedirectTo: window.location.origin }
  });
  if (error) toonFout('Inloggen mislukt: ' + error.message);
  else toonInfo('Check je mail voor de inloglink');
}

// Bij app-start: check sessie
const { data: { session } } = await supabase.auth.getSession();
if (session) {
  await laadTeamlid(session.user.id);
  toonHoofdscherm();
} else {
  toonLoginScherm();
}

// Teamlid ophalen
async function laadTeamlid(authUserId) {
  const { data, error } = await supabase
    .from('team')
    .select('*, roles(*)')
    .eq('auth_user_id', authUserId)
    .single();
  if (data) currentUser = data;
}
```

#### Stap 2.3 — Service worker updaten

In `sw.js`: cache-versie ophogen van `v15` → `v16`. Dit forceert oude PWA-installs om te refreshen.

```javascript
const CACHE_NAME = 'werkportaal-v16';
// rest blijft hetzelfde
```

---

### Week 3 — RLS-policies + uitrol

#### Stap 3.1 — Echte RLS-policies schrijven

Voor elke tabel: vervang `using (true)` door echte rules.

**Voorbeeld voor `team`:**

```sql
-- OUD (verwijderen):
drop policy if exists "Allow all" on team;

-- NIEUW:
-- Iedereen mag teamleden lezen (voor selectie in apps, taken, etc.)
create policy "Teamleden zien alle teamleden"
on team for select
to authenticated
using (true);

-- Alleen admins mogen teamleden aanmaken/wijzigen
create policy "Admins beheren team"
on team for all
to authenticated
using (
  exists (
    select 1 from team t
    where t.auth_user_id = auth.uid()
    and t.is_admin = true
  )
);

-- Iedereen mag eigen profiel updaten
create policy "Eigen profiel bewerken"
on team for update
to authenticated
using (auth_user_id = auth.uid())
with check (auth_user_id = auth.uid());
```

**Voorbeeld voor `apps`:**

```sql
drop policy if exists "Allow all" on apps;

-- Iedereen ziet apps die niet specifiek toegewezen zijn, of waar zij in zitten
create policy "Apps zichtbaar voor toegewezen leden"
on apps for select
to authenticated
using (
  assigned_to is null
  or array_length(assigned_to, 1) is null
  or (
    select id::text from team where auth_user_id = auth.uid()
  ) = any(assigned_to)
);

-- Alleen admins beheren apps
create policy "Admins beheren apps"
on apps for all
to authenticated
using (
  exists (
    select 1 from team where auth_user_id = auth.uid() and is_admin = true
  )
);
```

**Doe dit voor ALLE tabellen:** `team`, `apps`, `categories`, `announcements`, `roles`, `taken`, `invite_tokens`, `settings`.

#### Stap 3.2 — Acceptatietest met test-team

- Plan een sessie met 2-3 collega's (incl. minst-technische)
- Stuur ze een magic link
- Laat ze: inloggen, app openen, app toevoegen (als admin), aankondiging lezen, taak afvinken
- Noteer ALLE pijnpunten
- Fix → herhaal

#### Stap 3.3 — Productie-cutover

**Vrijdagavond 17:00 uitvoeren** (laagste activiteit):

1. Aankondiging in Signal: "Werkportaal vannacht offline voor update, vanaf maandag inloggen via magic link"
2. Backup productie (zie voorbereiding)
3. Voer alle SQL-migraties uit op productie-Supabase
4. Update Netlify env-vars (al gebeurd voor test, nu voor prod)
5. Deploy nieuwe Werkportaal-versie naar Netlify
6. Test zelf op telefoon: inloggen → app openen → uitloggen
7. Stuur magic link naar 1 collega als laatste check
8. Maandagochtend 08:00: stuur magic links naar iedereen + handleiding

#### Stap 3.4 — Handleiding voor team

Korte 1-pager met screenshots:

```
Hoi team,

Werkportaal heeft een upgrade gehad. Geen PIN-codes meer!

Inloggen werkt zo:
1. Ga naar werkportaal.fysiopraktijkzeist.nl
2. Vul je werkmail in: jouwnaam@fysiopraktijkzeist.nl
3. Klik op "Inloglink versturen"
4. Open je mail, klik op de link
5. Je bent ingelogd

Werkt op telefoon én laptop. De link is 1 uur geldig.

Vragen? App Marc.
```

---

## Email templates (Supabase Auth in NL)

In Supabase Dashboard → Authentication → Email Templates:

**Magic Link template:**

```html
<h2>Inloggen Werkportaal</h2>
<p>Hoi,</p>
<p>Klik op onderstaande knop om in te loggen op het Werkportaal van Fysiopraktijk Zeist.</p>
<p><a href="{{ .ConfirmationURL }}" style="background:#2CA8DF;color:white;padding:12px 24px;text-decoration:none;border-radius:6px;">Inloggen</a></p>
<p>Deze link is 1 uur geldig. Heb je deze mail niet aangevraagd? Negeer hem dan.</p>
<p>Vriendelijke groet,<br>Fysiopraktijk Zeist</p>
```

---

## Rollback-strategie

Als er iets misgaat tijdens de cutover:

1. **Frontend rollback:** Netlify → Deploys → klik vorige deploy → "Publish deploy"
2. **Database rollback:** voer omgekeerde SQL uit (drop nieuwe policies, herstel `using (true)` policies)
3. **Auth rollback:** geen ingrijpende rollback nodig — de PIN-flow staat nog in oude code, terugzetten = oude code redeployen

**Kritiek:** test de rollback in de test-omgeving VOORDAT je naar productie gaat.

---

## Definition of Done — Fase 1

- [ ] Alle 9 teamleden kunnen inloggen via magic link
- [ ] Geen PIN-flow meer in de code
- [ ] Anon-key staat in env-vars, niet in code
- [ ] Alle tabellen hebben echte RLS-policies (geen `true` policies)
- [ ] Audit met externe blik: niet-ingelogde Postman-request krijgt geen data terug
- [ ] Stagiaire heeft werkende inlog
- [ ] Service worker geüpdatet
- [ ] README in repo geüpdatet met nieuwe auth-flow
- [ ] Backup van situatie voor migratie veilig opgeslagen (minimaal 90 dagen bewaren)

**Pas als al deze vinkjes staan → fase 2.**
