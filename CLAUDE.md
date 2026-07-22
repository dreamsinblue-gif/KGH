# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

운송기사 배송 입력 시스템 — a delivery-tracking app for drivers and admins. See [3_개발기획서.md](3_개발기획서.md) for the full plan (screens, data model, status). No backend server: Firebase Firestore handles realtime sync, and Google Apps Script backs up Firestore to a Google Sheet on a 5-minute trigger.

**Deployed instances:**
- Firebase project: `delivery-tracker-9b6d7`
- Driver app: https://delivery-tracker-9b6d7.web.app/1_입력시스템(index).html
- Admin app: https://delivery-tracker-9b6d7.web.app/2_관리자대시보드(admin).html — admin login is `dreamsinblue@gmail.com` (Firebase Authentication user; password set in Firebase Console, not stored anywhere in this repo)
- Apps Script backup: source lives outside this repo at `../delivery-tracker-backup-script` (sibling folder, deployed via `clasp`), bound script project "배송기록 자동백업" — writes to Google Sheet https://docs.google.com/spreadsheets/d/1wUtlIpiU9eme7I1aPWtXKlQadPhOEj6W74VVBEgEZtQ/edit (tab `배송기록`). Runs on OAuth from the project-owner Google account, calling the Firestore REST API directly (bypasses `firestore.rules`, which only applies to Firebase client SDKs). Re-deploy with `clasp push` from that folder; re-run `setup()` in the Apps Script editor only if the 5-minute trigger needs recreating.

## Structure & running

There is no build step, package manager, or test suite — this is two standalone static HTML files, each a self-contained vanilla-JS app (no framework, no bundler) with Firebase compat SDK loaded via `<script>` tags from a CDN:

- [1_입력시스템(index).html](1_입력시스템(index).html) — driver-facing app. Two-step flow: select region + stores (2단계 등록), then bulk-register a simultaneous "운송시작" (start) timestamp for all selected stores. Lists registered entries with a start→complete flow indicator; "운송완료" (complete) writes an end timestamp, with a reset action to clear it.
- [2_관리자대시보드(admin).html](2_관리자대시보드(admin).html) — read-only admin app with two views toggled by local `view` state: `dashboard` (today's per-store status: 대기/운송중/완료) and `stats` (30-day lookback comparing each store's monthly average duration vs. a selected day's average).

To run either app, just open the `.html` file in a browser (or serve the directory statically) — no install/build/lint/test commands exist in this repo.

## Architecture notes

**Shared Firestore schema, no shared code.** Both apps independently read/write the same `deliveries` collection (`store`, `region`, `date`, `startAt`, `endAt`, `createdAt`), each with its own copy of `firebaseConfig`, `initFirebase`, timestamp/date helpers, and SVG icon functions. There is no shared module — a schema or helper change must be applied to both files by hand.

**Store list duplication.** The set of 36 stores (24 수도권 / 12 지방) is defined twice and must stay in sync and in the same display order:
- index.html: `STORE_DEFS` (array of `{name, region}`)
- admin.html: `METRO` + `RURAL` (flat name arrays, concatenated as `ALL_STORES`)

**Firebase config is live, not a placeholder.** Both files' `firebaseConfig` point at the `delivery-tracker-9b6d7` project and are deployed (see 배포된 인스턴스 above). If this project is ever swapped for a different Firebase project, `firebaseConfig` must be updated identically in both HTML files.

**Admin access setup is done.** admin.html gates the dashboard behind `firebase.auth().signInWithEmailAndPassword()` using the `ADMIN_EMAIL` constant defined in admin.html. This required two manual steps in the Firebase Console, already completed: (1) creating that exact email/password user under Authentication → Users, and (2) publishing [`firestore.rules`](firestore.rules)'s contents to Firestore Database → Rules (also deployable via `firebase deploy --only firestore:rules` using `firebase.json`/`.firebaserc` in this repo root). `ADMIN_EMAIL` in admin.html and the email hardcoded in `firestore.rules` must stay in sync if the admin account ever changes.

**`request.auth.token.email` and `Timestamp` methods have sharp edges in `firestore.rules`.** Comparing a claim that doesn't exist on the token (e.g. `email` for the driver app's anonymous sessions) throws and denies the whole rule, even inside `||` — guard with `'email' in request.auth.token` first. Separately, `Timestamp.year()/.month()/.day()` take no arguments and always read UTC; there is no timezone-parameter overload despite what stale examples online suggest — Asia/Seoul is computed by shifting `request.time` by 9 hours before reading date parts. Both are explained inline in `firestore.rules`; both only surfaced once rules were actually deployed against a live anonymous session, so re-verify live (not just `compiled successfully`) after editing this file.

**Concurrent-driver duplicate guard.** Multiple drivers can use index.html at once against the same shared `deliveries` collection, and documents have no owner/driver field to reconcile duplicates after the fact. Step 1's store selection (`renderStep1`/`takenBy()`) disables any store that already has an entry for today (showing 등록됨/완료 instead of a selectable chip) so two drivers can't both start the same store — "전체 선택" for 지방 also skips already-taken stores. `takenBy()` reads the full `entries` array directly, independent of what the bottom list renders.

**Completed entries are hidden, with no undo.** `render()` filters `entries` down to `visibleEntries` (only `!e.endAt`) before rendering the bottom list, so a completed delivery disappears from index.html entirely — there is no 초기화/reset button anymore (`resetEntryFirestore` was removed). A driver who taps 운송완료 by mistake cannot self-correct; admin.html has no reset capability either, so fixing a wrong completion currently requires editing Firestore directly.

**Render pattern.** Both apps use a manual `render()` that rebuilds `#app` innerHTML from module-level state, followed by a `wire()` call to reattach event listeners — no virtual DOM/diffing. Any state mutation must be followed by calling `render()`.

**Realtime subscriptions.** Firestore `onSnapshot` listeners drive re-renders on remote changes (`subscribeToday` in index.html; `subscribeDashboard` and `subscribeStats` in admin.html). Both start together as soon as the admin logs in — the dashboard's per-store delay indicator and the stats page's monthly averages read from the same 30-day `statsEntries` dataset, fetched once (`statsSubscribing` guard prevents re-subscribing when the stats page is later opened). Date-comparison recomputation on the stats page still happens client-side on selection change, not by re-querying. A 60-second `dashboardTimer` re-renders the dashboard so in-progress elapsed-time displays keep advancing between Firestore updates.
