# English Translation + Privacy Mode Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Translate the Rooride admin dashboard UI to English and add a locked, account-based "Privacy mode" that pseudonymizes identities and hides bank details, ID photos, Stripe IDs, and write actions from non-allowlisted accounts.

**Architecture:** All changes are in the single file `index.html`. Privacy is enforced at render time through one global flag (`PRIVACY_MODE`), the two existing identity helpers (`getUserName`/`getUserEmail` made privacy-aware), a few masking helpers, and inline `PRIVACY_MODE` ternaries / section gates at each direct field-access site. Translation is a set of string replacements plus the `lang` attribute and date locale.

**Tech Stack:** Plain HTML/CSS/vanilla JS, Firebase (auth + Firestore) compat SDK loaded from CDN. No build step, no test framework.

---

## Testing note (read first)

This is a single static HTML file wired to live Firebase; there is no automated test
harness. Verification for every task is **manual in the browser DevTools console**, using
the owner account (which loads real data) and toggling `PRIVACY_MODE` at runtime to
exercise the restricted view without needing a second Firebase account.

**Setup once before testing:**
1. Serve the page from an authorized origin. Firebase Auth only allows the domains in
   *Firebase console → Authentication → Settings → Authorized domains*. The deployed
   GitHub Pages domain already works. For local testing, either test on the deployed
   site, or add `localhost` to Authorized domains and run `python3 -m http.server` in the
   repo root, then open `http://localhost:8000`.
2. Log in with the owner account (`valpdu44@gmail.com`). Wait for tables to populate.

**Runtime privacy check pattern** (paste in DevTools console after data loads):
```js
PRIVACY_MODE = true;
switchPage('users');            // re-renders Users table masked
openUserPanel(allUsers[0].id);  // re-renders a user panel masked
// ...inspect, then restore:
PRIVACY_MODE = false; switchPage('users');
```

Commit after each task.

---

## File structure

- `index.html` — the only file modified. All tasks edit it.
- `docs/superpowers/specs/2026-06-09-english-translation-privacy-mode-design.md` — the
  approved spec (reference, not modified).

Execution order matters: **privacy helpers (Tasks 1–2) → privacy gating (Tasks 3–5) →
translation (Tasks 6–7) → final verification (Task 8).** Translation runs last so its
string edits never invalidate the gating edits. Where a later task's `old_string` would
otherwise be ambiguous, the exact surrounding text is given.

---

## Task 1: Privacy globals, masking helpers, activation, topbar pill

**Files:**
- Modify: `index.html` (script top ~line 633, `showApp` ~673, topbar HTML ~429)

- [ ] **Step 1: Add globals + masking helpers**

In the `// ── Cache ──` block, after `let natMap = {};` (line ~642), add:

```js
// ── Privacy mode ──
// Emails listed here get FULL access. Any other logged-in email is forced into
// privacy mode (sensitive data hidden). NOTE: this is display-side only and is
// bypassable by a technical user via DevTools / direct Firestore access — see the
// spec's "Known limitations".
const FULL_ACCESS_EMAILS = ['valpdu44@gmail.com'];
let PRIVACY_MODE = false;

function pseudoName(uid) {
  return 'User ' + (uid ? String(uid).slice(-4).toUpperCase() : '????');
}
function maskEmail(email) {
  if (!email) return '—';
  return email.charAt(0) + '•••@•••';
}
function maskPhone(phone) {
  if (!phone) return '—';
  const digits = String(phone).replace(/\D/g, '');
  return '••• ••• ••' + (digits.slice(-2) || '');
}
function maskId(val) {
  return val ? '•••' : '—';
}
```

- [ ] **Step 2: Set `PRIVACY_MODE` and toggle the pill in `showApp`**

Replace the body of `showApp` (lines ~673-678):

```js
function showApp(user) {
  PRIVACY_MODE = !FULL_ACCESS_EMAILS.includes((user.email || '').toLowerCase());
  document.getElementById('loginScreen').style.display = 'none';
  document.getElementById('appShell').style.display = 'block';
  document.getElementById('sidebarUserEmail').textContent = user.email;
  document.getElementById('privacyPill').style.display = PRIVACY_MODE ? 'inline-flex' : 'none';
  loadAllData();
}
```

- [ ] **Step 3: Add the privacy pill to the topbar**

In the topbar block, change the `<h1>` line (line ~429) from:

```html
      <h1 id="topbarTitle">Vue d'ensemble</h1>
```
to:
```html
      <h1 id="topbarTitle">Vue d'ensemble</h1>
      <span id="privacyPill" style="display:none; align-items:center; gap:6px; margin-left:14px; background:var(--yellow-bg); color:var(--yellow); border:1px solid var(--yellow); border-radius:100px; padding:4px 12px; font-size:12px; font-weight:600;">🔒 Privacy mode</span>
```

(The `topbar` is a flex row, so the pill sits next to the title. Translation of the
default `Vue d'ensemble` happens in Task 6.)

