# Codebase Improvements

Identified during code review on 2026-05-02.

---

## Bugs

### 1. `connectionLost` banner is dead UI
**File:** `src/Board.svelte:45`
`connectionLost` is declared as `false` and never set to `true`. The disconnection alerts at lines 289–303 and 350–363 can never appear. Either wire up a Firestore `onSnapshot` error handler to set it, or remove the dead UI.

### 2. CSV export has wrong timestamp
**File:** `src/components/Menu.svelte:91`
`card.created_at` is already a `Date` object (constructed in `src/firestore.js:121`), but `downloadCSV` does `new Date(card.created_at * 1000)`. Coercing a `Date` via multiplication calls `.valueOf()`, producing a timestamp ~1000× too far in the future.
**Fix:** replace with `card.created_at.toISOString()`.

### 3. Missing `await` on `undoReact`
**File:** `src/components/Card.svelte:107`
```js
if (card.reacted === emoji) {
  return undoReact($board, card);  // no await — errors silently dropped
}
```
The undo path is not awaited, so network errors are never caught or surfaced to the user.

### 4. Dead `{:else}` branch in Rank card list
**File:** `src/components/Rank.svelte:231`
`{#if $cards}` always evaluates truthy — `$cards` is a Svelte store wrapping an array, never falsy. The `{:else}` block (lines 245–249) is unreachable and also missing the `data-card-id` attribute required for drag-and-drop.

### 5. Board owner misses realtime sync when `open_permission` is active
**File:** `src/Board.svelte:164`
Only `!$board.owner` subscribes to Firestore board updates. When another `open_permission` user changes settings (e.g. toggles voting or obscure cards), the original owner never sees those changes until reload. The owner should also subscribe to remote board changes when `open_permission` is true.

---

## Incomplete Svelte 5 Migration

### 6. `run()` from `svelte/legacy` used in 8 files
Should be replaced with `$derived` or `$effect`:

| File | Line | Usage | Replacement |
|---|---|---|---|
| `src/Board.svelte` | 93 | sortedRanks sort | `$derived` |
| `src/components/Rank.svelte` | 49 | columnWidth computation | `$derived` |
| `src/components/Timer.svelte` | 106 | updateInterval call | `$effect` |
| `src/components/Menu.svelte` | 39 | auto-show timer on active | `$effect` |
| `src/components/IceBreaker.svelte` | 43 | CSS class string | `$derived` |
| `src/components/Alert.svelte` | 20 | CSS class string | `$derived` |
| `src/components/Button.svelte` | 24 | CSS class string | `$derived` |
| `src/components/Input.svelte` | 43 | CSS class string | `$derived` |
| `src/components/Textarea.svelte` | 43 | CSS class + autoresize reset | `$derived` / `$effect` |

### 7. `createEventDispatcher` and `on:event` syntax still used throughout
Svelte 5 uses callback props instead. Files still using the legacy pattern:
`Card.svelte`, `Rank.svelte`, `Menu.svelte`, `ReactDrawer.svelte`, `BoardRow.svelte`, `PasswordWall.svelte`, `BoardTable.svelte`, `Votes.svelte`, `IceBreaker.svelte`, `Header.svelte`, `RankOptions.svelte`

### 8. `createBubbler` still used in primitive components
**Files:** `src/components/Button.svelte:4`, `src/components/Input.svelte:4`, `src/components/Textarea.svelte:4`
`createBubbler` is a legacy compat shim. Should be replaced with explicit callback props passed through to the underlying element.

---

## i18n Gap

### 9. Entire "features" section of Splash.svelte is hardcoded English
**File:** `src/Splash.svelte:146–226`
Five feature cards and all their body text are raw English strings not wrapped in `$_()`. Affected content:
- Card titles: "Anonymous", "Simple, Clean & Intuitive", "Free & Open Source", "No Logins!", "End to End Encryption"
- All paragraph text within each card

Every non-English locale sees these in English. Keys should be added to `src/lang/en.json` and translations requested.

---

## Dependencies

### 10. `moment.js` (~300KB) used only for `fromNow()`
**Files:** `src/components/BoardRow.svelte:91`, `src/components/LocaleSelect.svelte:21`, `src/i18n.js:2–19`
The only user-facing use is a single relative timestamp. Can be replaced with the native `Intl.RelativeTimeFormat` API, eliminating a large dependency.

Also: moment locales are imported for `es`, `ko`, `de`, `ru`, `uk` but **not** `pt_BR` or `tr` — so those two locales currently display English relative timestamps.

### 11. Deprecated `event.keyCode`
**Files:** `src/components/Input.svelte:25,33`, `src/components/Textarea.svelte:27,35`
`keyCode` is deprecated. Replace with `event.key === 'Enter'` and `event.key === 'Escape'`.

---

## Dead Code

### 12. `getCards` exported but never imported
**File:** `src/api.js:108`
Cards are fetched via Firestore realtime subscription. The REST `getCards` export is unused.

### 13. Dead CSS in `Board.svelte`
**File:** `src/Board.svelte:381–443`
The following CSS rules reference elements or components no longer in the template:
- `.add-button` (and its `@media` variant)
- `.new-card-form`
- `.overflow-x-hidden`
- `:global(.svelte-tabs)`, `:global(.svelte-tabs__tab-list)`, `:global(.svelte-tabs li.svelte-tabs__tab)` — references a `svelte-tabs` component that no longer exists

### 14. Dead CSS in `Rank.svelte`
**File:** `src/components/Rank.svelte:263`
`.header { text-align: center; }` is defined but the `.header` class is not used anywhere in the template.

### 15. Commented-out delete board feature
**File:** `src/components/Menu.svelte:124–131`, `286–297`
Both `doDeleteBoard` and its menu item are commented out. Either complete and ship the feature or remove the dead code.

---

## Performance

### 16. `EncryptedText` double async waterfall per render
**File:** `src/components/EncryptedText.svelte`
Every render awaits `decrypt()` then serially awaits `checkBoardPassword()`. The password check result doesn't change during a session — it could be derived once in a parent and passed as a prop, or memoized in a store.
