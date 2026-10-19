# ScribbleScan handheld: notes for Claude

The phone app (Flutter) for ScribbleScan: snap a receipt, see where your money goes each month, keep budgets, and sign a computer in by scanning a QR code. Android first; iOS later.

**Read first:** `PLAN.md` in this repo (the phases, in order), then the CLAUDE.md of the sister repos, which hold the product decisions and the owner's preferences:

- [ScribbleScan](https://github.com/LyricScripts/ScribbleScan): the API this app talks to (`/v1`, OpenAPI contract at `/openapi.json`), deployment, Zitadel.
- [ScribbleScan-web](https://github.com/LyricScripts/ScribbleScan-web): the web app. Same screens, same API; reuse its wording, colors and behavior so the two feel like one product.
- [pairkit](https://github.com/LyricScripts/pairkit): QR device pairing. This app is the *scanner* that signs a computer in (`grant_to="shower"`).

## The owner

A Principal Software Engineer in Bangladesh building this as a portfolio project. Quality, tests and honest engineering matter more than feature count. Explain simply, and give one step at a time when they must do something themselves (installing tools, plugging in the phone, Google Cloud or Play Console settings). They have a Mac with Android Studio and Flutter installed and test on a real Android phone.

## Rules

- **Work phase by phase from `PLAN.md`.** One phase = one branch + one pull request. Show the plan for a phase before writing its code. Tick the phase off in `PLAN.md` in the same PR.
- **Backend changes go to the ScribbleScan repo** as their own PRs (marked `B1`… in the plan), merged before the app work that needs them. Keep its rule: every route has a response model, and its tests and ruff pass.
- **Talk only to the `/v1` API.** No other backend, no direct database or storage access. The API base URL comes from `--dart-define=API_URL=...`, never hard-coded.
- **Types come from the API's OpenAPI spec** (generated, never hand-edited), same as the web app. When the API changes, refresh `openapi.json` and regenerate. CI checks the generated code is current.
- **Only use things with a future** (the owner's rule): before adding a package, check it is maintained, supports the current stable Flutter, and has a real reason to exist. Prefer Flutter-team and well-established packages; prefer none.
- **Security:** sign-in is OIDC Authorization Code + PKCE in the system browser (never a WebView); tokens only in secure storage; nothing secret in the repo (no keystores, no `google-services.json` with real values, no API keys). HTTP (non-HTTPS) is allowed only in debug builds, for the local API on Wi-Fi.
- **Bangladesh first:** Bangla and English UI, Bangla digits accepted in amounts, ৳ and lakh grouping (৳1,25,000), dates shown and typed day-first (DD/MM/YYYY).
- **Uploads must never be lost or doubled:** a persistent queue, retries with one `Idempotency-Key` per photo, safe across app restarts and bad connections.
- Tests for logic (formatting, queue, auth state, API mapping) and widget tests for screens. `flutter analyze` clean, `dart format` applied, CI green before asking for review.

## Where code can run

A cloud Claude Code session can write code and run `flutter analyze` and `flutter test`, but it cannot reach the owner's phone. Anything that needs the device (camera, sign-in redirect, scanner) is checked by the owner running `flutter run` on their Mac; write down exactly what to tap and what they should see.