- [ ] **Step 4: Verify**

Reload, log in as owner. Expected: no privacy pill; in console `PRIVACY_MODE` is `false`.
Then run `PRIVACY_MODE = true; document.getElementById('privacyPill').style.display='inline-flex';`
→ yellow "🔒 Privacy mode" pill appears next to the title. Also test the helpers:
`pseudoName('abcd1234EF')` → `"User 34EF"`; `maskEmail('john@gmail.com')` → `"j•••@•••"`;
`maskPhone('+33 6 12 34 56 78')` → `"••• ••• ••78"`. Restore: `PRIVACY_MODE=false`.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Add privacy-mode globals, masking helpers, activation and topbar pill"
```

---

## Task 2: Make identity helpers privacy-aware

**Files:**
- Modify: `index.html` `getUserName` (~792-798), `getUserEmail` (~800-804)

- [ ] **Step 1: Update `getUserName`**

Replace (lines ~792-798):

```js
function getUserName(ref) {
  if (!ref) return '—';
  const uid = typeof ref === 'string' ? ref : ref.id || ref.path?.split('/').pop();
  if (PRIVACY_MODE) return pseudoName(uid);
  const u = usersMap[uid];
  if (!u) return uid?.substring(0, 8) + '...';
  return u.display_name || `${u.firstName || ''} ${u.lastName || ''}`.trim() || u.email || uid;
}
```

- [ ] **Step 2: Update `getUserEmail`**

Replace (lines ~800-804):

```js
function getUserEmail(ref) {
  if (!ref) return '';
  const uid = typeof ref === 'string' ? ref : ref.id || ref.path?.split('/').pop();
  const email = usersMap[uid]?.email || '';
  if (PRIVACY_MODE) return email ? maskEmail(email) : '';
  return email;
}
```

- [ ] **Step 3: Verify**

After data loads: `PRIVACY_MODE=true; getUserName(allUsers[0].id)` → `"User XXXX"`;
`getUserEmail(allUsers[0].id)` → masked or `""`. `PRIVACY_MODE=false; getUserName(allUsers[0].id)`
→ real name. Restore `PRIVACY_MODE=false`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Make getUserName/getUserEmail respect privacy mode"
```

---

## Task 3: Privacy gating — Users table + user panel

**Files:**
- Modify: `index.html` `renderUsersTable` (~826-837), `openUserPanel` (~954-1008)

- [ ] **Step 1: Gate name + email cells in `renderUsersTable`**

Change the name cell (line ~828) from:
```html
      <td><strong>${u.display_name || `${u.firstName || ''} ${u.lastName || ''}`.trim() || '—'}</strong></td>
```
to:
```html
      <td><strong>${PRIVACY_MODE ? pseudoName(u.id) : (u.display_name || `${u.firstName || ''} ${u.lastName || ''}`.trim() || '—')}</strong></td>
```

Change the email cell (line ~829) from:
```html
      <td class="mono">${u.email || '—'}</td>
```
to:
```html
      <td class="mono">${PRIVACY_MODE ? maskEmail(u.email) : (u.email || '—')}</td>
```

(The Wallet, Verified, Bank-completeness ✓/✗, and Joined columns stay — they are not PII.)

- [ ] **Step 2: Gate name/email/phone in `openUserPanel` Informations section**

Change the three rows (lines ~957-959) from:
```html
      <div class="detail-row"><span class="dl">Nom</span><span class="dv">${u.display_name || `${u.firstName || ''} ${u.lastName || ''}`.trim() || '—'}</span></div>
      <div class="detail-row"><span class="dl">Email</span><span class="dv mono">${u.email || '—'}</span></div>
      <div class="detail-row"><span class="dl">Téléphone</span><span class="dv mono">${u.phone_number || '—'}</span></div>
```
to:
```html
      <div class="detail-row"><span class="dl">Nom</span><span class="dv">${PRIVACY_MODE ? pseudoName(u.id) : (u.display_name || `${u.firstName || ''} ${u.lastName || ''}`.trim() || '—')}</span></div>
      <div class="detail-row"><span class="dl">Email</span><span class="dv mono">${PRIVACY_MODE ? maskEmail(u.email) : (u.email || '—')}</span></div>
      <div class="detail-row"><span class="dl">Téléphone</span><span class="dv mono">${PRIVACY_MODE ? maskPhone(u.phone_number) : (u.phone_number || '—')}</span></div>
```

- [ ] **Step 3: Hide the bank section in `openUserPanel`**

The bank block is the `<div class="detail-section">` containing `<h4>💰 Wallet & Banque</h4>`
(lines ~966-973). It mixes wallet balance (keep) with bank details (hide). Replace the whole
block from `<div class="detail-section">` (line ~966) through its closing `</div>` (line ~973)
with a version that keeps wallet balance and gates only the bank rows:

