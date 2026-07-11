# Group A Critical Bugfixes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix four game-breaking issues in "จัดตั้ง ออนไลน์" — no penalty for negative treasury, election calc stuck on host-only automation, no way out of a deadlocked coalition negotiation, and hardcode the term length to 16 turns — per [docs/superpowers/specs/2026-07-12-group-a-critical-bugfixes-design.md](../specs/2026-07-12-group-a-critical-bugfixes-design.md).

**Architecture:** Everything lives in one file, `index (3).html` — a static page with inline `<style>`/`<script>`, no build step, no bundler, no framework. Firestore (`jadtang_rooms` collection, one doc per room) is the only "server." All game logic runs client-side; whichever browser tab is `hostId` currently drives AI-controlled turns via `runHostAutomation()`. This plan keeps that architecture as-is and only adds/modifies functions within the existing `<script>` block.

**Tech Stack:** Vanilla JS (no modules, no TypeScript), Firebase JS SDK 10.7.1 (compat), Firestore. **No automated test framework exists in this project** — it's a single HTML file with no `package.json`. Pure-logic functions (no DOM/Firestore dependency) are verified with standalone `node -e` snippets before being wired into the file; everything that touches Firestore or the DOM is verified by hand in a browser against a real Firestore project.

## Global Constraints

- Do not introduce a build step, bundler, package.json, or any new dependency — the file stays a single static HTML file.
- All new user-facing strings must be in Thai, matching the existing tone (formal, bureaucratic-document style — see existing copy like `"รอผู้เล่นคนอื่นยืนยันนโยบาย"`).
- Reuse the existing `clamp(v,min=0,max=100)` helper (defined near the top of the `<script>` block) for any value that must stay in a 0–100 range — do not hand-roll new clamping.
- Function names introduced across tasks must match exactly (a later task consumes what an earlier task produces): `computeDeficitCrisis`, `resetRoomToLobby`, `nextPmCandidate`, `restartElectionSameTerm`, `concedePmCandidacy`.
- New room-state fields: `deficitStreak` (number, default `0`), `concededPm` (array of slot-id strings, default `[]`). Every code path that can start a fresh campaign/election (`handleCreateRoom`, `resetRoomToLobby`, `startNewTerm`, `restartElectionSameTerm`) must reset `deficitStreak` and/or `concededPm` per the spec — see each task below for exactly which.
- Exact formulas (from the approved spec, do not change): `trustPenalty = clamp(3 + streak*2, 0, 20)`, `unrestPenalty = clamp(2 + streak*1.5, 0, 15)`.
- `turnsPerTerm` changes from `8` to `16` everywhere a fresh room/term is initialized.

---

### Task 1: Room-creation defaults — 16 turns, `deficitStreak`, `concededPm`

**Files:**
- Modify: `index (3).html` — the `handleCreateRoom()` function

**Interfaces:**
- Produces: room documents now always contain `turnsPerTerm:16`, `deficitStreak:0`, `concededPm:[]` from the moment they're created. Later tasks (2–9) assume these fields exist on every room doc they read.

- [ ] **Step 1: Locate and replace the `roomData` object literal**

Find this exact block inside `handleCreateRoom()`:

```javascript
    turn:1, treasury:200, national:{economy:50,trust:50,unrest:20},
    popularity:Object.fromEntries(slots.map(s=>[s.id,{...emptyPop}])),
    oppositionActions:{}, queuedEvent:null,
    lastElection:null, pm:null, coalitionSlots:[], invitations:{}, cabinet:{}, coalitionParties:[],
    turn:1, turnsPerTerm:8, usedEventIds:[], currentEvent:null, currentEventOwnerSlot:null, currentEffectResult:null,
    collapsed:false, electionLock:false, pendingCollapse:false,
```

Replace with:

```javascript
    turn:1, treasury:200, national:{economy:50,trust:50,unrest:20},
    popularity:Object.fromEntries(slots.map(s=>[s.id,{...emptyPop}])),
    oppositionActions:{}, queuedEvent:null,
    lastElection:null, pm:null, coalitionSlots:[], invitations:{}, cabinet:{}, coalitionParties:[],
    turn:1, turnsPerTerm:16, usedEventIds:[], currentEvent:null, currentEventOwnerSlot:null, currentEffectResult:null,
    collapsed:false, electionLock:false, pendingCollapse:false, deficitStreak:0, concededPm:[],
```

