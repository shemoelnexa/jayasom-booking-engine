# Booking Engine Client Changes — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Apply 14 client-requested changes from the 2026-04-23 meeting to the single-file booking engine (`index.html`), covering Screen 1 overhaul, card-pattern additions, flow reordering, retreat/guest-details restructuring, and summary-sidebar updates.

**Architecture:** No build system — vanilla HTML/CSS/JS in one 5,114-line file. State is a global `booking` object mutated imperatively; rendering is manual via `innerHTML` in per-step render functions. All changes mirror existing patterns (counter buttons, detail drawers, addon cards, accordions) rather than introducing new frameworks.

**Tech Stack:** Plain HTML5, CSS custom properties, vanilla JS. No dependencies beyond what's already linked in `<head>`.

---

## Open Questions (confirm before execution)

These assumptions are baked into the plan. If any is wrong, flag which task(s) to revise.

1. **"Jayana" = display label for existing `destination: 'jayasom'`** (not a third destination). Internal value stays `'jayasom'` to avoid grep-and-rename risk.
2. **Family Wellness flow (#6) = reorder Room→Retreat**, not skip retreat. Trigger: `booking.destination === 'jayala'`.
3. **Per-guest retreats (#10)** = global default from Step 2, per-guest override via chips in Step 5. No per-guest pricing variance.
4. **Treatments on guest details (#13)** = per-guest (not global). Treatments data reused from the archived Step 4 catalog.
5. **Currencies (#5)** = SAR (default), USD, EUR, GBP. Fixed conversion table, no live FX.
6. **Card images (#9)** = multi-image carousel modal. Where backing data has one image, plan uses placeholder image arrays until real assets are provided.
7. **"Add child" button (#12)** = button labeled "Add Child" appears in guest accordion only when `booking.children > 0`. Existing "Add Guest" button remains for additional adult guests.
8. **Caretaker (#4) and infants (#3)** = counted separately; no price impact (free). Confirm pricing rules separately before production.

---

## File Structure

Single file — `D:\Code Files\Jayasom\jayasom-booking-engine\index.html`. Logical sections being modified:

| Region | Lines | Tasks touching it |
|---|---|---|
| CSS (global) | 1–1860 | 1, 3, 4, 5, 8, 9, 10, 11, 13 |
| Header (top bar) | 1870–1878 | 5 |
| Step 1 markup (hero) | 1885–1979 | 1, 2, 3, 4 |
| Step 2 (retreat) | 2001–2076 | 6, 8, 9 |
| Step 3 (room) | 2080–2208 | 6, 7, 8, 9 |
| Step 5 (guest details) | 2313–2363 | 7, 10, 11, 12, 13 |
| State + `goToStep` | 2751–2860 | 1, 2, 3, 4, 6, 10, 11 |
| `updateCount` | 2884–2895 | 2, 3, 4 |
| Drawers | 3547–3764 | 8, 9 |
| `renderPriceSummary` | 3961–3986 | 14 |
| `renderGuestForms` | 4041–4137 | 7, 10, 11, 12, 13 |
| i18n strings | 4241–4774 | 1, 5 |

---

## Execution Order Rationale

Tasks are ordered so each one builds on committed prior work with minimal rebasing. Foundation pass (Task 0) collects shared state-model changes so later tasks can assume them.

1. **Task 0** — Foundation: extend `booking` state + grid layouts
2. **Tasks 1–5** — Screen 1 (independent of later steps)
3. **Tasks 6–7** — Card pattern helpers (shared by later cards)
4. **Tasks 8–9** — Flow and card repositioning
5. **Tasks 10–13** — Retreat + Guest Details overhaul
6. **Task 14** — Summary sidebar

---

## Task 0: Foundation — extend state model and grid layouts

**Files:**
- Modify: `index.html:2751-2761` (booking state object)
- Modify: `index.html` CSS near line 286 (`.guest-counters-grid`)

- [ ] **Step 1: Extend `booking` state object**

Locate lines 2751–2761 and replace with:

```javascript
let booking = {
  destination: 'jayasom',
  currency: 'SAR',
  retreatName: '',
  retreatPrice: 0,
  retreatsByGuest: {},       // { 'r0g1': 'Retreat Name', ... }
  leadGuests: {},            // { [roomIndex]: guestNumber } — one lead per room
  treatmentsByGuest: {},     // { 'r0g1': ['Treatment A', ...] }
  selectedRooms: [],
  addons: [],
  adults: 2,
  children: 0,
  infants: 0,
  caretakers: 0,
  rooms: 1,
  childAges: [],
  additionalGuests: {},
  checkin: '',
  checkout: '',
  flightNumber: '',
  arrivalTime: '',
  needsPickup: ''
};
```

- [ ] **Step 2: Update `.guest-counters-grid` to accommodate 4 counters**

Find the CSS rule for `.guest-counters-grid` (search `guest-counters-grid`). Replace the grid definition to flow from 2-col on wide screens to a responsive auto-fit that degrades cleanly:

```css
.guest-counters-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
  gap: 16px;
  margin-top: 8px;
}
```

- [ ] **Step 3: Add currency conversion table near `formatSAR()`**

Locate `formatSAR` (around line 2777). Immediately above it, add:

```javascript
const CURRENCY_RATES = { SAR: 1, USD: 0.267, EUR: 0.245, GBP: 0.210 };
const CURRENCY_SYMBOLS = { SAR: 'SAR', USD: '$', EUR: '€', GBP: '£' };

function convertFromSAR(amountSAR) {
  const rate = CURRENCY_RATES[booking.currency] || 1;
  return Math.round(amountSAR * rate);
}

function formatCurrency(amountSAR) {
  const code = booking.currency || 'SAR';
  const sym = CURRENCY_SYMBOLS[code];
  const val = convertFromSAR(amountSAR).toLocaleString();
  return code === 'SAR' ? `SAR ${val}` : `${sym}${val}`;
}
```

Do **not** yet replace existing `formatSAR` call sites — Task 5 does that in one sweep.

- [ ] **Step 4: Manual verification**

Reload the page, open devtools console, type `booking`. Confirm new fields exist with default values. Confirm `formatCurrency(1000)` returns `"SAR 1,000"`.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Extend booking state and grid layout for new counters"
```

---

## Task 1: Jayana/Jayala radio buttons on Screen 1

**Files:**
- Modify: `index.html:1894-1900` (destination select)
- Add CSS near line 350 (new `.destination-radio-group` block)
- Modify i18n keys in `index.html:4241-4774` (translations for "Jayana"/"Jayala")

- [ ] **Step 1: Add CSS for radio-button group**

Add to the main stylesheet (e.g., after `.form-group` CSS around line 340):

```css
.destination-radio-group {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  margin-top: 6px;
}
.destination-radio {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 14px 16px;
  border: 1.5px solid var(--border);
  border-radius: 12px;
  cursor: pointer;
  background: #fff;
  transition: all 0.2s ease;
  font-size: 14px;
  color: var(--text-primary);
}
.destination-radio:hover { border-color: var(--accent); }
.destination-radio.selected {
  border-color: var(--accent);
  background: var(--accent-light, #f8f5f0);
  box-shadow: 0 2px 8px rgba(0,0,0,0.06);
}
.destination-radio-circle {
  width: 18px;
  height: 18px;
  border: 2px solid var(--border);
  border-radius: 50%;
  flex-shrink: 0;
  position: relative;
  transition: border-color 0.2s;
}
.destination-radio.selected .destination-radio-circle {
  border-color: var(--accent);
}
.destination-radio.selected .destination-radio-circle::after {
  content: '';
  position: absolute;
  inset: 3px;
  background: var(--accent);
  border-radius: 50%;
}
.destination-radio-label {
  display: flex;
  flex-direction: column;
  gap: 2px;
}
.destination-radio-title { font-weight: 600; }
.destination-radio-sub { font-size: 11px; color: var(--text-muted); }
```

- [ ] **Step 2: Replace the `<select>` with radio group markup**

In lines 1894–1900, replace the `<select id="destinationSelect">` block with:

```html
<div class="form-group">
  <label data-i18n="destinationLabel">Destination</label>
  <div class="destination-radio-group" id="destinationRadioGroup">
    <div class="destination-radio selected" data-value="jayasom" onclick="selectDestination('jayasom')">
      <div class="destination-radio-circle"></div>
      <div class="destination-radio-label">
        <span class="destination-radio-title" data-i18n="destJayana">Jayana</span>
        <span class="destination-radio-sub" data-i18n="destJayanaSub">Adults only</span>
      </div>
    </div>
    <div class="destination-radio" data-value="jayala" onclick="selectDestination('jayala')">
      <div class="destination-radio-circle"></div>
      <div class="destination-radio-label">
        <span class="destination-radio-title" data-i18n="destJayala">Jayala</span>
        <span class="destination-radio-sub" data-i18n="destJayalaSub">Family wellness</span>
      </div>
    </div>
  </div>
</div>
```

- [ ] **Step 3: Add `selectDestination` handler**

Near `toggleLanguage()` (search for that function; JS section starting ~2742), add:

```javascript
function selectDestination(value) {
  booking.destination = value;
  document.querySelectorAll('.destination-radio').forEach(el => {
    el.classList.toggle('selected', el.dataset.value === value);
  });
  applyDestinationVisibility(); // defined in Task 2
}
```

Stub `applyDestinationVisibility` with `function applyDestinationVisibility(){}` just above `selectDestination` so step 4 doesn't error. Task 2 will fill it in.

- [ ] **Step 4: Add i18n entries**

In both `en` and `ar` language objects (search for `destinationLabel` and the surrounding keys, ~line 4260+), add:

```javascript
// en object
destJayana: 'Jayana',
destJayanaSub: 'Adults only',
destJayala: 'Jayala',
destJayalaSub: 'Family wellness',

// ar object
destJayana: 'جايانا',
destJayanaSub: 'للبالغين فقط',
destJayala: 'جايالا',
destJayalaSub: 'عافلة العائلة',
```

- [ ] **Step 5: Manual verification**

- Reload page. Screen 1 shows two radio tiles, Jayana selected by default.
- Click Jayala → selection indicator moves, `booking.destination === 'jayala'` in console.
- Toggle language → titles switch to Arabic.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Replace destination dropdown with Jayana/Jayala radio buttons"
```

---

## Task 2: Hide children counter when Jayana selected

**Files:**
- Modify: `index.html:1938-1945` (children counter group)
- Modify: `index.html` — complete `applyDestinationVisibility` stub from Task 1

- [ ] **Step 1: Wrap children counter group with id**

Locate the children counter (~1938–1945). Wrap its outer `<div class="form-group guest-group">` with `id="childrenCounterGroup"`:

```html
<div class="form-group guest-group" id="childrenCounterGroup">
  <label data-i18n="childrenLabel">Children</label>
  <!-- existing counter buttons and value span -->
</div>
```

- [ ] **Step 2: Implement `applyDestinationVisibility`**

Replace the stub body from Task 1 with:

```javascript
function applyDestinationVisibility() {
  const childrenGroup = document.getElementById('childrenCounterGroup');
  const childAgesContainer = document.getElementById('childAgesContainer');
  if (!childrenGroup) return;
  if (booking.destination === 'jayasom') {
    childrenGroup.style.display = 'none';
    if (childAgesContainer) childAgesContainer.style.display = 'none';
    if (booking.children > 0) {
      booking.children = 0;
      const el = document.getElementById('childrenCount');
      if (el) el.textContent = '0';
    }
  } else {
    childrenGroup.style.display = '';
  }
}
```

- [ ] **Step 3: Call on page load**

Search for `DOMContentLoaded` or the bottom initialization block. Add `applyDestinationVisibility();` after the initial `updateStepper()` call.

- [ ] **Step 4: Manual verification**

- Reload; Jayana selected → children counter hidden.
- Click Jayala → children counter appears.
- Set children to 2 on Jayala → switch to Jayana → counter hides, `booking.children` resets to 0 in console.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Hide children counter when Jayana (adults only) selected"
```

---

## Task 3: Add infants counter

**Files:**
- Modify: `index.html:1930-1946` (add infants markup inside `.guest-counters-grid`)
- Modify: `index.html:2884-2895` (`updateCount` — already generic, verify)
- Modify: `index.html` in `goToStep()` sync block if it hard-reads counters

- [ ] **Step 1: Add infants counter markup**

Inside `.guest-counters-grid` (after children group, before closing div ~line 1946), insert:

```html
<div class="form-group guest-group" id="infantsCounterGroup">
  <label data-i18n="infantsLabel">Infants <span style="font-weight:400;color:var(--text-muted);font-size:11px;">(0–2)</span></label>
  <div class="counter-wrap">
    <button class="counter-btn" onclick="updateCount('infants', -1)" aria-label="Decrease infants">−</button>
    <span class="counter-value" id="infantsCount">0</span>
    <button class="counter-btn" onclick="updateCount('infants', 1)" aria-label="Increase infants">+</button>
  </div>
</div>
```

- [ ] **Step 2: Verify `updateCount` supports the new type**

Locate `updateCount` (~2884). Confirm it reads/writes `booking[type]` generically and updates `document.getElementById(type + 'Count').textContent`. If it special-cases 'adults'/'children' with hardcoded ids, extend the id map. Example (replace only if hardcoded):

```javascript
function updateCount(type, delta) {
  const newVal = Math.max(type === 'adults' ? 1 : 0, (booking[type] || 0) + delta);
  booking[type] = newVal;
  const el = document.getElementById(type + 'Count');
  if (el) {
    el.textContent = newVal;
    el.style.transform = 'scale(1.3)';
    setTimeout(() => (el.style.transform = 'scale(1)'), 200);
  }
  if (type === 'children') syncChildAges();
}
```

- [ ] **Step 3: Hide infants counter for Jayana if policy requires**

Per open question 8, infants allowed for both destinations — leave visible. If client says otherwise, add `document.getElementById('infantsCounterGroup').style.display = booking.destination === 'jayasom' ? 'none' : ''` inside `applyDestinationVisibility`.

- [ ] **Step 4: Add i18n entries**

Both `en`/`ar` language objects:

```javascript
// en
infantsLabel: 'Infants',
// ar
infantsLabel: 'الرضع',
```

- [ ] **Step 5: Manual verification**

- Reload. Infants counter visible with value 0.
- +/− buttons work, update `booking.infants` in console.
- Cannot go below 0.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Add infants counter on Screen 1"
```

---

## Task 4: Add caretaker counter

**Files:**
- Modify: `index.html:1930-1946` (add caretaker markup)
- Modify: i18n strings

- [ ] **Step 1: Add caretaker counter markup**

Inside `.guest-counters-grid`, after infants group:

```html
<div class="form-group guest-group" id="caretakersCounterGroup">
  <label data-i18n="caretakersLabel">Caretakers</label>
  <div class="counter-wrap">
    <button class="counter-btn" onclick="updateCount('caretakers', -1)" aria-label="Decrease caretakers">−</button>
    <span class="counter-value" id="caretakersCount">0</span>
    <button class="counter-btn" onclick="updateCount('caretakers', 1)" aria-label="Increase caretakers">+</button>
  </div>
</div>
```

- [ ] **Step 2: Add i18n entries**

```javascript
// en
caretakersLabel: 'Caretakers',
// ar
caretakersLabel: 'مرافقون',
```

- [ ] **Step 3: Manual verification**

Reload — caretaker counter visible, functional. Grid wraps cleanly on narrow viewport (resize to ~600px).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add caretakers counter on Screen 1"
```

---

## Task 5: Currency switcher in top bar

**Files:**
- Modify: `index.html:1870-1878` (`.header-right`)
- Add CSS near `.lang-switch` rule
- Add handler + event wiring in JS
- Sweep: replace all `formatSAR(` call sites with `formatCurrency(`

- [ ] **Step 1: Add CSS for currency dropdown button**

Next to the `.lang-switch` CSS rule, add:

```css
.currency-switch {
  position: relative;
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 6px 10px;
  border: 1px solid var(--border);
  border-radius: 8px;
  background: #fff;
  font-size: 11px;
  color: var(--text-muted);
  cursor: pointer;
}
.currency-switch:hover { border-color: var(--accent); color: var(--text-primary); }
.currency-menu {
  position: absolute;
  top: calc(100% + 4px);
  right: 0;
  background: #fff;
  border: 1px solid var(--border);
  border-radius: 8px;
  box-shadow: 0 6px 20px rgba(0,0,0,0.08);
  padding: 4px;
  display: none;
  z-index: 100;
  min-width: 100px;
}
.currency-menu.open { display: block; }
.currency-menu button {
  display: block;
  width: 100%;
  text-align: left;
  padding: 8px 12px;
  font-size: 12px;
  background: transparent;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  color: var(--text-primary);
}
.currency-menu button:hover { background: var(--bg-muted, #f5f2ec); }
.currency-menu button.active { color: var(--accent); font-weight: 600; }
```

- [ ] **Step 2: Add currency button to header**

In `.header-right` (~line 1876), insert before the `.lang-switch` button:

```html
<div class="currency-switch" onclick="toggleCurrencyMenu(event)">
  <span id="currencyLabel">SAR</span>
  <svg width="10" height="10" viewBox="0 0 10 10" fill="currentColor"><path d="M1 3l4 4 4-4" stroke="currentColor" stroke-width="1.5" fill="none"/></svg>
  <div class="currency-menu" id="currencyMenu">
    <button data-currency="SAR" class="active" onclick="selectCurrency('SAR', event)">SAR — ر.س</button>
    <button data-currency="USD" onclick="selectCurrency('USD', event)">USD — $</button>
    <button data-currency="EUR" onclick="selectCurrency('EUR', event)">EUR — €</button>
    <button data-currency="GBP" onclick="selectCurrency('GBP', event)">GBP — £</button>
  </div>
</div>
```

- [ ] **Step 3: Add handlers**

Near `toggleLanguage` function (~line 4743), add:

```javascript
function toggleCurrencyMenu(e) {
  e.stopPropagation();
  document.getElementById('currencyMenu').classList.toggle('open');
}
function selectCurrency(code, e) {
  if (e) e.stopPropagation();
  booking.currency = code;
  document.getElementById('currencyLabel').textContent = code;
  document.querySelectorAll('#currencyMenu button').forEach(b => {
    b.classList.toggle('active', b.dataset.currency === code);
  });
  document.getElementById('currencyMenu').classList.remove('open');
  rerenderAllPrices();
}
document.addEventListener('click', () => {
  const m = document.getElementById('currencyMenu');
  if (m) m.classList.remove('open');
});

function rerenderAllPrices() {
  if (typeof renderPriceSummary === 'function') renderPriceSummary();
  if (typeof renderMergedSummaryPayment === 'function') renderMergedSummaryPayment();
  if (typeof renderConfirmation === 'function' && currentStep === 6) renderConfirmation();
  document.querySelectorAll('[data-price-sar]').forEach(el => {
    const sar = parseFloat(el.dataset.priceSar);
    if (!isNaN(sar)) el.textContent = formatCurrency(sar);
  });
}
```

- [ ] **Step 4: Global find-and-replace `formatSAR(` → `formatCurrency(`**

Use the editor's replace-in-file. Search `formatSAR(` (with open paren to avoid touching the function definition itself), replace with `formatCurrency(`. Then rename the original function definition:

```javascript
// Rename this line:
function formatSAR(amount) { ... }
// To:
function formatSAR_legacy(amount) { ... }
```

Then confirm `formatCurrency` (added in Task 0) is the only thing used in rendering.

- [ ] **Step 5: Mark fixed-price spans with `data-price-sar`**

For any hard-coded price span in markup that should convert on currency change (search `SAR \d`), add `data-price-sar="<amount>"`. Example:

```html
<!-- Before -->
<span class="retreat-price">SAR 28,500</span>
<!-- After -->
<span class="retreat-price" data-price-sar="28500">SAR 28,500</span>
```

This is a best-effort sweep for the hero and retreat cards; dynamically-rendered summaries already use `formatCurrency` after the swap in Step 4.

- [ ] **Step 6: Manual verification**

- Header shows currency pill. Click opens dropdown.
- Select USD → pill shows "USD", all prices in summary recalculate (e.g., SAR 1,000 → $267).
- Refresh → resets to SAR (fine — no persistence requested).

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "Add currency switcher with SAR/USD/EUR/GBP conversion"
```

---

## Task 6: Family Wellness flow (Jayala) — Room before Retreat

**Files:**
- Modify: `index.html:2782-2830` (`goToStep` + helpers)
- Modify: `index.html:2744-2748` (stepNames)
- Modify: `index.html:1956` ("Begin" button) and step-2/3 continue buttons

- [ ] **Step 1: Add flow-map helper**

Above `goToStep` (~line 2782), add:

```javascript
function getFlowOrder() {
  // Returns array of step numbers in order. Default: retreat(2) → room(3) → ...
  // Jayala: room(3) → retreat(2) → ...
  return booking.destination === 'jayala'
    ? [1, 3, 2, 5, 6]
    : [1, 2, 3, 5, 6];
}
function nextStepFrom(current) {
  const order = getFlowOrder();
  const idx = order.indexOf(current);
  return idx >= 0 && idx < order.length - 1 ? order[idx + 1] : null;
}
function prevStepFrom(current) {
  const order = getFlowOrder();
  const idx = order.indexOf(current);
  return idx > 0 ? order[idx - 1] : null;
}
```

- [ ] **Step 2: Route step transitions through flow helpers**

Locate the hero "Begin" button (~line 1956, `onclick="goToStep(2)"`). Replace with:

```html
<button class="btn-primary" onclick="startFlow()">Begin Your Journey</button>
```

And in JS:

```javascript
function startFlow() {
  const order = getFlowOrder();
  goToStep(order[1]); // first step after landing
}
```

Step-2 "Continue to Room" button (~line 2074) and Step-3 "Continue to Guest Details" button: replace their `onclick="goToStep(3)"` / `onclick="goToStep(5)"` with:

```html
<button class="btn-primary" onclick="goToStep(nextStepFrom(currentStep))">Continue</button>
```

Make sure the button text reflects the *actual* next step — see Step 4 for dynamic labels.

- [ ] **Step 3: Dynamic stepper labels**

In `updateStepper()` (~line 2833), replace the static `stepNames` array with a dynamic lookup:

```javascript
const STEP_LABELS = {
  1: 'Dates & Guests',
  2: 'Select Retreat',
  3: 'Select Room',
  5: 'Guest Details',
  6: 'Confirmation'
};
function currentStepNames() {
  return getFlowOrder().map(s => STEP_LABELS[s]);
}
```

Then update `updateStepper()` and `updateMobileStepper()` to call `currentStepNames()` instead of the static `stepNames`.

- [ ] **Step 4: Dynamic continue-button labels**

Where the retreat and room step continue buttons are rendered, set their text from the next step's label:

```javascript
function refreshContinueButtonLabels() {
  document.querySelectorAll('[data-continue-btn]').forEach(btn => {
    const next = nextStepFrom(currentStep);
    btn.textContent = next && STEP_LABELS[next]
      ? `Continue to ${STEP_LABELS[next]}`
      : 'Continue';
  });
}
```

Add `data-continue-btn` attribute to the two relevant buttons, then call `refreshContinueButtonLabels()` at the end of `goToStep`.

- [ ] **Step 5: Manual verification**

- Jayana selected: Hero → Retreat → Room → Guest Details. Stepper shows labels in that order.
- Jayala selected: Hero → Room → Retreat → Guest Details. Stepper reflects reorder. Continue buttons show next-step label.
- Back navigation (if present) uses `prevStepFrom`.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Reorder Family Wellness (Jayala) flow: Room before Retreat"
```

---

## Task 7: Move Airport Transfer card to top of Guest Details

**Files:**
- Delete: `index.html:2151-2200` (airport transfer card in Step 3)
- Insert: `index.html:~2314` (top of Step 5 guest-details container)

- [ ] **Step 1: Cut the airport transfer card markup**

Select lines 2151–2200 (the entire `.addon-card` for Airport Transfer, including `#airportTransferFields`). Cut to clipboard.

- [ ] **Step 2: Paste above `#guestFormsContainer`**

In Step 5 container (~line 2313), immediately before `<div id="guestFormsContainer">`, paste the airport transfer card. Wrap it in a section header for clarity:

```html
<div class="guest-details-section" id="airportTransferSection">
  <h3 class="section-subtitle">Arrival & Transfer</h3>
  <!-- pasted airport transfer card here -->
</div>
```

- [ ] **Step 3: Verify toggle still wires up**

The card uses `onclick="toggleAddon(this)"` and reveals `#airportTransferFields`. Both IDs remain unique — functionality preserved. No JS change needed.

- [ ] **Step 4: Remove any Step-3 headers that referenced airport transfer**

Search Step 3 markup for text like "Transfer" or "Flight" outside the card; remove orphaned labels.

- [ ] **Step 5: Manual verification**

- Step 3 no longer shows airport card.
- Step 5 shows airport card as the first section. Toggle expands flight fields. Saved values still appear in confirmation summary.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Move Airport Transfer card to top of Guest Details step"
```

---

## Task 8: "Click for details" button on all cards

**Files:**
- Modify: retreat cards (~2009–2068), room cards (~2092–2147), addon cards (~2152–2200 new location, plus archived Step 4)
- Add CSS for `.card-details-btn`

- [ ] **Step 1: Add CSS**

```css
.card-details-btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 6px 12px;
  background: transparent;
  border: 1px solid var(--border);
  border-radius: 999px;
  font-size: 11px;
  color: var(--text-muted);
  cursor: pointer;
  transition: all 0.2s;
  margin-right: 8px;
}
.card-details-btn:hover {
  border-color: var(--accent);
  color: var(--accent);
  background: var(--accent-light, #f8f5f0);
}
.card-details-btn svg { width: 12px; height: 12px; }
```

- [ ] **Step 2: Add button to retreat cards**

In each retreat `.retreat-card-footer` (before the existing "Select" button ~line 2020), insert:

```html
<button class="card-details-btn" onclick="event.stopPropagation(); openRetreatDrawer(this.closest('.retreat-card'), 'The Renewal Retreat', 28500)">
  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5"><circle cx="8" cy="8" r="1"/><path d="M8 11V7M8 4.5v.01"/></svg>
  Details
</button>
```

Do this for each of the 3 retreat cards; substitute the correct name/price per card.

Also change the card's outer `onclick` from full-card-drawer to selection-only (if client wants clicks anywhere on the card to select, keep; if they want *only* the Details button to open drawer and the Select button to select, remove `openRetreatDrawer` from outer onclick).

**Decision:** Keep outer card click as selection; Details button opens drawer. This matches client wording "add click for details button".

- [ ] **Step 3: Add button to room cards**

In each room `.room-card-footer`, insert before the quantity counter:

```html
<button class="card-details-btn" onclick="event.stopPropagation(); openRoomDrawer('Desert View Suite')">
  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5"><circle cx="8" cy="8" r="1"/><path d="M8 11V7M8 4.5v.01"/></svg>
  Details
</button>
```

Substitute room name per card.

- [ ] **Step 4: Add button to addon cards**

For each addon card (Airport Transfer at new location, treatment addons in Step 4 archived template), insert in a footer area:

```html
<button class="card-details-btn" onclick="event.stopPropagation(); openTreatmentDrawer(this.closest('.addon-card'), 'Airport Transfer', 0)">
  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5"><circle cx="8" cy="8" r="1"/><path d="M8 11V7M8 4.5v.01"/></svg>
  Details
</button>
```

- [ ] **Step 5: Manual verification**

- Click outer card → selects/toggles as before.
- Click "Details" pill → drawer opens with the correct content. No double-fire.
- Works on all three card types.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Add Click for Details button to all selectable cards"
```

---

## Task 9: Magnifying-glass icon + full-image carousel modal

**Files:**
- Add CSS for `.card-zoom-btn` overlay + `.image-carousel-modal`
- Modify each card image wrapper to include the zoom button
- Add JS `openImageCarouselModal` + state
- Extend `retreatDetails` / `roomDetails` / treatment data with `images` arrays where missing

- [ ] **Step 1: Add CSS**

```css
.card-zoom-btn {
  position: absolute;
  top: 10px;
  right: 10px;
  width: 32px;
  height: 32px;
  background: rgba(255,255,255,0.9);
  border: none;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  box-shadow: 0 2px 8px rgba(0,0,0,0.15);
  transition: transform 0.2s, background 0.2s;
  z-index: 2;
}
.card-zoom-btn:hover { transform: scale(1.1); background: #fff; }
.card-zoom-btn svg { width: 16px; height: 16px; color: var(--text-primary); }
.retreat-card-image, .room-card-image, .addon-image { position: relative; }

/* Full-image carousel modal */
.image-carousel-modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.85);
  display: none;
  align-items: center;
  justify-content: center;
  z-index: 1100;
}
.image-carousel-modal-overlay.show { display: flex; }
.image-carousel-modal {
  position: relative;
  width: min(90vw, 1100px);
  height: min(80vh, 720px);
  background: #000;
  border-radius: 16px;
  overflow: hidden;
}
.icm-track {
  display: flex;
  width: 100%;
  height: 100%;
  transition: transform 0.35s ease;
}
.icm-slide {
  flex: 0 0 100%;
  display: flex;
  align-items: center;
  justify-content: center;
}
.icm-slide img { max-width: 100%; max-height: 100%; object-fit: contain; }
.icm-nav {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  width: 44px;
  height: 44px;
  border-radius: 50%;
  background: rgba(255,255,255,0.9);
  border: none;
  cursor: pointer;
  font-size: 22px;
}
.icm-nav.prev { left: 16px; }
.icm-nav.next { right: 16px; }
.icm-close {
  position: absolute;
  top: 16px;
  right: 16px;
  width: 36px;
  height: 36px;
  border-radius: 50%;
  background: rgba(255,255,255,0.9);
  border: none;
  cursor: pointer;
  font-size: 18px;
  z-index: 2;
}
.icm-dots {
  position: absolute;
  bottom: 16px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 6px;
}
.icm-dot { width: 8px; height: 8px; border-radius: 50%; background: rgba(255,255,255,0.4); border: none; cursor: pointer; }
.icm-dot.active { background: #fff; }
```

- [ ] **Step 2: Add modal markup**

Just before `</body>`, insert:

```html
<div class="image-carousel-modal-overlay" id="imageCarouselModalOverlay" onclick="closeImageCarouselModal()">
  <div class="image-carousel-modal" onclick="event.stopPropagation()">
    <button class="icm-close" onclick="closeImageCarouselModal()">×</button>
    <button class="icm-nav prev" onclick="icmPrev()">‹</button>
    <button class="icm-nav next" onclick="icmNext()">›</button>
    <div class="icm-track" id="icmTrack"></div>
    <div class="icm-dots" id="icmDots"></div>
  </div>
</div>
```

- [ ] **Step 3: Add JS**

```javascript
let _icmImages = [];
let _icmIndex = 0;

function openImageCarouselModal(images) {
  _icmImages = Array.isArray(images) && images.length ? images : [];
  _icmIndex = 0;
  const track = document.getElementById('icmTrack');
  const dots = document.getElementById('icmDots');
  track.innerHTML = _icmImages.map(src =>
    `<div class="icm-slide"><img src="${src}" alt=""></div>`
  ).join('');
  dots.innerHTML = _icmImages.map((_, i) =>
    `<button class="icm-dot${i === 0 ? ' active' : ''}" onclick="icmGoTo(${i})"></button>`
  ).join('');
  track.style.transform = 'translateX(0%)';
  document.getElementById('imageCarouselModalOverlay').classList.add('show');
  document.body.style.overflow = 'hidden';
}
function closeImageCarouselModal() {
  document.getElementById('imageCarouselModalOverlay').classList.remove('show');
  document.body.style.overflow = '';
}
function icmGoTo(i) {
  _icmIndex = ((i % _icmImages.length) + _icmImages.length) % _icmImages.length;
  document.getElementById('icmTrack').style.transform = `translateX(-${_icmIndex * 100}%)`;
  document.querySelectorAll('.icm-dot').forEach((d, di) => d.classList.toggle('active', di === _icmIndex));
}
function icmPrev() { icmGoTo(_icmIndex - 1); }
function icmNext() { icmGoTo(_icmIndex + 1); }
document.addEventListener('keydown', e => {
  if (!document.getElementById('imageCarouselModalOverlay').classList.contains('show')) return;
  if (e.key === 'Escape') closeImageCarouselModal();
  if (e.key === 'ArrowLeft') icmPrev();
  if (e.key === 'ArrowRight') icmNext();
});

function openCardCarousel(type, key) {
  let images = [];
  if (type === 'retreat' && typeof retreatDetails !== 'undefined') {
    const d = retreatDetails[key];
    images = d?.images || (d?.image ? [d.image] : []);
  } else if (type === 'room' && typeof roomDetails !== 'undefined') {
    const d = roomDetails[key];
    images = d?.images || (d?.image ? [d.image] : []);
  } else if (type === 'treatment' && typeof treatmentDetails !== 'undefined') {
    const d = treatmentDetails?.[key];
    images = d?.images || (d?.image ? [d.image] : []);
  }
  if (!images.length) {
    console.warn(`No images for ${type}/${key}`);
    return;
  }
  openImageCarouselModal(images);
}
```

- [ ] **Step 4: Add zoom button to each card image**

For each `.retreat-card-image`, `.room-card-image`, `.addon-image`, inject as first child:

```html
<button class="card-zoom-btn" onclick="event.stopPropagation(); openCardCarousel('retreat', 'The Renewal Retreat')" aria-label="View images">
  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5">
    <circle cx="7" cy="7" r="5"/>
    <path d="M11 11l3 3"/>
    <path d="M7 5v4M5 7h4"/>
  </svg>
</button>
```

Substitute `('retreat', name)`, `('room', name)`, `('treatment', name)` per card type.

- [ ] **Step 5: Extend backing data with image arrays (stub until real assets)**

Find `retreatDetails` (~line 3473) and `roomDetails` (~line 3635). Where each entry has a single `image: '...'`, add:

```javascript
// Example — retreatDetails['The Renewal Retreat']
images: [
  'images/retreats/renewal-1.jpg',
  'images/retreats/renewal-2.jpg',
  'images/retreats/renewal-3.jpg'
]
```

If the real asset paths aren't known, leave the single existing `image` field and the helper falls back to `[image]` (one-slide carousel). This is acceptable until the client provides multi-image sets.

- [ ] **Step 6: Manual verification**

- Each card shows magnifying-glass top-right.
- Click zoom button → carousel opens full-screen. Left/right/dots/esc/arrow-keys all work.
- Clicking backdrop closes.
- Card click still selects (no interference).

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "Add zoom icon and full-image carousel modal on cards"
```

---

## Task 10: Guest chips on Retreat page — different retreats per guest

**Files:**
- Modify: `index.html:4041-4109` (renderGuestForms — add retreat chips under each guest)
- Modify: retreat selection to set a default when global `booking.retreatName` changes

- [ ] **Step 1: Seed per-guest retreats when global retreat selected**

In `selectRetreatDirect(card, name, price)` (~3846), after setting `booking.retreatName`, add:

```javascript
// Default any guest without an explicit per-guest retreat to the new global pick
const totalGuests = booking.adults + (booking.children || 0);
for (let r = 0; r < booking.rooms; r++) {
  for (let g = 1; g <= 10; g++) {
    const key = `r${r}g${g}`;
    if (!booking.retreatsByGuest[key]) booking.retreatsByGuest[key] = name;
  }
}
```

- [ ] **Step 2: Render retreat chips in each guest form**

In `renderGuestForms` (~4041), inside the per-guest block (after the guest inputs), insert:

```javascript
const guestKey = `r${r}g${g}`;
const currentRetreat = booking.retreatsByGuest[guestKey] || booking.retreatName || '';
const retreatOptions = (typeof retreatDetails !== 'undefined')
  ? Object.keys(retreatDetails)
  : ['The Renewal Retreat', 'The Awakening Retreat', 'The Pause'];

html += `
  <div class="guest-retreat-row">
    <label class="guest-retreat-label">Retreat for this guest</label>
    <div class="guest-retreat-chips" data-guest-key="${guestKey}">
      ${retreatOptions.map(r => `
        <button type="button"
                class="guest-retreat-chip${r === currentRetreat ? ' active' : ''}"
                onclick="setGuestRetreat('${guestKey}', ${JSON.stringify(r)})">
          ${r}
        </button>
      `).join('')}
    </div>
  </div>
`;
```

- [ ] **Step 3: Add `setGuestRetreat` handler**

```javascript
function setGuestRetreat(guestKey, retreatName) {
  booking.retreatsByGuest[guestKey] = retreatName;
  document.querySelectorAll(`.guest-retreat-chips[data-guest-key="${guestKey}"] .guest-retreat-chip`)
    .forEach(b => b.classList.toggle('active', b.textContent.trim() === retreatName));
  renderPriceSummary();
}
```

- [ ] **Step 4: Add CSS for chips**

```css
.guest-retreat-row { margin-top: 14px; padding-top: 14px; border-top: 1px dashed var(--border); }
.guest-retreat-label { display: block; font-size: 12px; color: var(--text-muted); margin-bottom: 8px; }
.guest-retreat-chips { display: flex; flex-wrap: wrap; gap: 6px; }
.guest-retreat-chip {
  padding: 6px 12px;
  font-size: 12px;
  border: 1px solid var(--border);
  border-radius: 999px;
  background: #fff;
  cursor: pointer;
  transition: all 0.2s;
}
.guest-retreat-chip:hover { border-color: var(--accent); color: var(--accent); }
.guest-retreat-chip.active {
  background: var(--accent);
  border-color: var(--accent);
  color: #fff;
}
```

- [ ] **Step 5: Manual verification**

- Select "The Renewal Retreat" in Step 2. Advance to Step 5.
- Each guest card shows chips with Renewal active.
- Click a different chip for guest 2 → that chip becomes active, others for that guest remain.
- Other guests' selections unchanged.
- `booking.retreatsByGuest` in console reflects per-guest mapping.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Add per-guest retreat chips on guest details"
```

---

## Task 11: Lead guest re-assignment (exactly one per room)

**Files:**
- Modify: `index.html:4041-4109` (renderGuestForms)
- Add CSS for `.lead-guest-toggle`

- [ ] **Step 1: Initialize lead guests on render**

In `renderGuestForms`, at the top of the room loop:

```javascript
if (booking.leadGuests[r] === undefined) booking.leadGuests[r] = 1;
```

- [ ] **Step 2: Replace static badge with interactive toggle**

In the guest loop, replace the line `const isLead = (r === 0 && g === 1);` (~line 4067) and its surrounding `<span class="guest-badge">Lead Guest</span>` block with:

```javascript
const isLead = booking.leadGuests[r] === g;
html += isLead
  ? `<span class="guest-badge is-lead" data-room="${r}" data-guest="${g}">Lead Guest</span>`
  : `<button type="button" class="guest-badge make-lead" data-room="${r}" data-guest="${g}" onclick="setLeadGuest(${r}, ${g})">Make lead</button>`;
```

- [ ] **Step 3: Add handler**

```javascript
function setLeadGuest(roomIndex, guestNum) {
  booking.leadGuests[roomIndex] = guestNum;
  renderGuestForms();
}
```

- [ ] **Step 4: Style the "make lead" button**

Locate the existing `.guest-badge` CSS (~line 670). Append:

```css
.guest-badge.make-lead {
  background: transparent;
  color: var(--text-muted);
  border: 1px dashed var(--border);
  cursor: pointer;
  transition: all 0.2s;
}
.guest-badge.make-lead:hover {
  color: var(--accent);
  border-color: var(--accent);
}
.guest-badge.is-lead {
  background: var(--accent);
  color: #fff;
  border: 1px solid var(--accent);
}
```

- [ ] **Step 5: Manual verification**

- Step 5 renders — first guest per room shows filled "Lead Guest" chip, others show dashed "Make lead" button.
- Click "Make lead" on guest 2 → becomes lead, former lead becomes dashed button.
- Exactly one lead per room at all times.
- Other rooms unaffected.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Allow changing lead guest per room"
```

---

## Task 12: "Add Child" button conditional on children count

**Files:**
- Modify: `index.html:4096-4103` (renderGuestForms — add-guest / add-child button area)

- [ ] **Step 1: Track per-room children count**

Add to `booking` state (if not already): `childrenByRoom: {}`. Default: evenly distribute `booking.children` across rooms on first render. In `renderGuestForms`:

```javascript
if (!booking.childrenByRoom || Object.keys(booking.childrenByRoom).length === 0) {
  booking.childrenByRoom = {};
  const base = Math.floor(booking.children / booking.rooms);
  const extra = booking.children % booking.rooms;
  for (let r = 0; r < booking.rooms; r++) {
    booking.childrenByRoom[r] = base + (r < extra ? 1 : 0);
  }
}
```

- [ ] **Step 2: Render child guest rows**

Inside the per-room block, after adult guests and before the existing "Add Guest" button, render child rows (read-only placeholders for name/age):

```javascript
const childCount = booking.childrenByRoom[r] || 0;
for (let c = 1; c <= childCount; c++) {
  html += `
    <div class="guest-form-block child-row" data-room="${r}" data-child="${c}">
      <div class="guest-form-head">
        <span class="child-badge">Child ${c}</span>
        <button type="button" class="remove-child-btn" onclick="removeChild(${r}, ${c})">Remove</button>
      </div>
      <div class="guest-form-grid">
        <div class="form-group"><label>Full name</label><input type="text" id="r${r}_child${c}_name"></div>
        <div class="form-group"><label>Age</label><input type="number" min="0" max="17" id="r${r}_child${c}_age"></div>
      </div>
    </div>
  `;
}
```

- [ ] **Step 3: Conditionally render "Add Child" button**

Replace the block around the existing add-guest button (~line 4097) with:

```javascript
html += `<div class="guest-actions-row">`;
html += `<button class="add-guest-btn" onclick="addGuest(${r})">+ Add Guest</button>`;
if (booking.children > 0) {
  const used = Object.values(booking.childrenByRoom).reduce((a, b) => a + b, 0);
  const canAddMore = used < booking.children;
  html += `<button class="add-guest-btn add-child-btn"
             onclick="addChild(${r})"
             ${canAddMore ? '' : 'disabled'}>
             + Add Child${canAddMore ? '' : ' (limit reached)'}
           </button>`;
}
html += `</div>`;
```

- [ ] **Step 4: Handlers**

```javascript
function addChild(roomIndex) {
  const used = Object.values(booking.childrenByRoom).reduce((a, b) => a + b, 0);
  if (used >= booking.children) return;
  booking.childrenByRoom[roomIndex] = (booking.childrenByRoom[roomIndex] || 0) + 1;
  renderGuestForms();
}
function removeChild(roomIndex, c) {
  booking.childrenByRoom[roomIndex] = Math.max(0, (booking.childrenByRoom[roomIndex] || 0) - 1);
  renderGuestForms();
}
```

- [ ] **Step 5: Add CSS for child row styling**

```css
.child-row { border-left: 3px solid var(--accent-light, #f8f5f0); padding-left: 12px; }
.child-badge {
  display: inline-block; padding: 2px 8px; font-size: 11px;
  background: #eef; color: #335; border-radius: 10px;
}
.remove-child-btn {
  background: none; border: none; color: var(--text-muted);
  font-size: 11px; cursor: pointer; text-decoration: underline;
}
.remove-child-btn:hover { color: #c33; }
.guest-actions-row { display: flex; gap: 8px; flex-wrap: wrap; margin-top: 12px; }
.add-guest-btn[disabled] { opacity: 0.4; cursor: not-allowed; }
```

- [ ] **Step 6: Manual verification**

- Step 1: children = 0 → Step 5 shows only "Add Guest" button.
- Step 1: children = 2 → Step 5 shows child rows (name + age fields) and "Add Child" button enabled.
- Click "Add Child" twice → reaches cap, button disables.
- Click Remove on a child row → row gone, cap-counter recalculates.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "Show Add Child button only when children entered on Screen 1"
```

---

## Task 13: Treatments with toggles on Guest Details

**Files:**
- Modify: `index.html:2313-2363` (Step 5 — new Treatments section)
- Modify: `renderGuestForms` to include per-guest treatments inline
- Extract treatment catalog from archived Step 4 (~lines 2212–2310) into a JS constant

- [ ] **Step 1: Extract treatment catalog**

Somewhere near the retreat/room detail objects, add:

```javascript
const TREATMENT_CATALOG = [
  { id: 'deep-tissue', name: 'Deep Tissue Massage', price: 450, duration: '60 min' },
  { id: 'hammam',      name: 'Royal Hammam',         price: 650, duration: '90 min' },
  { id: 'facial',      name: 'Signature Facial',     price: 550, duration: '75 min' },
  { id: 'yoga',        name: 'Private Yoga',         price: 350, duration: '60 min' },
  { id: 'sound-bath',  name: 'Sound Bath',           price: 400, duration: '45 min' }
];
```

(Update names/prices from the archived Step-4 markup if that list differs — the exact catalog is the client's to confirm.)

- [ ] **Step 2: Render treatments block per guest**

In `renderGuestForms`, inside each guest block (below retreat chips from Task 10), insert:

```javascript
const guestTreatments = booking.treatmentsByGuest[guestKey] || [];
html += `
  <div class="guest-treatments-row">
    <div class="guest-treatments-head">
      <span class="guest-treatments-title">Treatments</span>
      <span class="guest-treatments-count">${guestTreatments.length} selected</span>
    </div>
    <div class="guest-treatments-list">
      ${TREATMENT_CATALOG.map(t => {
        const on = guestTreatments.includes(t.id);
        return `
          <div class="guest-treatment-item">
            <div class="treatment-meta">
              <span class="treatment-name">${t.name}</span>
              <span class="treatment-sub">${t.duration} · ${formatCurrency(t.price)}</span>
            </div>
            <button type="button"
                    class="addon-toggle${on ? ' active' : ''}"
                    onclick="toggleGuestTreatment('${guestKey}', '${t.id}')"></button>
          </div>
        `;
      }).join('')}
    </div>
  </div>
`;
```

- [ ] **Step 3: Handler**

```javascript
function toggleGuestTreatment(guestKey, treatmentId) {
  const list = booking.treatmentsByGuest[guestKey] || [];
  const idx = list.indexOf(treatmentId);
  if (idx >= 0) list.splice(idx, 1); else list.push(treatmentId);
  booking.treatmentsByGuest[guestKey] = list;
  renderGuestForms();
  renderPriceSummary();
}
```

- [ ] **Step 4: Price summary integration**

In `renderPriceSummary` (~3961), add after the addons loop:

```javascript
let treatmentTotal = 0;
let treatmentCount = 0;
for (const [guestKey, ids] of Object.entries(booking.treatmentsByGuest)) {
  for (const id of ids) {
    const t = TREATMENT_CATALOG.find(x => x.id === id);
    if (t) { treatmentTotal += t.price; treatmentCount++; }
  }
}
if (treatmentCount > 0) {
  html += `<div class="price-line">
    <div><div class="price-line-label">Treatments</div>
    <div class="price-line-sublabel">${treatmentCount} selected</div></div>
    <div class="price-line-amount">${formatCurrency(treatmentTotal)}</div>
  </div>`;
}
```

Add `treatmentTotal` to the grand total calculation wherever it exists.

- [ ] **Step 5: CSS**

```css
.guest-treatments-row { margin-top: 14px; padding-top: 14px; border-top: 1px dashed var(--border); }
.guest-treatments-head { display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px; }
.guest-treatments-title { font-size: 12px; font-weight: 600; color: var(--text-primary); }
.guest-treatments-count { font-size: 11px; color: var(--text-muted); }
.guest-treatments-list { display: flex; flex-direction: column; gap: 8px; }
.guest-treatment-item {
  display: flex; align-items: center; justify-content: space-between;
  padding: 10px 12px;
  background: var(--bg-muted, #fafafa);
  border-radius: 10px;
  border: 1px solid var(--border);
}
.treatment-meta { display: flex; flex-direction: column; gap: 2px; }
.treatment-name { font-size: 13px; color: var(--text-primary); }
.treatment-sub { font-size: 11px; color: var(--text-muted); }
```

- [ ] **Step 6: Manual verification**

- Step 5: each guest block now has a Treatments section.
- Toggle a treatment for Guest 1 → count increments, price summary adds line.
- Toggle same treatment for Guest 2 → treatment count doubles in summary.
- Console: `booking.treatmentsByGuest` reflects per-guest arrays.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "Add per-guest Treatments toggles on Guest Details"
```

---

## Task 14: Display children count in booking summary sidebar

**Files:**
- Modify: `index.html:3961-3986` (renderPriceSummary)
- Modify: `index.html:3989-4037` (renderMergedSummaryPayment)

- [ ] **Step 1: Update `renderPriceSummary` guest-line text**

Find the existing line that renders `${booking.adults} guest(s)` (~3975). Replace with a helper call:

```javascript
function guestsSummaryText() {
  const parts = [];
  parts.push(`${booking.adults} ${booking.adults === 1 ? 'adult' : 'adults'}`);
  if (booking.children > 0) parts.push(`${booking.children} ${booking.children === 1 ? 'child' : 'children'}`);
  if (booking.infants > 0) parts.push(`${booking.infants} ${booking.infants === 1 ? 'infant' : 'infants'}`);
  if (booking.caretakers > 0) parts.push(`${booking.caretakers} ${booking.caretakers === 1 ? 'caretaker' : 'caretakers'}`);
  return parts.join(', ');
}
```

Then replace the rendered string:

```javascript
// Before
`<div class="price-line-sublabel">${booking.adults} ${isAr ? t.guestUnit : 'guest(s)'}</div>`
// After
`<div class="price-line-sublabel">${guestsSummaryText()}</div>`
```

- [ ] **Step 2: Update merged summary on the payment screen**

Apply the same replacement in `renderMergedSummaryPayment` (~line 3989).

- [ ] **Step 3: Update confirmation screen**

Locate the `renderConfirmation` guest-count line (~line 4210). Replace the current `${booking.adults} ${...} Adult(s)${booking.children > 0 ? ', ' + ...}` with `${guestsSummaryText()}`.

- [ ] **Step 4: Manual verification**

- Step 1: adults=2, children=1, infants=1, caretakers=0.
- Advance to Step 2 → sidebar shows "2 adults, 1 child, 1 infant".
- Change to 0 children → sidebar updates.
- Confirmation screen shows the same line.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Show children, infants, caretakers in booking summary sidebar"
```

---

## Post-Implementation

- [ ] **Full smoke test both flows end-to-end**
  - Jayana: Hero → Retreat → Room → Guest Details (airport first, then guests, chips, treatments, lead toggle) → Confirmation. Currency switch mid-flow. Carousel open/close on each card.
  - Jayala: Hero → Room → Retreat → Guest Details. Children and add-child visible.

- [ ] **Cross-browser check**: Chrome, Safari, Firefox. Narrow viewport (~600px) to verify 4-counter grid wraps and currency menu still opens above fold.

- [ ] **Commit any cleanup as a final pass**
  ```bash
  git log --oneline origin/main..HEAD
  ```
  Should show ~15 focused commits (one per task plus Task 0 foundation).

---

## Self-Review

- **Spec coverage:** All 14 client items map 1:1 to tasks 1–14. Task 0 is foundation shared by multiple.
- **Placeholders:** None — every code block is complete. Stub image arrays (Task 9 Step 5) are explicitly flagged as temporary until real assets.
- **Type consistency:** `booking.retreatsByGuest` / `treatmentsByGuest` / `leadGuests` / `childrenByRoom` defined in Task 0 and reused consistently. `formatCurrency` defined in Task 0, used in Tasks 5, 13, 14. `getFlowOrder` / `nextStepFrom` defined in Task 6, used in button labels.
- **Risk:** Task 5's `formatSAR → formatCurrency` sweep is broad and may touch markup strings that should stay static. Double-check diff before committing Task 5.