```html
    <div class="detail-section">
      <h4>💰 Wallet & Banque</h4>
      <div class="detail-row"><span class="dl">Wallet Balance</span><span class="dv wallet-amount wallet-positive" style="font-size:18px;">${fmtMoney(u.walletBalance || 0)}</span></div>
      ${PRIVACY_MODE ? `<div class="detail-row"><span class="dl">Coordonnées bancaires</span><span class="dv" style="color:var(--gray-400);">Masquées (mode privacy)</span></div>` : `
      <div class="detail-row"><span class="dl">BSB</span><span class="dv mono">${u.bankDetails?.BSBNumber || '—'}</span></div>
      <div class="detail-row"><span class="dl">Account</span><span class="dv mono">${u.bankDetails?.accountNumber || '—'}</span></div>
      <div class="detail-row"><span class="dl">Titulaire</span><span class="dv">${u.bankDetails?.accountHolderName || '—'}</span></div>
      <div class="detail-row"><span class="dl">Bank complet</span><span class="dv">${u.bankDetails?.isBankDetailsComplete ? '✅' : '❌'}</span></div>`}
    </div>
```

- [ ] **Step 4: Mask Stripe IDs in the user panel payments sub-list**

In the "Paiements comme passager" list item (line ~992), change:
```html
            <div class="dli-sub">${fmtDate(p.paymentDate)} · Stripe: ${p.stripePaymentId || '—'}</div>
```
to:
```html
            <div class="dli-sub">${fmtDate(p.paymentDate)} · Stripe: ${PRIVACY_MODE ? maskId(p.stripePaymentId) : (p.stripePaymentId || '—')}</div>
```

- [ ] **Step 5: Verify**

After data loads: `PRIVACY_MODE=true; switchPage('users')` → names `User XXXX`, emails masked.
`openUserPanel(allUsers[0].id)` → Name/Email/Phone masked, bank rows replaced by
"Masquées (mode privacy)", wallet balance still shown, Stripe in payment list `•••`.
Restore `PRIVACY_MODE=false; switchPage('users')` → everything real again.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Gate users table and user panel behind privacy mode"
```

---

## Task 4: Privacy gating — Verification table + panel

**Files:**
- Modify: `index.html` `renderVerifTable` (~1200-1228), `openVerificationPanel` (~1231-1283)

- [ ] **Step 1: Gate name/email/thumbnail/action in `renderVerifTable`**

Replace the `const name = ...` line (line ~1201) and the `thumbHtml`/`actionHtml` builders
(lines ~1209-1217) so they respect privacy, and update the row cells (lines ~1221-1226).

Change `const name = ...` (line ~1201) from:
```js
    const name = u.display_name || `${u.firstName || ''} ${u.lastName || ''}`.trim() || '—';
```
to:
```js
    const name = PRIVACY_MODE ? pseudoName(u.id) : (u.display_name || `${u.firstName || ''} ${u.lastName || ''}`.trim() || '—');
```

Change `thumbHtml` (lines ~1209-1211) from:
```js
    const thumbHtml = hasDoc
      ? `<img src="${u.idPhoto}" alt="ID" style="height:36px;width:54px;object-fit:cover;border-radius:6px;border:1px solid var(--gray-200);cursor:pointer;" onerror="this.outerHTML='<span style=\\'color:var(--red);font-size:12px\\'>Erreur image</span>'">`
      : '<span style="color:var(--gray-300);font-size:12px;">—</span>';
```
to:
```js
    const thumbHtml = PRIVACY_MODE
      ? '<span style="color:var(--gray-400);font-size:12px;">🔒</span>'
      : hasDoc
        ? `<img src="${u.idPhoto}" alt="ID" style="height:36px;width:54px;object-fit:cover;border-radius:6px;border:1px solid var(--gray-200);cursor:pointer;" onerror="this.outerHTML='<span style=\\'color:var(--red);font-size:12px\\'>Erreur image</span>'">`
        : '<span style="color:var(--gray-300);font-size:12px;">—</span>';
```

Change `actionHtml` (lines ~1213-1217) from:
```js
    const actionHtml = !u.isVerified && hasDoc
      ? `<button class="filter-btn" style="background:var(--green-bg);border-color:var(--green);color:var(--green);" onclick="event.stopPropagation();approveUser('${u.id}')">Approuver ✓</button>`
      : u.isVerified
        ? '<span style="color:var(--green);font-size:12px;font-weight:600;">✓ Approuvé</span>'
        : '—';
```
to:
```js
    const actionHtml = PRIVACY_MODE
      ? '—'
      : !u.isVerified && hasDoc
        ? `<button class="filter-btn" style="background:var(--green-bg);border-color:var(--green);color:var(--green);" onclick="event.stopPropagation();approveUser('${u.id}')">Approuver ✓</button>`
        : u.isVerified
          ? '<span style="color:var(--green);font-size:12px;font-weight:600;">✓ Approuvé</span>'
          : '—';
```

Change the email cell (line ~1222) from:
```html
        <td class="mono" style="font-size:12px;">${u.email || '—'}</td>
