# Design — English translation + Privacy mode for Rooride Admin

**Date:** 2026-06-09
**File touched:** `index.html` (single-file Firebase admin dashboard, ~1574 lines)
**Author:** brainstormed with user (valpdu44@gmail.com)

## Goal

Two changes to the admin dashboard:

1. **Translate the whole UI from French to English.**
2. **Add a locked "Privacy mode"** that hides personal contact details and sensitive
   data, so the dashboard can be shown to a prospective buyer of the app who has not
   yet paid, without exposing real user PII.

## Context / current state

`index.html` is a self-contained dashboard:

- Pure client-side, talks **directly to Firestore** with a public Firebase config
  embedded in the page.
- On load it downloads **all** raw documents into the browser (`allUsers`, `allTrips`,
  `allPayments`, `allWalletTx`, `allCommissions`, `allNationalities`) via
  `db.collection(...).get()`.
- Auth: Firebase email/password. `checkAdmin()` reads the user doc and currently has a
  fallback that lets **any** authenticated user in (`showApp(user)` is called
  unconditionally).
- Pages: Overview, Users, ID Verification, Trips, Payments, Wallet, Statistics,
  Advanced search.

### Sensitive data currently exposed

| Field | Where |
|---|---|
| Email (`email`) | Users table, Payments table, Verification table, panels, search |
| Phone (`phone_number`) | User panel, Verification panel, search filter |
| Names (`display_name`, `firstName`, `lastName`) | Everywhere |
| Bank details (`bankDetails.BSBNumber`, `.accountNumber`, `.accountHolderName`) | User panel |
| ID photos (`idPhoto`) | Verification table (thumbnail) + Verification panel (full image) |
| Stripe payment IDs (`stripePaymentId`) | Overview, Payments table, panels, search |
| Nationality (`nationality`) | Stats (aggregated only) |

## Decisions (settled during brainstorming)

1. **Protection level: visual masking only.** Implemented in the client.
   Accepted as bypassable (see Known limitations).
2. **Activation: dedicated account, forced ON, no toggle for restricted accounts.**
3. **Account identification: hardcoded email allowlist** in `index.html`. Any logged-in
   email NOT in the list → privacy mode forced on.
4. **Identity fields: pseudonymize** (stable alias for names, masked email/phone) rather
   than blanking them out, to preserve referential continuity for the buyer.
5. Bank details, ID photos, Stripe IDs → **fully hidden** in privacy mode.

## Architecture / implementation approach

**Single-point redaction through the existing display helpers.**

The file already routes user identity through two central helpers used by every table,
panel, and search result:

- `getUserName(ref)` — resolves a user reference to a display name.
- `getUserEmail(ref)` — resolves a user reference to an email.

These become **privacy-aware**. Plus a small set of new masking helpers and a single
global flag. Sensitive sections that are not identity (bank block, ID photo, Stripe ID,
write actions) are gated with `if (!PRIVACY_MODE)`.

Rejected alternatives:
- Scattered `if (privacy)` at each render site — error-prone, easy to miss a field.
- A parallel sanitized data cache — overkill and breaks the join logic that relies on
  raw `usersMap`/`allTrips` for lookups.

### New globals / helpers

```js
const FULL_ACCESS_EMAILS = ['valpdu44@gmail.com']; // owner emails; editable
let PRIVACY_MODE = false;

function pseudoName(u)  // -> "User " + u.id.slice(-4).toUpperCase()
function maskEmail(e)   // -> first char + "•••@•••"  (or "—" if empty)
function maskPhone(p)   // -> "••• ••• ••" + last 2 digits (or "—" if empty)
```

### Activation

In the auth flow (`checkAdmin` / `showApp`):

```js
PRIVACY_MODE = !FULL_ACCESS_EMAILS.includes((user.email || '').toLowerCase());
```

When `PRIVACY_MODE` is true, render a visible **"Privacy mode" pill/banner in the
topbar** so it is obvious the protection is active (also reassures the owner).

### Field treatment when PRIVACY_MODE is ON