(Note: the source file already has a duplicate `turn:1,` on two lines — leave that as-is, it's harmless pre-existing redundancy not in scope for this plan.)

- [ ] **Step 2: Manually verify**

Start a local server and open the file:

```bash
cd "/Users/panupanpitak/Projects/judtang-online" && python3 -m http.server 8765
```

Open `http://localhost:8765/index%20(3).html` in a browser, enter a name, click "+ สร้างห้องใหม่". Open browser devtools console and run:

```javascript
db.collection('jadtang_rooms').doc(ui.roomCode).get().then(s=>console.log(s.data()))
```

Expected: logged object contains `turnsPerTerm: 16`, `deficitStreak: 0`, `concededPm: []`.

- [ ] **Step 3: Commit**

```bash
git add "index (3).html"
git commit -m "Default new rooms to 16 turns per term and add deficitStreak/concededPm fields"
```

---

### Task 2: Pure function — `computeDeficitCrisis`

**Files:**
- Modify: `index (3).html` — add near the existing helpers (`clamp`, `fmt`, `genRoomCode`) below the `/* ---------- helpers ---------- */` comment

**Interfaces:**
- Consumes: `clamp(v,min=0,max=100)` (already defined in the same helpers block)
- Produces: `computeDeficitCrisis(treasury, deficitStreak, national) -> {deficitStreak: number, national: {economy, trust, unrest}}`. Task 3 calls this.

- [ ] **Step 1: Write and run a standalone Node check of the intended logic (fails — function doesn't exist yet)**

```bash
node -e "
function clamp(v,min=0,max=100){return Math.max(min,Math.min(max,v));}
console.log(typeof computeDeficitCrisis);
"
```

Expected: prints `undefined` — confirming the function doesn't exist yet.

- [ ] **Step 2: Validate the logic in isolation with Node before touching the HTML file**

```bash
node -e "
function clamp(v,min=0,max=100){return Math.max(min,Math.min(max,v));}
function computeDeficitCrisis(treasury,deficitStreak,national){
  if(treasury<0){
    const streak=deficitStreak+1;
    const trustPenalty=clamp(3+streak*2,0,20);
    const unrestPenalty=clamp(2+streak*1.5,0,15);
    return {
      deficitStreak:streak,
      national:{
        economy:national.economy,
        trust:clamp(national.trust-trustPenalty),
        unrest:clamp(national.unrest+unrestPenalty),
      },
    };
  }
  return {deficitStreak:0,national:{...national}};
}

// case 1: first deficit turn
let r1=computeDeficitCrisis(-10,0,{economy:50,trust:50,unrest:20});
console.assert(r1.deficitStreak===1,'case1 streak');
console.assert(r1.national.trust===45,'case1 trust: expected 45 got '+r1.national.trust);
console.assert(r1.national.unrest===23.5,'case1 unrest: expected 23.5 got '+r1.national.unrest);

// case 2: third consecutive deficit turn, penalty grows
let r2=computeDeficitCrisis(-5,2,{economy:50,trust:50,unrest:20});
console.assert(r2.deficitStreak===3,'case2 streak');
console.assert(r2.national.trust===41,'case2 trust: expected 41 got '+r2.national.trust);

// case 3: treasury recovers, streak resets
let r3=computeDeficitCrisis(5,4,{economy:50,trust:10,unrest:90});
console.assert(r3.deficitStreak===0,'case3 streak reset');
console.assert(r3.national.trust===10 && r3.national.unrest===90,'case3 national unchanged');

// case 4: penalties never push trust/unrest out of 0-100
let r4=computeDeficitCrisis(-1,50,{economy:50,trust:5,unrest:95});
console.assert(r4.national.trust===0,'case4 trust floor: got '+r4.national.trust);
console.assert(r4.national.unrest===100,'case4 unrest ceiling: got '+r4.national.unrest);

console.log('ALL PASS');
"
```

Expected output: `ALL PASS` with no `console.assert` failure lines above it.

- [ ] **Step 3: Add the exact same function to `index (3).html`**

Find this block:

```javascript
/* ---------- helpers ---------- */
const clamp=(v,min=0,max=100)=>Math.max(min,Math.min(max,v));
const fmt=n=>Math.round(n).toLocaleString('en-US');
```

Replace with:

```javascript
/* ---------- helpers ---------- */
const clamp=(v,min=0,max=100)=>Math.max(min,Math.min(max,v));
const fmt=n=>Math.round(n).toLocaleString('en-US');
function computeDeficitCrisis(treasury,deficitStreak,national){
  if(treasury<0){
    const streak=deficitStreak+1;
    const trustPenalty=clamp(3+streak*2,0,20);
    const unrestPenalty=clamp(2+streak*1.5,0,15);
    return {
      deficitStreak:streak,
      national:{
        economy:national.economy,
        trust:clamp(national.trust-trustPenalty),
        unrest:clamp(national.unrest+unrestPenalty),
      },
    };
  }
  return {deficitStreak:0,national:{...national}};
}
```

- [ ] **Step 4: Confirm the function is present verbatim**

```bash
grep -A16 "function computeDeficitCrisis" "index (3).html"
```

Expected: prints the function body matching Step 3 exactly.

- [ ] **Step 5: Commit**

```bash
git add "index (3).html"
git commit -m "Add computeDeficitCrisis pure function for fiscal-crisis penalties"
```

---

### Task 3: Wire fiscal crisis into `nextTurn()` and reset `deficitStreak` on new terms

**Files:**
- Modify: `index (3).html` — the `nextTurn()` function and the `startNewTerm()` function

**Interfaces:**
- Consumes: `computeDeficitCrisis(treasury, deficitStreak, national)` from Task 2

- [ ] **Step 1: Modify `nextTurn()`**

Find this exact block:

```javascript
function nextTurn(){
  const r=ui.room;
  if(r.pendingCollapse){
    roomRef().update({collapsed:true,phase:'term_summary',treasury:Math.max(0,r.treasury-10)});
    return;
  }
  if(r.turn>=r.turnsPerTerm){
    roomRef().update({collapsed:false,phase:'term_summary'});
    return;
  }
  const nextR={...r,turn:r.turn+1};
  const {event,ownerSlot}=drawEventForRoom(nextR);
  roomRef().update({turn:r.turn+1,currentEvent:event,currentEventOwnerSlot:ownerSlot,currentEffectResult:null,queuedEvent:null});
}
```

Replace with:

```javascript
function nextTurn(){
  const r=ui.room;
  if(r.pendingCollapse){
    roomRef().update({collapsed:true,phase:'term_summary',treasury:Math.max(0,r.treasury-10)});
    return;
  }
  if(r.turn>=r.turnsPerTerm){
    roomRef().update({collapsed:false,phase:'term_summary'});
    return;
  }
  const crisis=computeDeficitCrisis(r.treasury,r.deficitStreak||0,r.national);
  const nextR={...r,turn:r.turn+1,national:crisis.national};
  const {event,ownerSlot}=drawEventForRoom(nextR);
  roomRef().update({turn:r.turn+1,national:crisis.national,deficitStreak:crisis.deficitStreak,currentEvent:event,currentEventOwnerSlot:ownerSlot,currentEffectResult:null,queuedEvent:null});
}
```

- [ ] **Step 2: Reset `deficitStreak` when a new term starts**

Find this exact block:

```javascript
  roomRef().update({phase:'campaign',term:r.term+1,slots,electionLock:false,invitations:{},pendingCollapse:false,oppositionActions:{},queuedEvent:null});
}
```

Replace with:

```javascript
  roomRef().update({phase:'campaign',term:r.term+1,slots,electionLock:false,invitations:{},pendingCollapse:false,oppositionActions:{},queuedEvent:null,deficitStreak:0});
}
```

- [ ] **Step 3: Manually verify in browser**

With the local server from Task 1 running, play through a room as the only human (others become AI): reach `governing` phase, then repeatedly choose the choice with the most negative `treasury` effect (e.g. "วิกฤตราคาพลังงาน" → "ตรึงราคาน้ำมันด้วยงบอุดหนุน", `treasury:-15`) until treasury goes negative, clicking "ดำเนินการต่อ" each time to advance turns.

In devtools console after treasury is negative and you've advanced 2+ turns:

```javascript
db.collection('jadtang_rooms').doc(ui.roomCode).get().then(s=>console.log(s.data().deficitStreak, s.data().national))
```

Expected: `deficitStreak` is `>= 2` and increasing by 1 each turn while treasury stays negative; `national.trust` decreasing and `national.unrest` increasing turn over turn, both stopping at their 0/100 bounds (never negative, never over 100).

- [ ] **Step 4: Commit**

```bash
git add "index (3).html"
git commit -m "Apply accumulating fiscal-crisis penalties each turn treasury stays negative"
```

---

### Task 4: UI — fiscal-crisis badge and red treasury display

**Files:**
- Modify: `index (3).html` — `topbar()` function and `PHASE_TEMPLATES.governing` function

**Interfaces:**
- Consumes: `r.deficitStreak`, `r.treasury` (already on every room doc after Tasks 1 and 3)

- [ ] **Step 1: Red treasury value in the topbar when negative**

Find this exact block:

```javascript
function topbar(){
  const r=ui.room;
  return `<div class="topbar">
    <div class="brand"><div class="room-pill">ห้อง ${r.roomCode}</div></div>
    <div class="stats">
      <div class="stat"><div class="label">งบประมาณแผ่นดิน</div><div class="value">฿${fmt(r.treasury)}ล.</div></div>
      <div class="stat"><div class="label">ความเชื่อมั่น</div><div class="value">${fmt(r.national.trust)}</div></div>
      <div class="stat"><div class="label">ความไม่สงบ</div><div class="value">${fmt(r.national.unrest)}</div></div>
      <button class="btn btn-ghost" style="padding:6px 12px;font-size:12px;" onclick="leaveRoom()">ออกจากห้อง</button>
    </div>
  </div>`;
}
```

Replace with (this also adds the Reset button placeholder wired up fully in Task 5 — adding the markup here now avoids touching this block twice):

```javascript
function topbar(){
  const r=ui.room;
  const treasuryStyle = r.treasury<0 ? ' style="color:var(--bad);"' : '';
  return `<div class="topbar">
    <div class="brand"><div class="room-pill">ห้อง ${r.roomCode}</div></div>
    <div class="stats">
      <div class="stat"><div class="label">งบประมาณแผ่นดิน</div><div class="value"${treasuryStyle}>฿${fmt(r.treasury)}ล.</div></div>
      <div class="stat"><div class="label">ความเชื่อมั่น</div><div class="value">${fmt(r.national.trust)}</div></div>
      <div class="stat"><div class="label">ความไม่สงบ</div><div class="value">${fmt(r.national.unrest)}</div></div>
      ${isHost()?`<button class="btn btn-ghost" style="padding:6px 12px;font-size:12px;" onclick="resetRoomToLobby()">รีเซ็ตเกม</button>`:''}
      <button class="btn btn-ghost" style="padding:6px 12px;font-size:12px;" onclick="leaveRoom()">ออกจากห้อง</button>
    </div>
  </div>`;
}
```

- [ ] **Step 2: Fiscal-crisis badge in the governing screen**

Find this exact block:

```javascript
      <div class="panel-dark"><h3 style="margin-top:0;font-size:14px;">สถานการณ์ประเทศ</h3>
        ${gauge('เศรษฐกิจ',r.national.economy)}
        ${gauge('ความเชื่อมั่นประชาชน',r.national.trust)}
        ${gauge('ความไม่สงบ',r.national.unrest,true)}
      </div>
```

Replace with:

```javascript
      <div class="panel-dark"><h3 style="margin-top:0;font-size:14px;">สถานการณ์ประเทศ</h3>
        ${gauge('เศรษฐกิจ',r.national.economy)}
        ${gauge('ความเชื่อมั่นประชาชน',r.national.trust)}
        ${gauge('ความไม่สงบ',r.national.unrest,true)}
        ${r.deficitStreak>0?`<div class="tag far" style="display:inline-block;margin-top:6px;">⚠️ วิกฤตการคลัง: สะสม ${r.deficitStreak} ไตรมาสติดต่อกัน</div>`:''}
      </div>
```

**Note:** `resetRoomToLobby()` is referenced here but not defined until Task 5 — the app will throw `resetRoomToLobby is not defined` if the reset button is clicked before Task 5 lands. That's expected and acceptable mid-plan (do not skip ahead); it does not affect page load or any other feature since the button's `onclick` string is only evaluated on click.

- [ ] **Step 3: Manually verify**

Reload the page from Task 3's browser tab (treasury already negative, `deficitStreak >= 2`). Confirm:
- The treasury number in the top bar renders in red/dark-red.
- The "สถานการณ์ประเทศ" panel shows a red-tinted badge reading "⚠️ วิกฤตการคลัง: สะสม N ไตรมาสติดต่อกัน" with the correct N.

- [ ] **Step 4: Commit**

```bash
git add "index (3).html"
git commit -m "Show fiscal-crisis warning badge and red treasury display"
```

---

### Task 5: Host-only "Reset Game" button

**Files:**
- Modify: `index (3).html` — add `resetRoomToLobby()` near `startNewTerm()`

**Interfaces:**
- Produces: `resetRoomToLobby()` — called by the button markup added in Task 4, Step 1.

- [ ] **Step 1: Add the function**

Find this exact block:

```javascript
function startNewTerm(){
  const r=ui.room;
  const slots=r.slots.map(s=>{
    if(!s.claimedBy){
      const drifted={...s.policy};
      ['economy','reform','security'].forEach(axis=>{ drifted[axis]=clamp(drifted[axis]+(Math.random()*10-5),-100,100); });
      return {...s,policy:drifted,submitted:false,budget:{north:0,isaan:0,central:0,bangkok:0,south:0}};
    }
    return {...s,submitted:false,budget:{north:0,isaan:0,central:0,bangkok:0,south:0}};
  });
  roomRef().update({phase:'campaign',term:r.term+1,slots,electionLock:false,invitations:{},pendingCollapse:false,oppositionActions:{},queuedEvent:null,deficitStreak:0});
}
```

Replace with (adds `resetRoomToLobby` right after, unchanged `startNewTerm` body):

```javascript
function startNewTerm(){
  const r=ui.room;
  const slots=r.slots.map(s=>{
    if(!s.claimedBy){
      const drifted={...s.policy};
      ['economy','reform','security'].forEach(axis=>{ drifted[axis]=clamp(drifted[axis]+(Math.random()*10-5),-100,100); });
      return {...s,policy:drifted,submitted:false,budget:{north:0,isaan:0,central:0,bangkok:0,south:0}};
    }
    return {...s,submitted:false,budget:{north:0,isaan:0,central:0,bangkok:0,south:0}};
  });
  roomRef().update({phase:'campaign',term:r.term+1,slots,electionLock:false,invitations:{},pendingCollapse:false,oppositionActions:{},queuedEvent:null,deficitStreak:0});
}

function resetRoomToLobby(){
  if(!isHost()) return;
  if(!confirm('รีเซ็ตเกมกลับไปหน้าเลือกพรรค? ความคืบหน้าของรอบนี้จะหายทั้งหมด')) return;
  const r=ui.room;
  const emptyBudget={north:0,isaan:0,central:0,bangkok:0,south:0};
  const slots=r.slots.map((s,i)=>({
    ...s,
    policy:{...PARTY_TEMPLATES[i].policy},
    submitted:false,
    budget:{...emptyBudget},
  }));
  const emptyPop={north:0,isaan:0,central:0,bangkok:0,south:0};
  roomRef().update({
    phase:'lobby', slots, term:1, treasury:200, national:{economy:50,trust:50,unrest:20},
    popularity:Object.fromEntries(slots.map(s=>[s.id,{...emptyPop}])),
    oppositionActions:{}, queuedEvent:null,
    lastElection:null, pm:null, coalitionSlots:[], invitations:{}, cabinet:{}, coalitionParties:[],
    turn:1, turnsPerTerm:16, usedEventIds:[], currentEvent:null, currentEventOwnerSlot:null, currentEffectResult:null,
    collapsed:false, electionLock:false, pendingCollapse:false, deficitStreak:0, concededPm:[],
  });
}
```

- [ ] **Step 2: Manually verify from each phase**

Using two browser tabs against the same room (tab A = host, tab B = a second player who joined with the room code):

1. In tab A (host), progress the game to `governing` phase (same room from Task 3/4).
2. Click "รีเซ็ตเกม", confirm the `confirm()` dialog.
3. Expected in **both** tabs: screen returns to the lobby ("แบ่งพรรคการเมือง") showing the same room code, both players' slots still show their claimed party and player name (not empty/unclaimed).
4. In devtools console: `db.collection('jadtang_rooms').doc(ui.roomCode).get().then(s=>console.log(s.data()))` — expect `phase:'lobby'`, `treasury:200`, `deficitStreak:0`, `concededPm:[]`, `turn:1`, `term:1`.
5. Repeat steps 1–4 starting from `campaign`, `coalition`, and `cabinet` phases to confirm the button works from every phase (the button only renders inside `topbar()`, which every phase past `lobby` includes).

- [ ] **Step 3: Commit**

```bash
git add "index (3).html"
git commit -m "Add host-only Reset Game button that returns the room to lobby"
```

---

### Task 6: Decouple election-result calculation from host-only automation

**Files:**
- Modify: `index (3).html` — `subscribeRoom()` and `runHostAutomation()`

**Interfaces:**
- Consumes: `maybeRunElection()` (already exists, unmodified — already safe to call from any client because it re-checks `electionLock` inside a Firestore transaction before writing)

- [ ] **Step 1: Call `maybeRunElection()` unconditionally from every client's snapshot handler**

Find this exact block:

```javascript
function subscribeRoom(code){
  if(ui.unsub) ui.unsub();
  ui.unsub = db.collection(ROOMS).doc(code).onSnapshot(snap=>{
    if(!snap.exists){ ui.room=null; ui.screen='home'; render(); return; }
    ui.room=snap.data();
    if(ui.screen!=='game') ui.screen='game';
    render();
    runHostAutomation();
  });
  ui.roomCode=code;
  localStorage.setItem('jadtang_last_room',code);
}
```

Replace with:

```javascript
function subscribeRoom(code){
  if(ui.unsub) ui.unsub();
  ui.unsub = db.collection(ROOMS).doc(code).onSnapshot(snap=>{
    if(!snap.exists){ ui.room=null; ui.screen='home'; render(); return; }
    ui.room=snap.data();
    if(ui.screen!=='game') ui.screen='game';
    render();
    maybeRunElection();
    runHostAutomation();
  });
  ui.roomCode=code;
  localStorage.setItem('jadtang_last_room',code);
}
```

- [ ] **Step 2: Remove the now-redundant campaign branch from `runHostAutomation()`**

Find this exact block:

```javascript
function runHostAutomation(){
  if(!isHost()||automationBusy||!ui.room) return;
  const r=ui.room;

  if(r.phase==='campaign'){ maybeRunElection(); return; }

  if(r.phase==='coalition' && !r.slots.find(s=>s.id===r.pm).claimedBy){
```

Replace with:

```javascript
function runHostAutomation(){
  if(!isHost()||automationBusy||!ui.room) return;
  const r=ui.room;

  if(r.phase==='coalition' && !r.slots.find(s=>s.id===r.pm).claimedBy){
```

- [ ] **Step 3: Manually verify with two tabs, non-host submitting last**

Using two browser tabs on the same room (tab A = host, tab B = second player):

1. Both players claim a party slot, click "เริ่มเกม" (host only), both reach `campaign`.
2. In tab A (host), submit policy/budget first.
3. In tab B (**non-host**), submit last.
4. Expected: both tabs transition to `election_result` within ~1 second of tab B's submission — confirming the calculation no longer depends on the host's tab being the one to react.
5. Repeat the same flow but simulate a "stuck host" by closing tab A's browser tab entirely right after tab A submits (before tab B submits). Then submit in tab B. Expected: tab B still transitions to `election_result` on its own (proving a non-host client can single-handedly trigger the election calculation even with zero other tabs open).

- [ ] **Step 4: Commit**

```bash
git add "index (3).html"
git commit -m "Let any connected client trigger the election calculation, not just the host"
```

---

### Task 7: Pure function — `nextPmCandidate`

**Files:**
- Modify: `index (3).html` — add near `confirmCoalition()`

**Interfaces:**
- Consumes: a room-shaped object with `.concededPm` (array) and `.lastElection.results` (array of `{id, seats, ...}`)
- Produces: `nextPmCandidate(r) -> {id, seats, ...} | null`. Task 8 calls this.

- [ ] **Step 1: Validate the logic in isolation with Node**

```bash
node -e "
function nextPmCandidate(r){
  const concededPm=r.concededPm||[];
  const ranked=[...r.lastElection.results].sort((a,b)=>b.seats-a.seats);
  return ranked.find(x=>x.seats>0 && !concededPm.includes(x.id)) || null;
}

const results=[{id:'s0',seats:80},{id:'s1',seats:60},{id:'s2',seats:40},{id:'s3',seats:0}];

// case 1: nobody conceded yet -> pick top seats
let c1=nextPmCandidate({concededPm:[],lastElection:{results}});
console.assert(c1.id==='s0','case1: expected s0 got '+c1.id);

// case 2: top conceded -> next runner-up
let c2=nextPmCandidate({concededPm:['s0'],lastElection:{results}});
console.assert(c2.id==='s1','case2: expected s1 got '+c2.id);

// case 3: parties with 0 seats are never candidates
let c3=nextPmCandidate({concededPm:['s0','s1','s2'],lastElection:{results}});
console.assert(c3===null,'case3: expected null got '+JSON.stringify(c3));

console.log('ALL PASS');
"
```

Expected output: `ALL PASS`.

- [ ] **Step 2: Add the exact same function to `index (3).html`**

Find this exact block:

```javascript
function confirmCoalition(){
  const r=ui.room;
  const acceptedIds=Object.entries(r.invitations).filter(([k,v])=>v==='accepted').map(([k])=>k);
  const coalitionSlots=[r.pm,...acceptedIds];
  const coalitionParties=coalitionSlots.map(id=>({id,satisfaction:id===r.pm?100:50}));
  stampAndThen('ลงนามแล้ว',()=>{ roomRef().update({phase:'cabinet',coalitionSlots,coalitionParties,cabinet:{}}); });
}
```

Replace with (unchanged `confirmCoalition`, adds `nextPmCandidate` right after):

```javascript
function confirmCoalition(){
  const r=ui.room;
  const acceptedIds=Object.entries(r.invitations).filter(([k,v])=>v==='accepted').map(([k])=>k);
  const coalitionSlots=[r.pm,...acceptedIds];
  const coalitionParties=coalitionSlots.map(id=>({id,satisfaction:id===r.pm?100:50}));
  stampAndThen('ลงนามแล้ว',()=>{ roomRef().update({phase:'cabinet',coalitionSlots,coalitionParties,cabinet:{}}); });
}

function nextPmCandidate(r){
  const concededPm=r.concededPm||[];
  const ranked=[...r.lastElection.results].sort((a,b)=>b.seats-a.seats);
  return ranked.find(x=>x.seats>0 && !concededPm.includes(x.id)) || null;
}
```

- [ ] **Step 3: Confirm the function is present verbatim**

```bash
grep -A4 "function nextPmCandidate" "index (3).html"
```

- [ ] **Step 4: Commit**

```bash
git add "index (3).html"
git commit -m "Add nextPmCandidate pure function for PM-concession cascade"
```

---

### Task 8: PM concession — `restartElectionSameTerm`, `concedePmCandidacy`, and the coalition-screen button

**Files:**
- Modify: `index (3).html` — add functions after `nextPmCandidate` (Task 7); modify `PHASE_TEMPLATES.coalition`

**Interfaces:**
- Consumes: `nextPmCandidate(r)` from Task 7
- Produces: `concedePmCandidacy()`, `restartElectionSameTerm()` — Task 9's AI automation calls `concedePmCandidacy()`.

- [ ] **Step 1: Add `restartElectionSameTerm()` and `concedePmCandidacy()`**

Find this exact block (the end of Task 7's edit):

```javascript
function nextPmCandidate(r){
  const concededPm=r.concededPm||[];
  const ranked=[...r.lastElection.results].sort((a,b)=>b.seats-a.seats);
  return ranked.find(x=>x.seats>0 && !concededPm.includes(x.id)) || null;
}
```

Replace with:

```javascript
function nextPmCandidate(r){
  const concededPm=r.concededPm||[];
  const ranked=[...r.lastElection.results].sort((a,b)=>b.seats-a.seats);
  return ranked.find(x=>x.seats>0 && !concededPm.includes(x.id)) || null;
}
function restartElectionSameTerm(){
  const r=ui.room;
  const slots=r.slots.map(s=>({...s,submitted:false,budget:{north:0,isaan:0,central:0,bangkok:0,south:0}}));
  roomRef().update({phase:'campaign',slots,electionLock:false,concededPm:[],pm:null,coalitionSlots:[],invitations:{},cabinet:{}});
}
function concedePmCandidacy(){
  const r=ui.room;
  const concededPm=[...(r.concededPm||[]),r.pm];
  const nextR={...r,concededPm};
  const next=nextPmCandidate(nextR);
  if(!next){ restartElectionSameTerm(); return; }
  roomRef().update({pm:next.id,concededPm,invitations:{},coalitionSlots:[],cabinet:{}});
}
```

- [ ] **Step 2: Add the "สละสิทธิ์" button to the coalition screen**

Find this exact block inside `PHASE_TEMPLATES.coalition`:

```javascript
    ${isPM?`<div class="btn-row"><button class="btn btn-stamp" ${ok?'':'disabled'} onclick="confirmCoalition()">ลงนามจัดตั้งรัฐบาล</button></div>`
      : (!pmSlot.claimedBy?`<div class="waiting-box">AI กำลังเจรจาจัดตั้งรัฐบาล...</div>`:`<div class="waiting-box">รอพรรคแกนนำสรุปรัฐบาลผสม...</div>`)}
  </div>
  `;
};
```

Replace with:

```javascript
    ${isPM?`<div class="btn-row"><button class="btn btn-ghost-dark" onclick="concedePmCandidacy()">สละสิทธิ์เป็นแกนนำจัดตั้งรัฐบาล</button><button class="btn btn-stamp" ${ok?'':'disabled'} onclick="confirmCoalition()">ลงนามจัดตั้งรัฐบาล</button></div>`
      : (!pmSlot.claimedBy?`<div class="waiting-box">AI กำลังเจรจาจัดตั้งรัฐบาล...</div>`:`<div class="waiting-box">รอพรรคแกนนำสรุปรัฐบาลผสม...</div>`)}
  </div>
  `;
};
```

- [ ] **Step 3: Manually verify the player-facing concede flow**

Using two tabs on a fresh room (same setup as Task 6):

1. Play through `campaign` with policies set so **no single party reaches majority** (e.g. keep both human parties' economy/reform/security sliders near 0 so seats split roughly evenly across many AI parties too).
2. Reach `coalition` phase as the human PM candidate (claim the top-seat slot before the election if needed, or replay until a human is the top party).
3. Click "สละสิทธิ์เป็นแกนนำจัดตั้งรัฐบาล".
4. Expected: `r.pm` changes to the next-highest-seat party that isn't the one you just conceded; `invitations`, `coalitionSlots`, `cabinet` are all empty again; the screen re-renders showing the new PM candidate's name in "พรรคแกนนำ: ...".
5. Repeat conceding until every party with seats has conceded once (verify via console: `db.collection('jadtang_rooms').doc(ui.roomCode).get().then(s=>console.log(s.data().concededPm))` — should grow by one id each time).
6. On the final concede (no candidate left), expected: phase returns to `campaign`, `concededPm` is empty again, both players must re-submit policy/budget.

- [ ] **Step 4: Commit**

```bash
git add "index (3).html"
git commit -m "Add PM-candidacy concession flow with cascade to next-largest party"
```

---

### Task 9: AI-controlled PM auto-concedes instead of getting stuck

**Files:**
- Modify: `index (3).html` — the coalition branch inside `runHostAutomation()`

**Interfaces:**
- Consumes: `concedePmCandidacy()` from Task 8

- [ ] **Step 1: Modify the AI coalition-automation branch**

Find this exact block:

```javascript
          const acceptedIds=Object.entries(invitations).filter(([k,v])=>v==='accepted').map(([k])=>k);
          const coalitionSlots=[r.pm,...acceptedIds];
          const coalitionParties=coalitionSlots.map(id=>({id,satisfaction:id===r.pm?100:50}));
          roomRef().update({invitations,phase:'cabinet',coalitionSlots,coalitionParties,cabinet:{}}).catch(()=>{}).finally(releaseAutomation);
        } catch(e){ console.error('AI coalition automation failed:',e); releaseAutomation(); }
      },700);
    }
    return;
  }
```

Replace with:

```javascript
          if(total<MAJORITY){
            concedePmCandidacy();
            releaseAutomation();
            return;
          }
          const acceptedIds=Object.entries(invitations).filter(([k,v])=>v==='accepted').map(([k])=>k);
          const coalitionSlots=[r.pm,...acceptedIds];
          const coalitionParties=coalitionSlots.map(id=>({id,satisfaction:id===r.pm?100:50}));
          roomRef().update({invitations,phase:'cabinet',coalitionSlots,coalitionParties,cabinet:{}}).catch(()=>{}).finally(releaseAutomation);
        } catch(e){ console.error('AI coalition automation failed:',e); releaseAutomation(); }
      },700);
    }
    return;
  }
```

- [ ] **Step 2: Manually verify the AI cascades on its own without any player input**

1. Create a fresh room, claim only one party slot (so the other 5 are AI-controlled), set your own party's policy sliders to something extreme (e.g. economy `100`, reform `-100`, security `100`) so you're unlikely to be invited into any AI-led coalition, and submit.
2. Let the room run entirely on AI (don't touch anything) until it reaches an AI-vs-AI coalition negotiation where the AI PM candidate can't reach majority on the first attempt.
3. Watch via console poll:

```javascript
setInterval(()=>db.collection('jadtang_rooms').doc(ui.roomCode).get().then(s=>console.log(s.data().phase, s.data().pm, s.data().concededPm)),1000)
```

Expected: `pm` changes across consecutive log lines (cascading through candidates) without the game ever staying on `phase:'coalition'` with the same `pm` and no `invitations` progress for more than a few seconds — eventually it either reaches `phase:'cabinet'` or, in the worst case, `phase:'campaign'` with `concededPm:[]` (full reset via `restartElectionSameTerm`), but it must **not** hang indefinitely on `coalition`.

- [ ] **Step 3: Commit**

```bash
git add "index (3).html"
git commit -m "Have AI-controlled PM candidates concede automatically instead of stalling"
```

---

### Task 10: Full regression pass across all four fixes

**Files:**
- None (verification only)

- [ ] **Step 1: Play one complete game start-to-finish with two human tabs**

Using the local server from Task 1, two tabs (host + second player), play a full term:
1. Confirm the breadcrumb/governing screen shows `ไตรมาสที่ N / 16` (Task 1/3).
2. Deliberately run treasury negative for 3+ turns; confirm the badge and red treasury track correctly and the game keeps running rather than crashing (Task 3/4).
3. Have the non-host tab submit campaign policy last; confirm election results appear promptly (Task 6).
4. Force a hung coalition (both human parties pick incompatible policies) and confirm the human PM can concede and, if pushed to the limit, the game auto-recovers to a fresh campaign rather than freezing (Task 8/9).
5. Mid-game, click "รีเซ็ตเกม" as host; confirm both tabs return to lobby with slots/names intact (Task 5).

- [ ] **Step 2: Check for console errors**

Open devtools console in both tabs throughout the run in Step 1. Expected: no uncaught exceptions logged at any point.

- [ ] **Step 3: Final commit (only if Step 1/2 surfaced fixups)**

If no fixups were needed, this task requires no commit — Tasks 1–9 already cover all shipped changes. If Step 1 or 2 surfaced a bug, fix it, re-run Step 1/2, then:

```bash
git add "index (3).html"
git commit -m "Fix regression found in Group A end-to-end verification"
```