```
to:
```html
        <td class="mono" style="font-size:12px;">${PRIVACY_MODE ? maskEmail(u.email) : (u.email || '—')}</td>
```

- [ ] **Step 2: Gate `openVerificationPanel` — name, photo, actions, info rows**

Change `const name = ...` (line ~1234) from:
```js
  const name = u.display_name || `${u.firstName || ''} ${u.lastName || ''}`.trim() || u.email || uid;
```
to:
```js
  const name = PRIVACY_MODE ? pseudoName(uid) : (u.display_name || `${u.firstName || ''} ${u.lastName || ''}`.trim() || u.email || uid);
```

Replace the `photoSection` assignment (lines ~1242-1255) so privacy mode shows a placeholder
instead of the image:
```js
  const photoSection = PRIVACY_MODE ? `
    <div class="detail-section">
      <h4>📄 Pièce d'identité</h4>
      <div style="background:var(--gray-100);border-radius:10px;padding:32px;text-align:center;color:var(--gray-400);font-size:13px;">
        🔒 Masqué (mode privacy)
      </div>
    </div>` : u.idPhoto ? `
    <div class="detail-section">
      <h4>📄 Pièce d'identité</h4>
      <div class="verify-photo-wrap" onclick="window.open('${u.idPhoto}', '_blank')">
        <img src="${u.idPhoto}" alt="Pièce d'identité" onerror="this.parentElement.innerHTML='<p style=\\'padding:24px;color:var(--red);font-size:13px;\\'>Impossible de charger l\\'image</p>'">
      </div>
      <p style="font-size:11px;color:var(--gray-400);text-align:center;margin-top:-8px;margin-bottom:16px;">Cliquer pour ouvrir en plein écran</p>
    </div>` : `
    <div class="detail-section">
      <h4>📄 Pièce d'identité</h4>
      <div style="background:var(--gray-100);border-radius:10px;padding:32px;text-align:center;color:var(--gray-400);font-size:13px;">
        Aucun document soumis
      </div>
    </div>`;
```

Gate the actions section: change the `actionsSection` condition (line ~1257) from:
```js
  const actionsSection = !u.isVerified && u.idPhoto ? `
```
to:
```js
  const actionsSection = !PRIVACY_MODE && !u.isVerified && u.idPhoto ? `
```

Gate the email + phone info rows (lines ~1274-1275) from:
```html
      <div class="detail-row"><span class="dl">Email</span><span class="dv mono">${u.email || '—'}</span></div>
      <div class="detail-row"><span class="dl">Téléphone</span><span class="dv mono">${u.phone_number || '—'}</span></div>
```
to:
```html
      <div class="detail-row"><span class="dl">Email</span><span class="dv mono">${PRIVACY_MODE ? maskEmail(u.email) : (u.email || '—')}</span></div>
      <div class="detail-row"><span class="dl">Téléphone</span><span class="dv mono">${PRIVACY_MODE ? maskPhone(u.phone_number) : (u.phone_number || '—')}</span></div>
```

- [ ] **Step 3: Verify**

`PRIVACY_MODE=true; switchPage('verification')` → table shows pseudonyms, masked emails,
🔒 instead of ID thumbnails, no Approve buttons. `openVerificationPanel(allUsers.find(u=>u.idPhoto).id)`
→ photo section shows "🔒 Masqué (mode privacy)", no Approve/Reject actions, email/phone masked.
Restore `PRIVACY_MODE=false; switchPage('verification')`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Gate verification table and panel behind privacy mode"
```

---

## Task 5: Privacy gating — Stripe IDs in payments, overview, trip & payment panels

**Files:**
- Modify: `index.html` `renderPaymentsTable` (~858-867), `renderOverviewPayments` (~873-881), `openTripPanel` (~1011-1086), `openPaymentPanel` (~1087-1137)

- [ ] **Step 1: Mask Stripe ID in `renderPaymentsTable`**

Change (line ~864):
```html
      <td class="mono" style="font-size:11px;">${p.stripePaymentId || '—'}</td>
```
to:
```html
      <td class="mono" style="font-size:11px;">${PRIVACY_MODE ? maskId(p.stripePaymentId) : (p.stripePaymentId || '—')}</td>
```

(Passenger name/email already go through `getUserName`/`getUserEmail` → privacy-aware.)

- [ ] **Step 2: Mask Stripe ID in `renderOverviewPayments`**

Change (line ~878):
```html
      <td class="mono" style="font-size:11px;">${p.stripePaymentId || '—'}</td>
```
to:
```html
      <td class="mono" style="font-size:11px;">${PRIVACY_MODE ? maskId(p.stripePaymentId) : (p.stripePaymentId || '—')}</td>
```

- [ ] **Step 3: Mask Stripe IDs in `openTripPanel` and `openPaymentPanel`**

There are three exact sites (driver/passenger names already use `getUserName` → privacy-aware).

