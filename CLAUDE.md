# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

운송기사 배송 입력 시스템 — a delivery-tracking app for drivers and admins. See [3_개발기획서.md](3_개발기획서.md) for the full plan (screens, data model, status). No backend server: Firebase Firestore handles realtime sync, and Google Apps Script (external, not in this repo) backs up Firestore to a Google Sheet on a 5-minute trigger.

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

**Firebase config is a placeholder.** Both files ship with `firebaseConfig` containing `YOUR_API_KEY` etc. — a real Firebase project's config must be pasted into both files identically before either app functions (see 향후 계획 in the plan doc).

**Render pattern.** Both apps use a manual `render()` that rebuilds `#app` innerHTML from module-level state, followed by a `wire()` call to reattach event listeners — no virtual DOM/diffing. Any state mutation must be followed by calling `render()`.

**Realtime subscriptions.** Firestore `onSnapshot` listeners drive re-renders on remote changes (`subscribeToday` in index.html; `subscribeDashboard` and `subscribeStats` in admin.html). admin.html's stats view fetches the full 30-day window once (`statsSubscribing` guard prevents re-subscribing) and recomputes date comparisons client-side on selection change, rather than re-querying.
