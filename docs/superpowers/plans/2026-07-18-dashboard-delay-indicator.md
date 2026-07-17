# 관리자 대시보드 경과시간/지연 표시 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Show, per store, live elapsed time (in-progress) or actual duration (completed) on the admin dashboard, with a "지연" badge when it exceeds the relevant average by 15+ minutes.

**Architecture:** Reuse the existing 30-day `statsEntries` dataset (already computed for the stats page) to derive an expected-duration baseline per store (수도권) or per region (지방, aggregated across all 12 rural stores). Compare each dashboard entry's elapsed/actual minutes against that baseline. A 60-second client-side timer re-renders the dashboard so in-progress elapsed time keeps advancing between Firestore updates.

**Tech Stack:** Vanilla JS in `2_관리자대시보드(admin).html`, no new dependencies.

## Prerequisite

This plan modifies code introduced by `docs/superpowers/plans/2026-07-18-admin-access-restriction.md` (the `onAuthStateChanged`-based `initFirebase()`, the `authUser` login gate). **That plan's Task 1 must be implemented and committed before starting this one** — the "Find" blocks below match its resulting code, not the original anonymous-auth version.

## Global Constraints

- 판단 기준: 수도권 지점은 그 지점 자신의 월평균, 지방 지점은 지방 전체(12개 지점) 합산 월평균 대비로 판단한다 (지방은 중간 허브 도착 시각만 기록되므로 개별 지점 이력이 아니라 지방 전체를 하나의 공정으로 본다).
- 지연 임계치: 평균 대비 +15분 이상.
- 표시 범위: 운송중(진행중) 항목과 완료된 항목 모두 — 대기 항목은 표시 없음.
- 이력이 없어 평균을 낼 수 없는 지점/지역은 지연 여부를 판단하지 않는다(뱃지 없이 소요분만, 그마저 없으면 미표시).
- 이 프로젝트에는 테스트 스위트가 없으므로([CLAUDE.md](../../../CLAUDE.md)) 검증은 브라우저 수동 확인이다.
- 새 CSS는 최소한으로 — 기존 `.stats-note`(지방 안내문구용)를 재사용하고, 뱃지 하나(`.badge.delay`)만 새로 추가한다.

---

### Task 1: Load stats data eagerly on login + start a 1-minute dashboard tick

**Files:**
- Modify: `2_관리자대시보드(admin).html`

**Interfaces:**
- Produces: module-level `let dashboardTimer = null;`. Task 2/3 rely on `statsEntries` being populated as soon as the dashboard is visible (not only after visiting the stats page), and on `render()` being called every 60s while logged in.

- [ ] **Step 1: Capture the current baseline**

Open `2_관리자대시보드(admin).html` in a browser and log in (per the admin-access-restriction feature). Confirm the dashboard loads. Note that today, `statsEntries` stays empty until you click "소요시간 통계" — this task changes that.

- [ ] **Step 2: Add the timer variable**

Find (this line was added by the admin-access-restriction plan, near the other `let unsub...`/auth state declarations):

```js
let authUser = null;         // 로그인한 Firebase Auth 사용자 (관리자)
let authError = null;        // 로그인 실패 메시지
let loggingIn = false;       // 로그인 요청 진행 중 여부
```

Add immediately after it:

```js
let dashboardTimer = null;   // 대시보드용 1분 틱 (진행중 항목 경과시간 갱신)
```

- [ ] **Step 3: Eagerly subscribe to stats and start the tick on login**

Find the `onAuthStateChanged` callback (added by the admin-access-restriction plan):

```js
    firebase.auth().onAuthStateChanged((user) => {
      authUser = user;
      live = !!user;
      if(user){
        subscribeDashboard();
      }else{
        if(unsubDashboard){ unsubDashboard(); unsubDashboard = null; }
        if(unsubStats){ unsubStats(); unsubStats = null; }
        statsSubscribing = false;
        dashboardEntries = [];
        statsEntries = [];
        view = "dashboard";
      }
      render();
    });
```

Replace with:

```js
    firebase.auth().onAuthStateChanged((user) => {
      authUser = user;
      live = !!user;
      if(user){
        subscribeDashboard();
        subscribeStats();
        if(!dashboardTimer){
          dashboardTimer = setInterval(() => render(), 60000);
        }
      }else{
        if(unsubDashboard){ unsubDashboard(); unsubDashboard = null; }
        if(unsubStats){ unsubStats(); unsubStats = null; }
        statsSubscribing = false;
        dashboardEntries = [];
        statsEntries = [];
        view = "dashboard";
        if(dashboardTimer){ clearInterval(dashboardTimer); dashboardTimer = null; }
      }
      render();
    });
```

- [ ] **Step 4: Verify in a browser**

Reload `2_관리자대시보드(admin).html`, log in, open DevTools console, and type `statsEntries.length` — it should be a number (possibly `0` if there's no real data yet, but the variable should already be populated/attempted without visiting the stats page first). Log out and confirm no JS errors are thrown (the `clearInterval` guard should handle a null timer safely — it won't be null here, but confirm no console errors either way).

