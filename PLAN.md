# Plan

Each phase ends in a working, tested app that the owner can run on their phone. Do them in order; tick the box in the PR that finishes a phase. **B** phases are backend changes in the [ScribbleScan](https://github.com/LyricScripts/ScribbleScan) repo; **A** phases are this app.

Suggested packages are starting points, not decisions: check each is maintained and fits (see CLAUDE.md) before adding it, and say in the PR why it was chosen.

---

## A0. Foundation

- [ ] `flutter create` with org `com.lyricscripts`, app id `com.lyricscripts.scribblescan`, Android and iOS folders (iOS stays unbuilt for now).
- [ ] Folder layout by feature: `lib/app/` (startup, routing, theme), `lib/api/`, `lib/features/<name>/`, `lib/shared/`.
- [ ] Config through `--dart-define`: `API_URL` (e.g. `http://192.168.x.x:8000` in debug), `FLAVOR` (`dev`/`prod`). Debug-only Android network config allowing cleartext HTTP to private addresses; release allows HTTPS only.
- [ ] Theme matching the web app: ledger green `#0F6B57`, cool paper `#F6F8F7`, ink `#1B2A2F`, stamp crimson `#B4233C` (over budget only), the per-category colors from ScribbleScan-web's `app.css`, the Hind Siliguri font bundled (Bangla + Latin), light and dark.
- [ ] Localization set up (`flutter gen-l10n`, ARB files for `en` and `bn`; English strings first, Bangla filled in A9).
- [ ] Routing (suggested: `go_router`) and state management (suggested: `flutter_riverpod`).
- [ ] Lints (`flutter_lints` or stricter), `dart format`, GitHub Actions CI: analyze, test, build a debug APK.
- [ ] README: what it is, how to run on a phone, how to point it at a local API.

**Done when:** the owner runs `flutter run --dart-define=API_URL=http://<PC-IP>:8000` and the app opens with the ScribbleScan look and shows "Server reachable" from `GET /health`.

## A1. API client from the contract

- [ ] `openapi.json` copied from the API; generate a typed Dart client (choose a maintained generator that needs no hand edits; `swagger_parser` or OpenAPI Generator's `dart-dio` are candidates). A script and CI step check the generated code matches `openapi.json`.
- [ ] One HTTP layer: base URL, bearer token, `X-App-Version` header, clear error types (signed out, offline, server problem, validation) mapped from the API's `{detail: {code, message}}` and status codes.
- [ ] Tests against a fake server for success, 401, 422, 5xx and timeouts.

**Done when:** tests pass and a debug screen lists `GET /v1/categories` from the real local API.

## B1. Backend: native sign-in and app version (ScribbleScan PR)

- [ ] `deploy/zitadel_setup.py`: register a **Native** OIDC app (Authorization Code + PKCE, no secret, refresh tokens on), redirect URI `com.lyricscripts.scribblescan:/oauthredirect`, post-logout `com.lyricscripts.scribblescan:/logout`. Same project, so access tokens carry the same audience and the API needs no auth change.
- [ ] `GET /v1/config`: add `mobile_client_id` and `min_app_version` (new settings `SCRIBBLESCAN_OIDC_MOBILE_CLIENT_ID`, `SCRIBBLESCAN_MIN_APP_VERSION`), with response-model and tests.
- [ ] Optional middleware: requests with `X-App-Version` below the minimum get `426 Upgrade Required` with a clear message; requests without the header (the web app) are unaffected.
- [ ] Docs in `deploy/README.md`: how to add Google as a sign-in option in Zitadel (Google Cloud OAuth client, Zitadel identity provider, link accounts by email).

## A2. Sign-in

- [ ] Start-up reads `GET /v1/config`; shows "Update the app" if below `min_app_version`.
- [ ] `dev` mode: email sign-in via `POST /v1/dev/token` (local testing only; hidden in prod builds).
- [ ] `oidc` mode: Authorization Code + PKCE in the system browser (suggested: `flutter_appauth`), refresh tokens, silent renewal, sign-out that also ends the Zitadel session.
- [ ] Tokens in secure storage (suggested: `flutter_secure_storage`); signed-out state handled everywhere (401 → back to sign-in, without losing queued uploads).
- [ ] Tests for token refresh and expiry handling.

**Done when:** the owner signs in on the phone (dev mode locally; Zitadel + Google once deployed), closes and reopens the app and is still signed in, and can sign out.

## A3. Spending and receipts (read-only)

- [ ] Space switcher (`GET /v1/spaces`), remembered between launches.
- [ ] Spending screen (`/summary`): month switcher, total in ৳ with lakh grouping, change from last month, the spend strip by category, budget meters with "Close to budget" / "Over budget", "N receipts need a check".
- [ ] Receipts list (`/expenses`): grouped by day, filters (category, needs a check), pull to refresh.
- [ ] Receipt detail: photo (`/documents/{id}/file`, cached), fields, status.
- [ ] Widget tests for each screen with fake data; formatting tests (lakh, Bangla digits, DD/MM/YYYY).

## A4. Capture and upload

- [ ] Capture: document scanner with edge detection and crop (suggested on Android: Google ML Kit document scanner, on-device), plus pick from gallery and pick PDF.
- [ ] Shrink photos before upload (long side about 2000 px, JPEG), keeping them readable.
- [ ] **Persistent upload queue**: survives app restarts; retries with backoff; one `Idempotency-Key` per photo, kept across retries; background upload when connectivity returns (suggested: `workmanager`). Shows each item: waiting, sending, reading, done, check.
- [ ] After upload, poll the document until it leaves `processing`.
- [ ] Tests for the queue: restart in the middle, duplicate send, server 5xx, offline.

**Done when:** in airplane mode the owner snaps three receipts, turns the network back on, and all three arrive once each.

## A5. Review and correct

- [ ] Review form built from `GET /v1/schemas/{doc_type}` (as the web app does): text, amount, date and row fields; line items/transactions editable.
- [ ] Amounts accept Bangla digits and commas; dates typed and shown DD/MM/YYYY, sent as ISO.
- [ ] Problems in plain words (same texts as the web app's `issueText`), category picker, "Save", "Save as correct anyway", delete, "possible duplicate" notice.
- [ ] Tests for form building and value parsing.

## A6. Budgets, spaces, export

- [ ] Budgets per category (`PUT /budgets`).
- [ ] New space (personal / business).
- [ ] Export the month as Excel or CSV and open the phone's share sheet.
- [ ] Linked devices list with "Disconnect" (`GET/DELETE /v1/devices`).

## B2. Backend: phone signs a computer in (ScribbleScan + ScribbleScan-web PRs)

pairkit already supports this (`grant_to="shower"`). The API needs a new grant:

- [ ] `issue_grant` accepts scope `session`: mint a **desktop session token** (`ds_` prefix, stored hashed like `dt_`), full access for that user, expires (suggest 30 days, renewed on use up to a hard limit), listed in `/v1/devices` with the computer's browser and OS, revocable there.
- [ ] `resolve()` accepts `ds_` tokens as a normal user (not upload-only).
- [ ] Tests: approval only by the signed-in phone user; a photographed QR is useless after the real scan; revoke works at once; expiry; a `dt_` upload token can't approve a sign-in.
- [ ] ScribbleScan-web: on the sign-in page, "Sign in with your phone" shows the QR (`grant_to="shower"`), waits, collects the token, and signs in. Sign-out on that computer deletes the token.

## A7. Sign a computer in by scanning

- [ ] QR scanner screen (suggested: `mobile_scanner`). Only accept links to this app's own domain; anything else shows "This isn't a ScribbleScan code."
- [ ] Claim with the user's token, show "Sign in on Chrome on Windows?", then Approve / Deny.
- [ ] Tests for link parsing and the confirm flow.

**Done when:** the owner opens the web app on a PC, taps "Sign in with your phone", scans, approves, and the PC is signed in; disconnecting it from the phone signs the PC out.

## B3 + A8. Notifications (optional for the first release)

- [ ] Backend: `POST /v1/push-devices` to register a Firebase Cloud Messaging token; send "Receipt ready" / "Needs a check" when processing finishes; budget alert at 80% and 100%. Firebase credentials only through settings, never in git.
- [ ] App: ask for permission at a sensible moment (after the first upload, not at launch); tapping a notification opens that receipt.

## A9. Release preparation

- [ ] Bangla translations for every string; check layouts with Bangla text.
- [ ] App icon and splash in the ScribbleScan style; accessibility pass (screen reader labels, text scaling, contrast).
- [ ] Privacy screen: what is uploaded and where; delete account flow (needs a backend endpoint; add a B task if missing).
- [ ] Release build: signing set up by the owner (keystore kept outside git; write the steps), Play Console internal testing track.
- [ ] Screenshots and README with a short demo GIF.

## Later

iOS build (Xcode on the Mac, Sign in with Apple required if Google sign-in is offered: App Store guideline 4.8), offline viewing of past months, home-screen widget for this month's total.