`openTripPanel` linked-payments sub-list (line ~1068), change:
```html
            <div class="dli-sub">${getUserName(p.passengerId)} · Stripe: ${p.stripePaymentId || '—'}</div>
```
to:
```html
            <div class="dli-sub">${getUserName(p.passengerId)} · Stripe: ${PRIVACY_MODE ? maskId(p.stripePaymentId) : (p.stripePaymentId || '—')}</div>
```

`openPaymentPanel` Stripe ID row (line ~1101), change:
```html
      <div class="detail-row"><span class="dl">Stripe ID</span><span class="dv mono" style="font-size:11px;">${p.stripePaymentId || '—'}</span></div>
```
to:
```html
      <div class="detail-row"><span class="dl">Stripe ID</span><span class="dv mono" style="font-size:11px;">${PRIVACY_MODE ? maskId(p.stripePaymentId) : (p.stripePaymentId || '—')}</span></div>
```

`openPaymentPanel` panel title (line ~1135) leaks the Stripe ID into the title — change:
```js
  openPanel('Paiement ' + (p.stripePaymentId || p.id.substring(0, 8)), html);
```
to:
```js
  openPanel('Paiement ' + (PRIVACY_MODE ? p.id.substring(0, 8) : (p.stripePaymentId || p.id.substring(0, 8))), html);
```

After editing, run `git grep -n "stripePaymentId" index.html` and confirm every interpolation
that renders to the UI is wrapped in a `PRIVACY_MODE` ternary. (The `'Paiement '` prefix is
translated in Task 7.)

- [ ] **Step 4: Verify**

`PRIVACY_MODE=true; switchPage('payments')` → Stripe column shows `•••`. Open a payment
(`openPaymentPanel(allPayments[0].id)`) and a trip with payments → Stripe IDs masked, party
names pseudonymized. `git grep -n "stripePaymentId" index.html` shows every interpolation
wrapped in a `PRIVACY_MODE` ternary. Restore `PRIVACY_MODE=false`.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Mask Stripe IDs across payment, overview, trip and payment views in privacy mode"
```

---

## Task 6: English translation — static HTML

**Files:**
- Modify: `index.html` `<html>` tag (line 2), login (~349-359), sidebar (~365-423), topbar default title (~429), all page headers/table headers/placeholders/filter buttons/loading rows (~439-604), side panel default title (~614)

- [ ] **Step 1: Set language attribute**

Line 2: `<html lang="fr">` → `<html lang="en">`.

- [ ] **Step 2: Apply the static-HTML string map**

Replace each French string on the left with the English on the right (exact text, keep all
surrounding markup/attributes/IDs identical). All are in the HTML body (lines ~349-617):

| French | English |
|---|---|
| `Connecte-toi avec ton compte admin` | `Sign in with your admin account` |
| `Mot de passe` (password placeholder) | `Password` |
| `Se connecter` | `Sign in` |
| `Navigation` (nav-section) | `Navigation` |
| `Vue d'ensemble` (nav-item, line ~376) | `Overview` |
| `Utilisateurs` (nav-item) | `Users` |
| `Vérification ID` (nav-item) | `ID Verification` |
| `Trajets` (nav-item) | `Trips` |
| `Paiements` (nav-item) | `Payments` |
| `Wallet Transactions` (nav-item) | `Wallet Transactions` |
| `Statistiques` (nav-item) | `Statistics` |
| `Outils` (nav-section) | `Tools` |
| `Recherche avancée` (nav-item) | `Advanced search` |
| `Déconnexion` | `Log out` |
| `Vue d'ensemble` (topbar `<h1>`, line ~429) | `Overview` |
| `Recherche rapide : email, Stripe ID, nom...` | `Quick search: email, Stripe ID, name...` |
| `Utilisateurs` (stat-label) | `Users` |
| `Trajets` (stat-label) | `Trips` |
| `Paiements` (stat-label) | `Payments` |
| `Total wallet en circulation` | `Total wallet in circulation` |
| `Somme des balances positives` | `Sum of positive balances` |
| `Derniers paiements` | `Latest payments` |
| `Passager` (overview th) | `Passenger` |
| `Montant` (overview th) | `Amount` |
| `Statut` (overview th) | `Status` |
| `Chargement...` (all loading rows) | `Loading...` |
| `Tous les utilisateurs` | `All users` |
| `Filtrer par email ou nom...` | `Filter by email or name...` |
| `Utilisateur` (users th) | `User` |
| `Vérifié` (users th) | `Verified` |
| `Bank` (users th) | `Bank` |
| `Inscrit le` (users th) | `Joined` |
| `Tous les trajets` | `All trips` |
| `Tous` (trips filter btn) | `All` |
| `Sans passager` | `No passengers` |
| `Avec passagers` | `With passengers` |
| `Trajet` (trips th) | `Route` |
| `Driver` (trips th) | `Driver` |
| `Prix/place` | `Price/seat` |
| `Places` (trips th) | `Seats` |
| `Tous les paiements` | `All payments` |
| `Rechercher Stripe ID...` | `Search Stripe ID...` |
| `Promo` (payments th) | `Promo` |
| `Utilisateur` (wallet th) | `User` |
| `Type` (wallet th) | `Type` |
| `Description` (wallet th) | `Description` |
| `En attente de review` (verif stat-label) | `Pending review` |
| `Pièce soumise, non vérifiée` | `Document submitted, not verified` |
| `Vérifiés` (verif stat-label) | `Verified` |
| `Sans pièce soumise` (verif stat-label) | `No document submitted` |
| `Pas encore de document` | `No document yet` |
| `Taux de vérification` (verif stat-label) | `Verification rate` |
| `Dossiers à vérifier` | `Cases to review` |
| `En attente` (verif filter btn) | `Pending` |
| `Vérifiés` (verif filter btn) | `Verified` |
| `Tous` (verif filter btn) | `All` |
| `Pièce d'identité` (verif th) | `ID document` |
| `Action` (verif th) | `Action` |
| `🔍 Recherche avancée` | `🔍 Advanced search` |
| `Recherche un email, un Stripe Payment ID, un nom d'utilisateur... pour retracer l'historique complet.` | `Search an email, a Stripe Payment ID, a username... to trace the full history.` |
| `Rechercher` (search button) | `Search` |
| `Détails` (panel default title) | `Details` |

