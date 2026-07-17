# 관리자 페이지 접근 제한 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Gate `2_관리자대시보드(admin).html` behind a Firebase email/password login, and add Firestore security rules so historical delivery data is actually protected server-side (not just hidden by a UI gate).

**Architecture:** Replace admin.html's anonymous sign-in with `onAuthStateChanged`-driven login screen (`signInWithEmailAndPassword` against one shared admin account). Firestore rules split reads: any authenticated client (including the driver app's anonymous session) may `list` recent (~24h) documents; only the admin identity may `list` older/historical documents. `create`/`update` stay open to any authenticated client, unchanged. `1_입력시스템(index).html` (driver app) is not modified.

**Tech Stack:** Vanilla JS, Firebase compat SDK (already loaded via CDN `<script>` tags), Firebase Authentication (email/password), Firestore Security Rules. No build tools, no test framework, no new npm dependencies — consistent with the rest of this repo.

## Global Constraints

- No backend server is introduced — Firebase Auth and Firestore Rules are both serverless managed features (see spec "결정한 접근 방식").
- `1_입력시스템(index).html` keeps using anonymous auth; it is out of scope and must not be modified.
- There is no test suite in this project (see [CLAUDE.md](../../../CLAUDE.md)); verification is manual (browser + Firebase Console's Rules Playground) throughout this plan.
- `ADMIN_EMAIL` is a plain constant in admin.html, not a secret — the actual password is never stored in code; it is set once in the Firebase Console when the admin account is created.
- New UI should reuse admin.html's existing visual language (`--paper`/`--green` palette tokens); copy `.form-card`/`.field`/`.save-btn` styling from index.html rather than inventing a new look (see spec section 1, "스타일 재사용").
- Exact Firestore rule syntax was intentionally deferred by the design spec to this implementation plan (spec section 2) — Task 2 below is where that syntax is finalized.

---

### Task 1: Add Firebase Auth login/logout gate to admin.html

**Files:**
- Modify: `2_관리자대시보드(admin).html`

**Interfaces:**
- Produces: module-level state `authUser`, `authError`, `loggingIn`; functions `login(password)`, `logout()`, `renderLogin()`. Task 2 relies on the `ADMIN_EMAIL` constant this task introduces having the *same literal value* as the one hardcoded into `firestore.rules`.

- [ ] **Step 1: Capture the current (pre-change) behavior as a baseline**

Open `2_관리자대시보드(admin).html` directly in a browser (double-click it, or drag it into a browser window). Confirm the dashboard renders immediately with no login prompt — this is the gap this task closes. (The Firebase calls will fail with an `auth/api-key-not-valid` error banner since `firebaseConfig` is still a placeholder; that's expected and unrelated to this change.)

- [ ] **Step 2: Add the missing CSS classes (copied from index.html) plus one new input style**

In `2_관리자대시보드(admin).html`, find this existing block (around line 50-54):

```css
.err-banner{
  background:#FBEAEA; border:1.5px solid var(--red); color:#7A2626; border-radius:10px;
  padding:12px 14px; font-size:12.5px; margin-top:14px; line-height:1.6;
}

.stat-grid{display:grid; grid-template-columns:repeat(4,1fr); gap:12px; margin-top:16px;}
```

Insert the following between the `.err-banner` block and `.stat-grid` (these three class names — `.form-card`, `.field`, `.save-btn` — exist in index.html but not admin.html; copying them here keeps both apps visually consistent without inventing new style):

```css
.form-card{
  background:var(--white); border:1.5px solid var(--line); border-radius:14px; padding:16px;
  margin-top:16px;
}
.form-card h2{font-size:14.5px; font-weight:700; color:var(--green); margin:0 0 12px;}
.field{margin-bottom:14px;}
.field label{display:block; font-size:11px; color:var(--gray); font-weight:700; margin-bottom:6px;}
.field input{
  width:100%; padding:10px 12px; border:1.5px solid var(--line); border-radius:8px;
  font-size:14px; font-family:'Noto Sans KR',sans-serif; background:var(--white); color:var(--ink);
}
.save-btn{
  width:100%; padding:14px 0; border:none; border-radius:12px; background:var(--amber); color:#fff;
  font-weight:700; font-size:15px; cursor:pointer; margin-top:2px;
}
.save-btn:disabled{background:var(--paper-2); color:var(--gray); cursor:not-allowed;}
```

(`.field input` is new — neither HTML file has a text `<input>` today, so there's nothing to copy for it. It reuses the same `--line`/`--white`/`--ink` tokens as everything else.)

- [ ] **Step 3: Add `ADMIN_EMAIL` constant**

Find (around line 117-126):

```js
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_PROJECT_ID.appspot.com",
  messagingSenderId: "YOUR_SENDER_ID",
  appId: "YOUR_APP_ID"
};
const COLLECTION = "deliveries";
/* ============================================================= */
```

Replace with:

```js
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_PROJECT_ID.appspot.com",
  messagingSenderId: "YOUR_SENDER_ID",
  appId: "YOUR_APP_ID"
};
const COLLECTION = "deliveries";
// 관리자 로그인 계정 식별자 (비밀 아님 — 비밀번호는 Firebase 콘솔에서만 설정됨).
// firestore.rules의 ADMIN_EMAIL 값과 반드시 동일해야 함.
const ADMIN_EMAIL = "admin@yourproject.local";
/* ============================================================= */
```

- [ ] **Step 4: Add auth state variables**

Find (around line 173):

```js
let selectedDateIdx = 0;     // stats 페이지 날짜 선택 (0 = 오늘)
```

Add immediately after it:

```js
let authUser = null;         // 로그인한 Firebase Auth 사용자 (관리자)
let authError = null;        // 로그인 실패 메시지
let loggingIn = false;       // 로그인 요청 진행 중 여부
```

- [ ] **Step 5: Replace `initFirebase()` and add `login`/`logout`**

Find (around line 175-187):

```js
async function initFirebase(){
  try{
    firebase.initializeApp(firebaseConfig);
    db = firebase.firestore();
    await firebase.auth().signInAnonymously();
    subscribeDashboard();
  }catch(e){
    console.error("Firebase 초기화 실패", e);
    loadError = "Firebase 연결에 실패했습니다. firebaseConfig 값이 올바른지 확인해 주세요. (" + (e && e.message ? e.message : e) + ")";
    live = false;
    render();
  }
}
```

Replace with:

```js
async function initFirebase(){
  try{
    firebase.initializeApp(firebaseConfig);
    db = firebase.firestore();
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
  }catch(e){
    console.error("Firebase 초기화 실패", e);
    loadError = "Firebase 연결에 실패했습니다. firebaseConfig 값이 올바른지 확인해 주세요. (" + (e && e.message ? e.message : e) + ")";
    live = false;
    render();
  }
}

async function login(password){
  loggingIn = true; authError = null; render();
  try{
    await firebase.auth().signInWithEmailAndPassword(ADMIN_EMAIL, password);
  }catch(e){
    console.error("로그인 실패", e);
    authError = "비밀번호가 올바르지 않습니다.";
  }
  loggingIn = false; render();
}

async function logout(){
  await firebase.auth().signOut();
}
```

- [ ] **Step 6: Add `renderLogin()`**

Find `function renderDashboard(){` (around line 265) and insert this new function immediately before it:

```js
function renderLogin(){
  return `
    <div class="topbar">
      <div class="topbar-row">
        <div class="brand">
          <div class="eyebrow">${TODAY}</div>
          <div class="title">배송 관리자 대시보드</div>
        </div>
      </div>
    </div>
    ${authError ? `<div class="err-banner">${authError}</div>` : ""}
    <form class="form-card" id="loginForm">
      <h2>관리자 로그인</h2>
      <div class="field">
        <label>비밀번호</label>
        <input type="password" id="adminPassword" placeholder="비밀번호 입력" required>
      </div>
      <button type="submit" class="save-btn" ${loggingIn ? "disabled" : ""}>${loggingIn ? "로그인 중..." : "로그인"}</button>
    </form>
  `;
}
```

- [ ] **Step 7: Gate `render()` on auth state**

Find (around line 249-253):

```js
function render(){
  const app = document.getElementById("app");
  app.innerHTML = view === "dashboard" ? renderDashboard() : renderStats();
  wire();
}
```

Replace with:

```js
function render(){
  const app = document.getElementById("app");
  app.innerHTML = !authUser ? renderLogin() : (view === "dashboard" ? renderDashboard() : renderStats());
  wire();
}
```

- [ ] **Step 8: Add a logout button to both views**

In `renderDashboard()`, find (around line 267-274):

```js
    <div class="topbar">
      <div class="topbar-row">
        <div class="brand">
          <div class="eyebrow">${TODAY}</div>
          <div class="title">배송 관리자 대시보드</div>
        </div>
        <button class="chip-btn" id="statsBtn">${iconChart(14)}소요시간 통계</button>
      </div>
```

Replace the button line with a small button group:

```js
    <div class="topbar">
      <div class="topbar-row">
        <div class="brand">
          <div class="eyebrow">${TODAY}</div>
          <div class="title">배송 관리자 대시보드</div>
        </div>
        <div style="display:flex; gap:8px;">
          <button class="chip-btn" id="statsBtn">${iconChart(14)}소요시간 통계</button>
          <button class="chip-btn outline" id="logoutBtn">로그아웃</button>
        </div>
      </div>
```

In `renderStats()`, find (around line 342-347):

```js
        <div class="brand">
          <div class="eyebrow">지점별 소요시간</div>
          <div class="title">운송 소요시간 통계</div>
        </div>
        <button class="chip-btn outline" id="backBtn">← 대시보드로</button>
      </div>
```

Replace with:

```js
        <div class="brand">
          <div class="eyebrow">지점별 소요시간</div>
          <div class="title">운송 소요시간 통계</div>
        </div>
        <div style="display:flex; gap:8px;">
          <button class="chip-btn outline" id="backBtn">← 대시보드로</button>
          <button class="chip-btn outline" id="logoutBtn">로그아웃</button>
        </div>
      </div>
```

- [ ] **Step 9: Wire the login form and logout button**

Find `function wire(){` (around line 416) and add these two blocks inside it (order doesn't matter relative to the existing `statsBtn`/`backBtn`/`dateSel` blocks):

```js
  const loginForm = document.getElementById("loginForm");
  if(loginForm){
    loginForm.addEventListener("submit", (e) => {
      e.preventDefault();
      const pw = document.getElementById("adminPassword").value;
      if(pw) login(pw);
    });
  }
  const logoutBtn = document.getElementById("logoutBtn");
  if(logoutBtn){
    logoutBtn.addEventListener("click", () => { logout(); });
  }
```

- [ ] **Step 10: Verify in a browser**

Open `2_관리자대시보드(admin).html` directly in a browser. Confirm:
- A login screen appears (password field + "로그인" button) instead of the dashboard — this proves the gate works even before a real Firebase project exists, because `onAuthStateChanged` reports `null` immediately with no persisted session.
- Typing a password and submitting shows a spinner state ("로그인 중...") then an error banner ("비밀번호가 올바르지 않습니다"), since `firebaseConfig` is still a placeholder — this is expected until Task 4.
- No new console errors other than the pre-existing `auth/api-key-not-valid` one from Step 1.

- [ ] **Step 11: Commit**

```bash
git add "2_관리자대시보드(admin).html"
git commit -m "$(cat <<'EOF'
Gate admin dashboard behind Firebase email/password login

Replaces anonymous sign-in with an onAuthStateChanged-driven login
screen so the UI itself no longer opens the dashboard to anyone.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: Add Firestore security rules

**Files:**
- Create: `firestore.rules`

**Interfaces:**
- Consumes: `ADMIN_EMAIL` value from Task 1 (`admin@yourproject.local`) — must match exactly.
- Produces: rule logic that Task 4's manual verification exercises.

**Design note (read before writing the rule):** the `deliveries` collection has no field that means "today" from Firestore's point of view — `date` is a plain client-formatted string. Rather than trying to recompute "today" inside a security rule (fragile — Firestore's rules language has no reliable string-date formatting), this task uses the existing `createdAt` server timestamp and a rolling 24-hour window as a practical stand-in for "recent/today's data," which is what both apps actually need read access to. Anything older than 24 hours old is only readable by the authenticated admin. This is a deliberate simplification (rolling window vs. calendar-day boundary) — flagged here rather than left implicit.

- [ ] **Step 1: Write `firestore.rules`**

Create `firestore.rules` at the project root:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /deliveries/{deliveryId} {
      // 기사 앱(index.html)과 관리자 로그인 상태 모두 create/update 가능 (기존 동작 유지)
      allow create, update: if request.auth != null;

      // list(쿼리) 읽기:
      //  - 관리자(ADMIN_EMAIL로 로그인)는 전체 기간 조회 가능 (30일 통계용)
      //  - 그 외 인증된 클라이언트(기사 앱의 익명 세션 포함)는 최근 24시간 이내
      //    생성된 문서만 조회 가능 — 오늘 등록 목록을 보여주는 데는 충분하고,
      //    30일치 range 쿼리는 이 조건을 통과하지 못해 전체 요청이 거부됨.
      allow list: if request.auth != null &&
        (
          request.auth.token.email == "admin@yourproject.local"
          || resource.data.createdAt > request.time - duration.value(1, 'd')
        );
    }
  }
}
```

- [ ] **Step 2: Verify with Firebase Console's Rules Playground**

This requires a real Firebase project (see Task 4 for full one-time setup). Once the project exists and this file's contents have been pasted into **Firebase Console → Firestore Database → Rules**, use **Rules Playground** (same tab) to simulate:

1. Simulation type `list`, path `/deliveries`, authenticated: yes, provider: anonymous → expect **거부(deny)** if you set a query constraint representing a 30+ day-old `createdAt`, and **허용(allow)** for a `createdAt` within the last 24 hours.
2. Simulation type `list`, path `/deliveries`, authenticated: yes, custom auth token with `email` claim set to `admin@yourproject.local` → expect **허용(allow)** regardless of `createdAt`.
3. Simulation type `create`, path `/deliveries/test123`, authenticated: yes (anonymous) → expect **허용(allow)**.

- [ ] **Step 3: Commit**

```bash
git add firestore.rules
git commit -m "$(cat <<'EOF'
Add Firestore security rules for admin-only historical reads

Anonymous/driver sessions keep read access to recent (~24h) documents;
older documents (used by the 30-day stats query) require the admin
email identity. Reference file only — must be pasted into the Firebase
Console manually, same as the firebaseConfig placeholder.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 3: Document the new manual setup step in CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Extend the "Firebase config is a placeholder" note**

Find this paragraph in `CLAUDE.md`:

```
**Firebase config is a placeholder.** Both files ship with `firebaseConfig` containing `YOUR_API_KEY` etc. — a real Firebase project's config must be pasted into both files identically before either app functions (see 향후 계획 in the plan doc).
```

Replace with:

```
**Firebase config is a placeholder.** Both files ship with `firebaseConfig` containing `YOUR_API_KEY` etc. — a real Firebase project's config must be pasted into both files identically before either app functions (see 향후 계획 in the plan doc).

**Admin access requires manual one-time setup.** admin.html gates the dashboard behind `firebase.auth().signInWithEmailAndPassword()` using the `ADMIN_EMAIL` constant defined in admin.html. Two manual steps in the Firebase Console are required before this works, neither of which is scripted in this repo: (1) create that exact email/password user under Authentication → Users, and (2) paste [`firestore.rules`](firestore.rules)'s contents into Firestore Database → Rules. `ADMIN_EMAIL` in admin.html and the email hardcoded in `firestore.rules` must match exactly.
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "$(cat <<'EOF'
Document admin auth + Firestore rules manual setup in CLAUDE.md

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 4: End-to-end manual verification against a real Firebase project

**Files:** none (verification only — no code changes)

This task assumes a real Firebase project has been created and its config pasted into both HTML files (the pre-existing, already-documented placeholder step — see CLAUDE.md). If that hasn't happened yet, do it first; it's a prerequisite unrelated to this feature.

- [ ] **Step 1: Create the admin account**

Firebase Console → Authentication → Users → Add user. Email: the exact value used for `ADMIN_EMAIL` in admin.html (e.g. `admin@yourproject.local`). Set a real password.

- [ ] **Step 2: Enable Email/Password sign-in method**

Firebase Console → Authentication → Sign-in method → enable "Email/Password" (it's off by default; only Anonymous is likely enabled already since index.html uses it).

- [ ] **Step 3: Publish the Firestore rules**

Firebase Console → Firestore Database → Rules → replace contents with `firestore.rules` from this repo → Publish.

- [ ] **Step 4: Verify correct login**

Open `2_관리자대시보드(admin).html`, enter the admin password from Step 1 → dashboard should load with today's per-store status.

- [ ] **Step 5: Verify wrong password is rejected**

Log out, enter an incorrect password → confirm "비밀번호가 올바르지 않습니다" error banner, dashboard does not load.

- [ ] **Step 6: Verify logout**

While logged in, click "로그아웃" → confirm the login screen reappears.

- [ ] **Step 7: Verify the driver app is unaffected**

Open `1_입력시스템(index).html` (no login involved) → confirm it still registers a test delivery and shows it in the list, exactly as before this change.

- [ ] **Step 8: Verify historical data is actually protected**

Open a fresh, unauthenticated browser tab's DevTools console on `2_관리자대시보드(admin).html`'s origin, without logging in through the UI. Since anonymous sign-in is self-service, this proves whether the *rule* — not the UI — is what's protecting the data:

```js
firebase.auth().signInAnonymously().then(() => {
  return firebase.firestore().collection("deliveries")
    .where("date", ">=", "2026-06-01") // any date older than 24h
    .get();
}).then(snap => console.log("rows:", snap.size))
  .catch(err => console.error("DENIED:", err.code));
```

Expected: `DENIED: permission-denied`. If this instead returns rows, the rule from Task 2 is not correctly published — stop and re-check Step 3 before considering this feature done.

- [ ] **Step 9: Verify the stats page works for the logged-in admin**

While logged in as admin, open the "소요시간 통계" page and change the 조회일 dropdown a few times → confirm data loads without errors (this exercises the same 30-day range query as Step 8, now correctly authorized).