| Data | Behavior |
|---|---|
| Name (display/first/last) | Stable alias `User 4F2A` (derived from UID) via `getUserName` / `pseudoName` |
| Email | Masked `j•••@•••` via `getUserEmail` / `maskEmail` |
| Phone | Masked `••• ••• ••12` via `maskPhone` |
| Bank details block (BSB, account #, holder, complete flag) | Section hidden in user panel |
| ID photo (`idPhoto`) | Not rendered. Thumbnail + full image replaced with "Hidden in privacy mode" placeholder; click-to-open disabled |
| Stripe payment IDs | Masked `•••` everywhere |
| Wallet balances / amounts / statuses | **Visible** (business metrics, not personal contact data) |
| Nationalities + aggregated stats | **Visible** |
| Approve / Reject buttons (verification) | **Hidden** — restricted account must not mutate Firestore |

Search continues to work but renders pseudonymized titles and masked subtitles
(automatic, because it reuses `getUserName` / `getUserEmail`). Note: search still
*matches* against the raw fields internally (typing a known email/phone can reveal which
pseudonym it maps to). This is a deliberate, accepted trade-off consistent with the
"masking only / bypassable" decision — not a separate hardening item.

## English translation scope

- `<html lang="fr">` → `lang="en"`.
- Date formatting `toLocaleDateString('fr-FR')` / `toLocaleTimeString('fr-FR')` →
  **`en-AU`** (Australian context: BSB banking, `$`; keeps day/month/year order).
- All static UI strings translated:
  - Login card ("Connecte-toi avec ton compte admin" → "Sign in with your admin
    account", "Mot de passe" → "Password", "Se connecter" → "Sign in", error prefix).
  - Sidebar: Vue d'ensemble→Overview, Utilisateurs→Users, Vérification ID→ID
    Verification, Trajets→Trips, Paiements→Payments, Wallet Transactions (kept),
    Statistiques→Statistics, Outils→Tools, Recherche avancée→Advanced search,
    Déconnexion→Log out.
  - `titles` map (topbar) and all `<h1>/<h3>` headings.
  - Table headers (Passager→Passenger, Montant→Amount, Statut→Status, Utilisateur→User,
    Vérifié→Verified, Inscrit le→Joined, Trajet→Route, Prix/place→Price/seat,
    Places→Seats, etc.).
  - Placeholders, filter buttons (Tous→All, Sans passager→No passengers, Avec
    passagers→With passengers, En attente→Pending, Vérifiés→Verified), empty/loading
    states (Chargement…→Loading…, "Aucun X trouvé"→"No X found").
  - Verification page labels (En attente de review→Pending review, Taux de
    vérification→Verification rate, etc.).
  - Detail-panel labels (Nom→Name, Téléphone→Phone, Wallet & Banque→Wallet & Bank,
    Titulaire→Account holder, Bank complet→Bank complete, etc.).
  - Stats page labels (Utilisateurs total→Total users, Revenue total→Total revenue,
    Drivers actifs→Active drivers, Passagers actifs→Active passengers, Les deux→Both,
    Aucun rôle→No role, Non renseignée→Not provided, etc.).
  - Search page copy and result type labels (Utilisateur→User, Paiement→Payment,
    Trajet→Trip).
  - Action buttons (Approuver→Approve, Rejeter→Reject) and all `confirm()` / `alert()`
    messages.
  - Badge values "Oui/Non" → "Yes/No".
- **Not translated:** values coming from the database (trip statuses, city names,
  transaction types) — they are data, not UI.

## Known limitations / future hardening

Privacy mode is **display-side only and therefore bypassable** by a technical user:
- Browser DevTools console → inspect the `allUsers` array (raw docs incl. phone/email).
- Reuse the embedded Firebase config to query Firestore directly with the buyer's
  credentials.

For protection that cannot be bypassed, a follow-up is required:
- **Firestore Security Rules** restricting the buyer's account from reading sensitive
  fields/collections, or
- Serving the restricted account a pre-redacted dataset.

This is explicitly out of scope for this change and documented here as the recommended
next step before relying on privacy mode against a technical evaluator.

## Verification (manual)

No unit tests possible for a single static HTML file. Manual checklist:

1. **Allowlisted account** (`valpdu44@gmail.com`): log in → entire UI in English,
   no "Privacy mode" pill, all data visible (email, phone, bank, ID photos, Stripe IDs,
   Approve/Reject present).
2. **Non-allowlisted account**: log in → "Privacy mode" pill visible; names show as
   `User XXXX`; email/phone masked; user-panel bank section absent; verification page
   shows counts/status but ID thumbnails/images replaced by placeholder and
   Approve/Reject buttons absent; Stripe IDs show `•••`; search returns pseudonymized
   results.
3. Confirm wallet balances, amounts, nationalities and aggregate stats remain visible in
   both modes.
```