Notes:
- Several headers/labels appear more than once (e.g. `Chargement...`, `Date`, `Statut`,
  `Utilisateur`, `Tous`, `Vérifiés`). `Date` is already English-identical (leave it). For the
  repeated ones, translate **every** occurrence in the static HTML — use
  `git grep -n 'Chargement'` etc. to confirm none remain in the body.
- Do not touch French text inside `<script>` yet — that's Task 7.

- [ ] **Step 3: Verify**

Reload, log in. Expected: login screen, sidebar, topbar, all page headers, table headers,
placeholders, filter buttons and loading states are in English. `git grep -nE '(Chargement|Utilisateurs|Trajets|Paiements|Déconnexion|Recherche|Vérifi)' index.html` returns only matches inside `<script>` (handled next), none in the HTML body.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Translate static HTML UI to English"
```

---

## Task 7: English translation — JS strings + date locale

**Files:**
- Modify: `index.html` script: `fmtDate` (~781-785), login error (~654), `titles` map (~699-703), all render-function strings, panel labels, stats labels, search labels, verification strings, confirm/alert messages, "Oui/Non" badges.

- [ ] **Step 1: Date locale**

In `fmtDate` (lines ~783-784) change both `'fr-FR'` to `'en-AU'`:
```js
  return d.toLocaleDateString('en-AU', { day: '2-digit', month: '2-digit', year: 'numeric' }) + ' ' + d.toLocaleTimeString('en-AU', { hour: '2-digit', minute: '2-digit' });
