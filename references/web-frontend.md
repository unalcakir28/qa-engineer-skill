# Web frontend / UI checklist

Merge the applicable items into the Phase 1 matrix. Where a browser is
available (Playwright, or the Chrome tools when present), execute these for
real; where it isn't, drive the API directly and read the component code, and
mark UI-only cases `NOT RUN`.

## Forms and input

- Submit empty; submit with only whitespace; submit with one required field
  missing at a time.
- Client validation vs server validation: bypass the client (devtools, direct
  API call) and confirm the server rejects it too. A field guarded only in the
  browser is a finding.
- Error messages: appear next to the right field, disappear when fixed,
  localised, not stacked/duplicated, don't shift the layout catastrophically.
- Paste (including multi-line and formatted paste), autofill, browser
  autocomplete, drag-and-drop into inputs, IME/composition input.
- Max length: enforced or gracefully truncated — and consistent with the DB
  limit. Very long words/URLs shouldn't break the layout.
- Number inputs: `-`, `,` vs `.` decimal separator (Turkish locale!), leading
  zeros, `e`, spinners at min/max, paste of a non-numeric string.
- Date pickers: manual typing, invalid date, end-before-start, locale format
  (`dd.MM.yyyy`), min/max date, timezone shift on save-then-reload.
- Selects/autocomplete: empty option list, one option, hundreds of options, no
  search results, slow search, keyboard-only selection, clearing the value.
- File upload: wrong type, 0-byte file, huge file, many files, weird filename
  (spaces, Turkish characters, emoji, `../`), cancel mid-upload, upload then
  submit immediately.
- Unsaved-changes protection: navigate away / close tab with a dirty form.

## State, timing and race conditions

- **Double click** the submit button — one record or two? Is the button disabled
  during the request?
- Slow network (throttle to 3G): loading indicators, double submits, timeouts,
  buttons clickable while in flight.
- Submit, then hit browser **Back**, then submit again.
- Refresh (F5) mid-flow: is state preserved or cleanly reset — not half-way.
- Two tabs: edit the same record in both; delete in one and act on it in the
  other; log out in one and keep clicking in the other.
- Session/token expiry mid-flow: does it redirect to login and preserve or
  cleanly discard the work? After re-login, does it return to the right place?
- Rapid interactions: type fast in a debounced search, spam pagination, click
  two filters before the first response lands (out-of-order responses — does the
  stale response overwrite the fresh one?).
- Optimistic updates: what happens when the request then fails — does the UI
  roll back, and does it tell the user?
- Stale cache: create/delete an item and check the list, the counter, the
  detail page and any cached dropdown all update.

## Rendering states

Every data-driven view needs all five checked: **loading, empty, populated,
error, partial/degraded**. The empty and error states are the ones that get
shipped untested.

- Empty: 0 items — is there a sensible empty state, not a broken table?
- Long content: 1 000 rows, very long names, deeply nested trees — overflow,
  truncation with tooltip, scroll performance.
- Error: API 500/timeout/403 — visible message, retry option, no infinite
  spinner, no blank white page.
- Partial data: nullable fields missing — no `undefined`, `NaN`, `null`,
  `Invalid Date`, `[object Object]` rendered on screen.
- Numbers and dates formatted per locale; currency symbol and thousands
  separator right; percentages not off by 100.

## Navigation and URL

- Deep link straight to a detail/step-N page: does it load, or explode because
  it expected state from the previous page?
- Deep link while logged out → login → back to the intended page.
- Invalid/nonexistent id in the URL → clean 404 state, not a crash.
- Query params: missing, garbage, tampered (someone else's id, a bigger
  `pageSize`), and whether filters survive a refresh/share of the URL.
- Browser back/forward through a multi-step flow, modals and filtered lists.
- Route guards: hit an admin-only route's URL directly as a normal user.

## Permissions in the UI

- For each role: which buttons/menus/columns are visible, and does hiding a
  button actually stop the action (the API must refuse it too).
- Read-only users can't mutate through any path — inline edit, keyboard
  shortcut, drag-and-drop, bulk action, export.
- A user whose role changes while logged in.

## Cross-browser, responsive, accessibility (light pass)

- Mobile viewport (375px) and a tablet width: does anything overlap, get cut
  off, or become unclickable? Are modals and tables usable?
- Zoom to 200%; long Turkish/German labels breaking a fixed-width button.
- Keyboard only: tab order, focus visible, `Enter` submits, `Esc` closes the
  modal, focus trapped in the modal and restored on close.
- Basic a11y: inputs have labels, buttons have accessible names, images have
  alt text, error messages are announced, colour isn't the only signal.
- Console: no errors or React key/hydration warnings during the flow — read the
  console after each scenario, not just at the end.
- Network tab: no duplicated requests, no request firing on every keystroke, no
  secrets or tokens in URLs, no request to a hardcoded localhost/dev host.

## Content and injection

- Turkish characters (`İ ı Ş ş Ğ ğ Ç ç Ö ö Ü ü`), emoji, RTL text and 500-char
  strings in every text field — check display, edit-reload round trip, search
  and export.
- XSS: store `<script>alert(1)</script>`, `<img src=x onerror=alert(1)>` and
  `"><svg onload=alert(1)>` in a text field, then view it everywhere it renders
  (list, detail, tooltip, PDF/CSV export, email). Report if executed or if HTML
  breaks out of its container.
- CSV export of a value starting with `=` or `+` (formula injection).
- Markdown/rich text: unclosed tags, links with `javascript:`, huge images.
