# Plan: Settings button + Pomodoro duration configuration

## Context
`index.html` is a single-file Pomodoro app. Durations are hardcoded in `DURATIONS`
(line 1196: pomodoro 25m, rest 5m, longRest 15m). There is no way for the user to
change them, and the long-rest interval is not implemented (`nextTabAfterEnd` at
line 1805 never returns `longRest`). We will add a **Settings** button to the left
of the existing **Restart session** button in the topbar, which opens a modal to
configure: pomodoro minutes, rest minutes, long-rest minutes, and "long rest every
N pomodoros". Defaults: 25 / 5 / 15 / 4. Settings persist in localStorage.

## Affected files
- `index.html` (only file) — HTML (topbar + modal), CSS (modal styling), JS (settings state + logic).

## Steps

### 1. Add Settings button (HTML)
In the topbar `.top-actions` (around line 980-985), insert a settings button
**before** the `restartBtn`:
```html
<button id="settingsBtn" class="ghost-link" type="button" aria-label="Settings" title="Settings">
  <span aria-hidden="true">⚙</span>
  <span>Settings</span>
</button>
```
Reuse the existing `.ghost-link` style (already styled in sci-fi theme).

### 2. Add Settings modal (HTML)
Mirror the existing restart modal pattern (lines 1137-1172), place after it. Use
`role="dialog"`, `aria-modal="true"`, `.settings-modal` id, label `settingsTitle`.
Inputs (type number, min enforced in JS):
- `setPomodoro`  — "Pomodoro (minutes)"   default 25
- `setRest`      — "Rest (minutes)"       default 5
- `setLongRest`  — "Long rest (minutes)"  default 15
- `setCycle`     — "Long rest every N pomodoros" default 4

Footer: `settingsCancelBtn` (ghost) + `settingsSaveBtn` (btn). Read current values
from `settings` on open.

### 3. Settings state + persistence (JS)
- Add `settings` LS key to `LS_KEYS` (line 1184), e.g. `settings: "pomo_settings_v1"`.
- Replace the constant `DURATIONS` (line 1196) with a derived object. Define
  `DEFAULT_SETTINGS = { pomodoro: 25, rest: 5, longRest: 15, cycle: 4 }` and a
  `settings` object loaded via `loadSettings()` (parse localStorage, fall back to
  defaults, `clamp`/`Number` coercion). Add helper:
  ```js
  function durationsFromSettings(s) {
    return {
      pomodoro: Math.max(1, s.pomodoro) * 60,
      rest: Math.max(1, s.rest) * 60,
      longRest: Math.max(1, s.longRest) * 60,
    };
  }
  ```
  Keep a mutable `let DURATIONS = durationsFromSettings(settings);` so `tabToDuration`
  (line 1665) keeps working unchanged.
- Add `loadSettings()` / `saveSettings()` mirroring `readSessionCounts`/`writeSessionCounts`.

### 4. Wire Settings modal behavior (JS)
- `openSettings()` / `closeSettings()` toggling `display: grid/none` (mirror
  restart modal functions at line 1879). Populate inputs from `settings`, focus
  first input.
- `settingsBtn` click → `openSettings()`. `settingsCancelBtn` and outside-click and
  `Escape` → `closeSettings()` (mirror lines 1886-1893).
- `settingsSaveBtn` click:
  - Read inputs, coerce to numbers, validate: minutes >= 1, cycle >= 1 (integer).
    If invalid, show inline error / ignore save (revert).
  - Update `settings`, `saveSettings()`, recompute `DURATIONS = durationsFromSettings(settings)`.
  - If `timerState !== "running"`: call `setTimerForTab(currentTab)` so the display
    and remaining time reflect new durations immediately.
  - If `timerState === "running"`: leave the in-progress session untouched; new
    durations apply from the next session (avoid corrupting the live ring progress).
  - `closeSettings()`.

### 5. Implement long-rest cycle (JS)
Update `nextTabAfterEnd` (line 1805) so the configured interval is honored:
```js
function nextTabAfterEnd(tab) {
  if (tab === "pomodoro") {
    // sessionCounts.pomodoro already incremented for this completed pomodoro
    return settings.cycle > 0 && sessionCounts.pomodoro % settings.cycle === 0
      ? "longRest"
      : "rest";
  }
  return "pomodoro";
}
```
Note: `sessionCounts.pomodoro` is incremented before `nextTabAfterEnd` is called
(line 1790), so the modulo check is correct for the just-finished pomodoro.

## Edge cases / risks
- Cycle based on **lifetime** completed pomodoro count (`sessionCounts.pomodoro`,
  persisted). This keeps the cycle stable across reloads; acceptable.
- Changing `cycle` mid-session shifts the modulo boundary — fine, applies next pomodoro.
- Validation: protect against 0/negative/NaN and non-integer cycle; clamp minutes to
  >= 1 so `DURATIONS` stays positive (ring math divides by `durationSeconds`).
- Don't mutate a running timer's `durationSeconds` (would break ring progress display).

## Validation
- Open `index.html` in a browser.
- Verify Settings button appears left of Restart session.
- Open modal, change values (e.g. 10/2/8/2), save; idle timer display updates to the
  new duration. Reload page → values persist.
- Run a pomodoro (use `DEV.setTime(2)`) and confirm: after 2nd/4th/... pomodoro a
  `longRest` tab auto-activates (Long Rest badge increments), otherwise `rest`.
- Invalid input (e.g. 0 or empty) is rejected and settings unchanged.
- Escape / outside-click / Cancel close the modal without saving.