```

- [ ] **Step 2: Apply the JS string map**

Replace each French string (left) with English (right) inside `<script>`. Keep all code,
template-literal placeholders `${...}`, and markup identical — change only the human text.

| French (in JS) | English |
|---|---|
| `'Erreur : ' + err.message` | `'Error: ' + err.message` |
| `titles` map: `'Vue d\'ensemble'` | `'Overview'` |
| `titles` map: `'Utilisateurs'` | `'Users'` |
| `titles` map: `'Trajets'` | `'Trips'` |
| `titles` map: `'Paiements'` | `'Payments'` |
| `titles` map: `'Wallet Transactions'` | `'Wallet Transactions'` |
| `titles` map: `'Recherche avancée'` | `'Advanced search'` |
| `titles` map: `'Statistiques'` | `'Statistics'` |
| `titles` map: `'Vérification ID'` | `'ID Verification'` |
| `Aucun utilisateur trouvé` | `No users found` |
| `Aucun trajet trouvé` | `No trips found` |
| `Aucun paiement trouvé` | `No payments found` |
| `Aucun paiement` (overview empty) | `No payments` |
| `Aucune transaction` (wallet empty) | `No transactions` |
| `Oui` (isVerified badge) | `Yes` |
| `Non` (isVerified badge) | `No` |
| `Nom` (panel label) | `Name` |
| `Email` | `Email` |
| `Téléphone` | `Phone` |
| `Statut` (all panel labels) | `Status` |
| `UID` | `UID` |
| `Vérifié` (panel label) | `Verified` |
| `Staff` | `Staff` |
| `Inscrit le` (panel label) | `Joined` |
| `💰 Wallet & Banque` | `💰 Wallet & Bank` |
| `Coordonnées bancaires` (privacy row, from Task 3) | `Bank details` |
| `Masquées (mode privacy)` | `Hidden (privacy mode)` |
| `Titulaire` | `Account holder` |
| `Bank complet` | `Bank complete` |
| `🚗 Trajets comme driver` | `🚗 Trips as driver` |
| `Aucun trajet` | `No trips` |
| `places` (e.g. `${t.reservedSeats||0}/${t.totalSeats||'?'} places`) | `seats` |
| `💳 Paiements comme passager` | `💳 Payments as passenger` |
| `Aucun paiement` | `No payments` |
| `🔄 Wallet Transactions` | `🔄 Wallet Transactions` |
| `Aucune transaction` | `No transactions` |
| `Aucun dossier dans cette catégorie` | `No cases in this category` |
| `Vérifié ✓` (status badge) | `Verified ✓` |
| `En attente` (status badge) | `Pending` |
| `Pas de pièce` | `No document` |
| `Erreur image` | `Image error` |
| `Approuver ✓` | `Approve ✓` |
| `✓ Approuvé` | `✓ Approved` |
| `✓ Vérifié` (panel status) | `✓ Verified` |
| `En attente de review` (panel status) | `Pending review` |
| `Pas de pièce soumise` (panel status) | `No document submitted` |
| `📄 Pièce d'identité` | `📄 ID document` |
| `🔒 Masqué (mode privacy)` (from Task 4) | `🔒 Hidden (privacy mode)` |
| `Cliquer pour ouvrir en plein écran` | `Click to open full screen` |
| `Impossible de charger l\'image` | `Could not load image` |
| `Aucun document soumis` | `No document submitted` |
| `Actions` (panel h4) | `Actions` |
| `✓ Approuver la pièce` | `✓ Approve document` |
| `✗ Rejeter la pièce` | `✗ Reject document` |
| `Rejeter supprime le document et notifie l'utilisateur de re-soumettre` | `Rejecting deletes the document and notifies the user to resubmit` |
| `✓ Utilisateur vérifié` | `✓ User verified` |
| `Vérification — ` (panel title prefix) | `Verification — ` |
| `Enregistrement...` (approve btn) | `Saving...` |
| `✓ Approuvé !` | `✓ Approved!` |
| `Erreur lors de la mise à jour : ` | `Error while updating: ` |
| `Rejeter la pièce d'identité ? L'utilisateur devra en soumettre une nouvelle.` | `Reject the ID document? The user will have to submit a new one.` |
| `Erreur : ` (reject catch) | `Error: ` |
| `Chargement des statistiques...` | `Loading statistics...` |
| `Utilisateurs total` | `Total users` |
| `drivers` (e.g. `${driverCount} drivers · ${passengerCount} passagers`) | `drivers` |
| `passagers` | `passengers` |
| `Taux de vérification` (stats) | `Verification rate` |
| `vérifiés` (`${verifiedUsers} / ${totalUsers} vérifiés`) | `verified` |
| `Revenue total` | `Total revenue` |
| `Commission : ` | `Commission: ` |
| `Wallet en circulation` | `Wallet in circulation` |
| `trajets` (`${allTrips.length} trajets · ${allPayments.length} paiements`) | `trips` |
| `paiements` | `payments` |
| `Nationalités des utilisateurs` | `User nationalities` |
| `renseignées` (`${usersWithNat} renseignées / ${totalUsers}`) | `provided` |
| `Aucune nationalité renseignée` | `No nationality provided` |
| `Non renseignée` | `Not provided` |
| `Statuts des trajets` | `Trip statuses` |
| `total` (`${allTrips.length} total`) | `total` |
| `Profils utilisateurs` | `User profiles` |
| `Drivers actifs` | `Active drivers` |
| `Passagers actifs` | `Active passengers` |
| `Les deux` | `Both` |
| `Profil incomplet (bank)` | `Incomplete profile (bank)` |
| `Aucun rôle` | `No role` |
| `🗺️ Trajet` (trip panel h4) | `🗺️ Trip` |
| `De` (trip panel label) | `From` |
| `Vers` (trip panel label) | `To` |
| `Départ` (trip panel label) | `Departure` |
| `Arrivée` (trip panel label) | `Arrival` |
| `🪑 Places & Prix` | `🪑 Seats & Price` |
| `Prix / place` (trip panel label) | `Price / seat` |
| `Total affiché` | `Displayed total` |
| `Places totales` | `Total seats` |
| `Places réservées` | `Reserved seats` |
| `Places dispo` | `Available seats` |
| `👤 Driver` (trip panel h4) | `👤 Driver` |
| `Cliquer pour voir le profil` (all 3 panels) | `Click to view profile` |
| `💸 Calcul payout driver` | `💸 Driver payout calculation` |
| `Total paiements reçus` | `Total payments received` |
| `Commission 15%` (all occurrences) | `Commission 15%` |
| `À virer au driver` | `To pay the driver` |
| `💳 Paiements liés` | `💳 Linked payments` |
| `📊 Commission Records` | `📊 Commission Records` |
| `Montant commission` | `Commission amount` |
| `% commission` | `Commission %` |
| `Prix total trajet` | `Trip total price` |
| `💳 Paiement` (payment panel h4) | `💳 Payment` |
| `Montant` (panel labels, all) | `Amount` |
| `Stripe ID` (payment panel label) | `Stripe ID` |
| `Discount` | `Discount` |
| `Promo code` | `Promo code` |
| `👤 Passager` (payment panel h4) | `👤 Passenger` |
| `🚗 Trajet associé` | `🚗 Associated trip` |
| `💸 Décomposition` | `💸 Breakdown` |
| `Montant payé` | `Amount paid` |
| `Dont frais réservation ($4)` | `Incl. booking fee ($4)` |
| `Prix trajet (hors frais)` | `Trip price (excl. fees)` |
| `Part driver` | `Driver share` |
| `Paiement ` (payment panel title prefix, line ~1135) | `Payment ` |
| `🔄 Transaction Wallet` (wallet panel h4) | `🔄 Wallet Transaction` |
| `👤 Utilisateur` (wallet panel h4) | `👤 User` |
| `Transaction Wallet` (wallet panel title, line ~1161) | `Wallet Transaction` |
| `Recherche...` (search loading) | `Searching...` |
| `👤 Utilisateur` (search result type) | `👤 User` |
| `💳 Paiement` (search result type) | `💳 Payment` |
| `🚗 Trajet` (search result type) | `🚗 Trip` |
| `Driver: ` (search sub) | `Driver: ` |
| `Aucun résultat pour ` | `No results for ` |
| `résultat(s) trouvé(s)` | `result(s) found` |
| `inconnu` (default trip status) | `unknown` |