- [ ] **Step 5: Commit**

```bash
git add "2_관리자대시보드(admin).html"
git commit -m "$(cat <<'EOF'
Load stats data eagerly on admin login and add dashboard tick

Both dashboard and stats now share one 30-day subscription, and a
60s timer keeps in-progress elapsed-time displays advancing.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: Add delay-judgment helper functions

**Files:**
- Modify: `2_관리자대시보드(admin).html`

**Interfaces:**
- Consumes: `statsEntries` (array of `{store, region, date, startAt, endAt}`, populated by Task 1), `minutesBetween(a, b)` (existing function, returns rounded minutes between two `Date`s).
- Produces: `elapsedMinutes(entry)`, `expectedMinutes(store, region)`, `isDelayed(entry)` — consumed by Task 3's row rendering. `entry` here means an object with `{store, region, startAt: Date|null, endAt: Date|null}`, matching `dashboardEntries`' shape.

- [ ] **Step 1: Write the three helper functions**

Find:

```js
function findEntry(store){
  return dashboardEntries.find(e => e.store === store) || null;
}
```

Add immediately after it:

```js
function elapsedMinutes(entry){
  if(!entry.startAt) return null;
  const endTime = entry.endAt || new Date();
  return minutesBetween(entry.startAt, endTime);
}

function expectedMinutes(store, region){
  const completed = region === "지방"
    ? statsEntries.filter(e => e.region === "지방" && e.startAt && e.endAt)
    : statsEntries.filter(e => e.store === store && e.startAt && e.endAt);
  if(!completed.length) return null;
  return Math.round(completed.reduce((sum, e) => sum + minutesBetween(e.startAt, e.endAt), 0) / completed.length);
}

function isDelayed(entry){
  const elapsed = elapsedMinutes(entry);
  if(elapsed === null) return null;
  const expected = expectedMinutes(entry.store, entry.region);
  if(expected === null) return null;
  return (elapsed - expected) >= 15;
}
```

- [ ] **Step 2: Verify with the browser console**

Reload `2_관리자대시보드(admin).html`, log in, open DevTools console, and run:

```js
elapsedMinutes({ startAt: new Date(Date.now() - 20*60000), endAt: null })
// expect: 20 (approximately)

expectedMinutes("존재하지않는지점", "수도권")
// expect: null (no history for a nonexistent store)

isDelayed({ store: "존재하지않는지점", region: "수도권", startAt: new Date(Date.now() - 20*60000), endAt: null })
// expect: null (can't judge — no baseline)
```

- [ ] **Step 3: Commit**

```bash
git add "2_관리자대시보드(admin).html"
git commit -m "$(cat <<'EOF'
Add elapsed-time and delay-judgment helpers to admin dashboard

expectedMinutes() compares metro stores against their own monthly
average and rural stores against the combined rural average, since
rural times only reflect hub arrival, not per-store delivery.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 3: Display elapsed/duration + delay badge in dashboard rows

**Files:**
- Modify: `2_관리자대시보드(admin).html`

**Interfaces:**
- Consumes: `elapsedMinutes(entry)`, `isDelayed(entry)` from Task 2; `findEntry(store)` (existing).

- [ ] **Step 1: Add the delay badge CSS**

Find:

```css
.legend-row{display:flex; gap:16px; margin:14px 0 10px; font-size:12px; color:var(--gray); align-items:center;}
.legend-row .dot{width:9px; height:9px; border-radius:50%; display:inline-block; margin-right:5px;}
```

Add immediately after it:

```css
.badge.delay{
  font-family:'JetBrains Mono',monospace; font-size:9.5px; font-weight:700;
  padding:2px 7px; border-radius:10px; background:#FBEAEA; color:var(--red); margin-left:6px;
}
```

- [ ] **Step 2: Widen the row grid to fit a 5th column**

Find:

```css
.row-head{display:grid; grid-template-columns:20px 1fr 40px 40px; font-size:11px; color:var(--gray); padding:0 0 6px; border-bottom:1px solid var(--line); align-items:center;}
.row{display:grid; grid-template-columns:20px 1fr 40px 40px; align-items:center; padding:8px 0; border-bottom:1px solid var(--line);}
```

Replace with:

```css
.row-head{display:grid; grid-template-columns:20px 1fr 40px 40px 74px; font-size:11px; color:var(--gray); padding:0 0 6px; border-bottom:1px solid var(--line); align-items:center;}
.row{display:grid; grid-template-columns:20px 1fr 40px 40px 74px; align-items:center; padding:8px 0; border-bottom:1px solid var(--line);}
```

- [ ] **Step 3: Add the rural note and a 5th header cell in `regionCard()`**

Find:

```js
function regionCard(label, storeList, isRural){
  const doneN = storeList.filter(n => { const e = findEntry(n); return e && e.endAt; }).length;
  return `
    <div class="region-card">
      <div class="region-head">
        <span class="name">${label}</span>
        <span class="ratio">${doneN} / ${storeList.length} 완료</span>
      </div>
      <div class="row-head">
        <span></span><span>지점</span>
        ${iconStart(14)}
        ${isRural ? iconWarehouse(14) : iconDone(14)}
      </div>
```

Replace with:

```js
function regionCard(label, storeList, isRural){
  const doneN = storeList.filter(n => { const e = findEntry(n); return e && e.endAt; }).length;
  return `
    <div class="region-card">
      <div class="region-head">
        <span class="name">${label}</span>
        <span class="ratio">${doneN} / ${storeList.length} 완료</span>
      </div>
      ${isRural ? `<div class="stats-note">*지방점 소요시간은 중간 허브 도착 기준이며, 지연 판정은 지방 전체 평균 대비로 계산됩니다</div>` : ""}
      <div class="row-head">
        <span></span><span>지점</span>
        ${iconStart(14)}
        ${isRural ? iconWarehouse(14) : iconDone(14)}
        <span>소요</span>
      </div>
```

- [ ] **Step 4: Render the elapsed/duration cell + badge per row**

Find (still inside `regionCard()`):

```js
      ${storeList.map(name => {
        const e = findEntry(name);
        const status = !e ? "wait" : (e.endAt ? "done" : "going");
        const startTxt = e && e.startAt ? fmtTime(e.startAt) : "--:--";
        const endTxt = e && e.endAt ? fmtTime(e.endAt) : "--:--";
        return `
          <div class="row">
            <span class="status-dot ${status}"></span>
            <span class="name">${name}</span>
            <span class="time ${status === "wait" ? "" : "going"}">${startTxt}</span>
            <span class="time ${status === "done" ? "done" : ""}">${endTxt}</span>
          </div>
        `;
      }).join("")}
```

Replace with:

```js
      ${storeList.map(name => {
        const e = findEntry(name);
        const status = !e ? "wait" : (e.endAt ? "done" : "going");
        const startTxt = e && e.startAt ? fmtTime(e.startAt) : "--:--";
        const endTxt = e && e.endAt ? fmtTime(e.endAt) : "--:--";
        const elapsed = e ? elapsedMinutes(e) : null;
        const delayed = e ? isDelayed(e) : null;
        const elapsedTxt = elapsed === null ? "" : (elapsed + "분" + (status === "going" ? "째" : ""));
        return `
          <div class="row">
            <span class="status-dot ${status}"></span>
            <span class="name">${name}</span>
            <span class="time ${status === "wait" ? "" : "going"}">${startTxt}</span>
            <span class="time ${status === "done" ? "done" : ""}">${endTxt}</span>
            <span class="time">${elapsedTxt}${delayed ? `<span class="badge delay">지연</span>` : ""}</span>
          </div>
        `;
      }).join("")}
```

- [ ] **Step 5: Verify in a browser**

Log in and open the dashboard. Confirm:
- 지방 카드 상단에 안내문구("*지방점 소요시간은...")가 보인다.
- 대기 중인 지점은 5번째 칸이 비어 있다.
- (실제 Firestore에 데이터가 있다면) 진행중 항목은 "OO분째"가 보이고, 60초 후 자동으로 숫자가 올라간다.
- 이력이 없는 지점/지역은 지연 뱃지 없이 소요분만(또는 아무것도) 보인다 — 콘솔에 에러가 없어야 한다.

- [ ] **Step 6: Commit**

```bash
git add "2_관리자대시보드(admin).html"
git commit -m "$(cat <<'EOF'
Show elapsed/duration and delay badge on admin dashboard rows

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 4: Update CLAUDE.md's realtime-subscriptions note

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update the "Realtime subscriptions" paragraph**

Find:

```
**Realtime subscriptions.** Firestore `onSnapshot` listeners drive re-renders on remote changes (`subscribeToday` in index.html; `subscribeDashboard` and `subscribeStats` in admin.html). admin.html's stats view fetches the full 30-day window once (`statsSubscribing` guard prevents re-subscribing) and recomputes date comparisons client-side on selection change, rather than re-querying.
```

Replace with:

```
**Realtime subscriptions.** Firestore `onSnapshot` listeners drive re-renders on remote changes (`subscribeToday` in index.html; `subscribeDashboard` and `subscribeStats` in admin.html). Both start together as soon as the admin logs in — the dashboard's per-store delay indicator and the stats page's monthly averages read from the same 30-day `statsEntries` dataset, fetched once (`statsSubscribing` guard prevents re-subscribing when the stats page is later opened). Date-comparison recomputation on the stats page still happens client-side on selection change, not by re-querying. A 60-second `dashboardTimer` re-renders the dashboard so in-progress elapsed-time displays keep advancing between Firestore updates.
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "$(cat <<'EOF'
Document eager stats loading and dashboard tick in CLAUDE.md

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```