- [ ] **Step 3: Verify no French UI text remains**

Run: `git grep -nE 'Chargement|Utilisateur|Trajet|Paiement|Vérifi|Recherche|Aucun|Téléphone|Banque|Approuv|Rejeter|passager|nationalit|renseign' index.html`
Expected: no matches that are human-facing UI text. (Matches inside Firestore field names like
`stripePaymentId` are fine; matches in comments are fine. The grep is a guide — eyeball the
results.) Reload the app and click through every page + open a user, trip, payment, wallet, and
verification panel; run a search; open Statistics. Confirm all visible text is English and dates
render day/month/year.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Translate JS-generated UI strings to English and switch date locale to en-AU"
```

---

## Task 8: Final end-to-end verification (both modes)

**Files:**
- None modified (verification only). If issues are found, fix in `index.html` and amend the
  relevant task's commit or add a follow-up commit.

- [ ] **Step 1: Owner (full access) walkthrough**

Log in as `valpdu44@gmail.com`. Confirm: no privacy pill; UI fully English; Users table shows
real names/emails; user panel shows real phone + bank details (BSB/account/holder); Verification
shows real ID thumbnails/images and Approve/Reject buttons work; Payments show full Stripe IDs;
Statistics and search show real data.

- [ ] **Step 2: Restricted (privacy) walkthrough**

Simulate the buyer: in console run `PRIVACY_MODE = true;` then visit each page via
`switchPage(...)` (and re-open panels). Confirm the full spec checklist:
- "🔒 Privacy mode" pill (set `document.getElementById('privacyPill').style.display='inline-flex'` to view it).
- Names → `User XXXX`; emails → `j•••@•••`; phones → `••• ••• ••NN`.
- User panel: bank section shows "Bank details — Hidden (privacy mode)"; wallet balance still visible.
- Verification: ID thumbnails/images replaced by 🔒 placeholder; no Approve/Reject.
- Stripe IDs everywhere → `•••`.
- Wallet balances, amounts, nationalities, aggregate stats still visible.
- Search returns pseudonymized titles and masked subtitles.

- [ ] **Step 3: Real second-account check (optional but recommended)**

If feasible, create a throwaway Firebase Auth account NOT in `FULL_ACCESS_EMAILS`, log in with
it, and confirm privacy mode engages automatically on real login (not just via the console
toggle) and cannot be turned off from the UI.

- [ ] **Step 4: Final commit (if any fixes were made)**

```bash
git add index.html
git commit -m "Fix issues found during final privacy/translation verification"
```

---

## Self-review notes (already reconciled against the spec)

- Spec "pseudonymize identities" → Tasks 2, 3, 4 (name via `pseudoName`, email via `maskEmail`,
  phone via `maskPhone`).
- Spec "bank details fully hidden" → Task 3 Step 3.
- Spec "ID photos fully hidden" → Task 4 Steps 1–2.
- Spec "Stripe IDs masked" → Tasks 3 (sub-list), 5 (tables + panels).
- Spec "Approve/Reject hidden in privacy mode" → Task 4 Steps 1–2.
- Spec "wallet balances / aggregate stats visible" → preserved in Task 3 Step 3; stats page not gated.
- Spec "hardcoded email allowlist, forced, no toggle" → Task 1 Steps 1–2.
- Spec "visible privacy pill" → Task 1 Steps 2–3.
- Spec "full English translation + en-AU dates + don't translate DB values" → Tasks 6, 7.
- Method/name consistency: `pseudoName`, `maskEmail`, `maskPhone`, `maskId`, `PRIVACY_MODE`,
  `FULL_ACCESS_EMAILS`, `privacyPill` used identically across all tasks.
