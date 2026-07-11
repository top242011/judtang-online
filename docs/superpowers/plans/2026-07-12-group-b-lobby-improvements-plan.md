# Group B Lobby Improvements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let the host enable/disable which of the 6 fixed parties are in play, lock an empty slot to AI-only or kick a claimed player back to AI, and require every claimed player to mark themselves Ready before the host can start the game — per [docs/superpowers/specs/2026-07-12-group-b-lobby-improvements-design.md](../specs/2026-07-12-group-b-lobby-improvements-design.md).

**Architecture:** Same single-file `index (3).html`, no build step, Firestore as the only backend. This plan builds on Group A (already merged into `main` at commit `9375c22`) and does not touch any Group A code paths except `resetRoomToLobby()`'s slot-rebuild (adding one field to its reset).

**Tech Stack:** Vanilla JS, Firebase JS SDK 10.7.1 (compat), Firestore. No automated test framework exists in this project. Every task in this plan is a Lobby/Firestore/UI change with no pure-logic component, so all verification is manual browser-based (two tabs simulating two players, exactly as used throughout Group A).

## Global Constraints

- Do not introduce a build step, bundler, package.json, or any new dependency.
- All new user-facing strings must be in Thai, matching the existing tone.
- New room-state fields, one per slot (not room-level): `enabled` (boolean, default `true`), `aiLocked` (boolean, default `false`), `ready` (boolean, default `false`).
- Minimum enabled-party count is 2 — enforced by disabling the "ปิดพรรคนี้" button in the UI before the count would drop below 2, not by an error after the click.
- A slot can only be disabled or AI-locked while `!s.claimedBy` (unclaimed). A claimed slot must be "kicked" first (which unclaims it and sets `aiLocked:true` in the same write).
- Function names introduced across tasks must match exactly (a later task's UI wiring depends on an earlier task's function): `toggleSlotEnabled`, `toggleSlotAiLock`, `kickSlot`, `toggleMyReady`.
- `startGame()` must filter `slots` down to `enabled===true` parties — this is the single choke point where disabled parties leave the game for the rest of that term; no other function needs to check `enabled` after this.
- `resetRoomToLobby()` (from Group A) must reset `ready:false` on every slot but must NOT reset `enabled`/`aiLocked` — those carry over from the host's prior lobby configuration via the existing `...s` spread.

---

### Task 1: Room-creation and reset defaults for `enabled`/`aiLocked`/`ready`

**Files:**
- Modify: `index (3).html` — `handleCreateRoom()` and `resetRoomToLobby()`

**Interfaces:**
- Produces: every slot in a freshly created room, and every slot after a host reset, has `enabled:true`, `aiLocked:false`, `ready:false` (reset preserves `enabled`/`aiLocked`, resets `ready`). Tasks 2-4 assume every slot has these three fields.

- [ ] **Step 1: Add the three fields to `handleCreateRoom()`'s slot template**

Find this exact block:

```javascript
  const slots=PARTY_TEMPLATES.map((t,i)=>({
    id:'s'+i, name:t.name, color:t.color, emblem:t.name[3]||t.name[0],
    policy:{...t.policy}, claimedBy:null, playerDisplayName:null,
    submitted:false, budget:{north:0,isaan:0,central:0,bangkok:0,south:0}
  }));
```

Replace with:

```javascript
  const slots=PARTY_TEMPLATES.map((t,i)=>({
    id:'s'+i, name:t.name, color:t.color, emblem:t.name[3]||t.name[0],
    policy:{...t.policy}, claimedBy:null, playerDisplayName:null,
    submitted:false, budget:{north:0,isaan:0,central:0,bangkok:0,south:0},
    enabled:true, aiLocked:false, ready:false,
  }));
```

- [ ] **Step 2: Reset `ready` (but not `enabled`/`aiLocked`) in `resetRoomToLobby()`**

Find this exact block:

```javascript
  const slots=r.slots.map((s,i)=>({
    ...s,
    policy:{...PARTY_TEMPLATES[i].policy},
    submitted:false,
    budget:{...emptyBudget},
  }));
```

Replace with:

```javascript
  const slots=r.slots.map((s,i)=>({
    ...s,
    policy:{...PARTY_TEMPLATES[i].policy},
    submitted:false,
    budget:{...emptyBudget},
    ready:false,
  }));
```

(The `...s` spread already carries `enabled`/`aiLocked` forward unchanged — this step only adds the explicit `ready:false` override.)

- [ ] **Step 3: Manually verify**

```bash
cd "/Users/panupanpitak/Projects/judtang-online" && python3 -m http.server 8771
```

Open `http://localhost:8771/index%20(3).html`, create a room, and in devtools console:

```javascript
db.collection('jadtang_rooms').doc(ui.roomCode).get().then(s=>console.log(s.data().slots.map(x=>({id:x.id,enabled:x.enabled,aiLocked:x.aiLocked,ready:x.ready}))))
```

Expected: all 6 slots show `enabled:true, aiLocked:false, ready:false`.

Then click "รีเซ็ตเกม" (from the topbar — works even while still in lobby) and re-run the same console query — expected: still `enabled:true, aiLocked:false, ready:false` (nothing to have changed yet since you haven't toggled anything). This confirms the reset path doesn't crash or drop the new fields.

- [ ] **Step 4: Commit**

```bash
git add "index (3).html"
git commit -m "Add enabled/aiLocked/ready slot fields to room creation and reset"
```

---

### Task 2: Enable/disable toggle with a 2-party minimum, filtered at `startGame()`

**Files:**
- Modify: `index (3).html` — `slotCard()`, `startGame()`; add `toggleSlotEnabled()`

**Interfaces:**
- Consumes: `s.enabled` (Task 1), `isHost()` (pre-existing)
- Produces: `toggleSlotEnabled(idx)` — called by the button this task adds to `slotCard()`.

- [ ] **Step 1: Rewrite `slotCard()` to branch on `enabled` and add the disable/enable button**

Find this exact block:

```javascript
function slotCard(s,i,mine){
  const taken=!!s.claimedBy;
  const isMine=mine&&mine.id===s.id;
  return `<div class="slot-card ${taken?'taken':''}">
    <span class="dot" style="background:${s.color}"></span>
    <div class="info">
      <div class="sname">${s.name}</div>
      <div class="smeta">${taken?('ผู้เล่น: '+(s.playerDisplayName||'ไม่ทราบชื่อ')+(isMine?' (คุณ)':'')):'ว่าง — จะให้ AI เล่นแทนถ้าไม่มีใครเลือก'}</div>
    </div>
    ${!taken?`<button class="btn btn-ghost-dark" onclick="claimSlot(${i})">เข้าร่วมพรรคนี้</button>`:(isMine?`<span class="tag human">คุณ</span>`:`<span class="tag human">มีคนแล้ว</span>`)}
  </div>`;
}
```

Replace with:

```javascript
function slotCard(s,i,mine){
  const taken=!!s.claimedBy;
  const isMine=mine&&mine.id===s.id;
  const host=isHost();
  if(!s.enabled){
    return `<div class="slot-card" style="opacity:0.45;">
      <span class="dot" style="background:${s.color}"></span>
      <div class="info">
        <div class="sname">${s.name}</div>
        <div class="smeta">ปิดใช้งาน</div>
      </div>
      ${host?`<button class="btn btn-ghost-dark" onclick="toggleSlotEnabled(${i})">เปิดพรรคนี้</button>`:''}
    </div>`;
  }
  const enabledCount=ui.room.slots.filter(x=>x.enabled).length;
  const canDisable=host && !taken && enabledCount>2;
  return `<div class="slot-card ${taken?'taken':''}">
    <span class="dot" style="background:${s.color}"></span>
    <div class="info">
      <div class="sname">${s.name}</div>
      <div class="smeta">${taken?('ผู้เล่น: '+(s.playerDisplayName||'ไม่ทราบชื่อ')+(isMine?' (คุณ)':'')):'ว่าง — จะให้ AI เล่นแทนถ้าไม่มีใครเลือก'}</div>
    </div>
    ${!taken?`<button class="btn btn-ghost-dark" onclick="claimSlot(${i})">เข้าร่วมพรรคนี้</button>`:(isMine?`<span class="tag human">คุณ</span>`:`<span class="tag human">มีคนแล้ว</span>`)}
    ${host && !taken?`<button class="btn btn-ghost-dark" ${canDisable?'':'disabled'} onclick="toggleSlotEnabled(${i})">ปิดพรรคนี้</button>`:''}
  </div>`;
}
```

- [ ] **Step 2: Add `toggleSlotEnabled()`**

Find this exact block (the end of `updateMySlotField()`):

```javascript
function updateMySlotField(field,val){
  const slots=ui.room.slots.map(s=>s.claimedBy===MY_ID?{...s,[field]:val}:s);
  roomRef().update({slots});
}
```

Replace with (unchanged `updateMySlotField`, adds `toggleSlotEnabled` right after):

```javascript
function updateMySlotField(field,val){
  const slots=ui.room.slots.map(s=>s.claimedBy===MY_ID?{...s,[field]:val}:s);
  roomRef().update({slots});
}
function toggleSlotEnabled(idx){
  if(!isHost()) return;
  const slots=ui.room.slots.map(s=>({...s}));
  const target=slots[idx];
  if(target.claimedBy) return;
  if(target.enabled){
    const enabledCount=slots.filter(s=>s.enabled).length;
    if(enabledCount<=2) return;
    target.enabled=false;
  } else {
    target.enabled=true;
  }
  roomRef().update({slots});
}
```

- [ ] **Step 3: Filter disabled parties out at `startGame()`**

Find this exact block:

```javascript
function startGame(){
  const slots=ui.room.slots.map(s=>({...s,submitted:false,budget:{north:0,isaan:0,central:0,bangkok:0,south:0}}));
  roomRef().update({phase:'campaign',slots});
}
```

Replace with:

```javascript
function startGame(){
  const slots=ui.room.slots.filter(s=>s.enabled).map(s=>({...s,submitted:false,budget:{north:0,isaan:0,central:0,bangkok:0,south:0}}));
  roomRef().update({phase:'campaign',slots});
}
```

- [ ] **Step 4: Manually verify**

Using the local server from Task 1, in a fresh room as host: click "ปิดพรรคนี้" on 4 of the 6 unclaimed slots one at a time. After disabling the 4th one (2 remain enabled), confirm the "ปิดพรรคนี้" button on both remaining slots is now disabled (greyed out / unclickable) — this proves the 2-party floor is enforced before the click, not after.

Claim one of the 2 remaining slots yourself, have a second browser tab (with a different `localStorage.setItem('jadtang_pid', ...)`) join and claim the other, both mark ready (you'll need this working from Task 4 to actually start — for this task's verification, you can instead call `startGame()` directly from the console once both are claimed, bypassing the ready gate which doesn't exist yet). Confirm via:

```javascript
db.collection('jadtang_rooms').doc(ui.roomCode).get().then(s=>console.log(s.data().phase, s.data().slots.length))
```

Expected: `phase:'campaign'`, `slots.length === 2` (the disabled 4 are gone from the array entirely).

- [ ] **Step 5: Commit**

```bash
git add "index (3).html"
git commit -m "Add host-only enable/disable toggle for parties, filtered at game start"
```

---

### Task 3: AI-lock unclaimed slots and kick claimed players back to AI

**Files:**
- Modify: `index (3).html` — `slotCard()`, `claimSlot()`; add `toggleSlotAiLock()`, `kickSlot()`

**Interfaces:**
- Consumes: `s.aiLocked` (Task 1)
- Produces: `toggleSlotAiLock(idx)`, `kickSlot(idx)` — called by buttons this task adds to `slotCard()`.

- [ ] **Step 1: Extend `slotCard()`'s enabled branch with AI-lock and kick markup**

Find this exact block (the enabled-branch `return` from Task 2):

```javascript
  const enabledCount=ui.room.slots.filter(x=>x.enabled).length;
  const canDisable=host && !taken && enabledCount>2;
  return `<div class="slot-card ${taken?'taken':''}">
    <span class="dot" style="background:${s.color}"></span>
    <div class="info">
      <div class="sname">${s.name}</div>
      <div class="smeta">${taken?('ผู้เล่น: '+(s.playerDisplayName||'ไม่ทราบชื่อ')+(isMine?' (คุณ)':'')):'ว่าง — จะให้ AI เล่นแทนถ้าไม่มีใครเลือก'}</div>
    </div>
    ${!taken?`<button class="btn btn-ghost-dark" onclick="claimSlot(${i})">เข้าร่วมพรรคนี้</button>`:(isMine?`<span class="tag human">คุณ</span>`:`<span class="tag human">มีคนแล้ว</span>`)}
    ${host && !taken?`<button class="btn btn-ghost-dark" ${canDisable?'':'disabled'} onclick="toggleSlotEnabled(${i})">ปิดพรรคนี้</button>`:''}
  </div>`;
}
```

Replace with:

```javascript
  const enabledCount=ui.room.slots.filter(x=>x.enabled).length;
  const canDisable=host && !taken && enabledCount>2;
  return `<div class="slot-card ${taken?'taken':''}">
    <span class="dot" style="background:${s.color}"></span>
    <div class="info">
      <div class="sname">${s.name}</div>
      <div class="smeta">${taken?('ผู้เล่น: '+(s.playerDisplayName||'ไม่ทราบชื่อ')+(isMine?' (คุณ)':'')):(s.aiLocked?'ล็อคให้ AI เล่นแทน':'ว่าง — จะให้ AI เล่นแทนถ้าไม่มีใครเลือก')}</div>
    </div>
    ${!taken && !s.aiLocked?`<button class="btn btn-ghost-dark" onclick="claimSlot(${i})">เข้าร่วมพรรคนี้</button>`:''}
    ${taken?(isMine?`<span class="tag human">คุณ</span>`:`<span class="tag human">มีคนแล้ว</span>`):(s.aiLocked?`<span class="tag ai">AI เท่านั้น</span>`:'')}
    ${host && !taken?`<button class="btn btn-ghost-dark" onclick="toggleSlotAiLock(${i})">${s.aiLocked?'ปลดล็อค AI':'ล็อคเป็น AI'}</button>`:''}
    ${host && taken && !isMine?`<button class="btn btn-ghost-dark" onclick="kickSlot(${i})">เตะออก</button>`:''}
    ${host && !taken?`<button class="btn btn-ghost-dark" ${canDisable?'':'disabled'} onclick="toggleSlotEnabled(${i})">ปิดพรรคนี้</button>`:''}
  </div>`;
}
```

- [ ] **Step 2: Add `toggleSlotAiLock()` and `kickSlot()`**

Find this exact block (the end of `toggleSlotEnabled()` from Task 2):

```javascript
function toggleSlotEnabled(idx){
  if(!isHost()) return;
  const slots=ui.room.slots.map(s=>({...s}));
  const target=slots[idx];
  if(target.claimedBy) return;
  if(target.enabled){
    const enabledCount=slots.filter(s=>s.enabled).length;
    if(enabledCount<=2) return;
    target.enabled=false;
  } else {
    target.enabled=true;
  }
  roomRef().update({slots});
}
```

Replace with (unchanged `toggleSlotEnabled`, adds the two new functions right after):

```javascript
function toggleSlotEnabled(idx){
  if(!isHost()) return;
  const slots=ui.room.slots.map(s=>({...s}));
  const target=slots[idx];
  if(target.claimedBy) return;
  if(target.enabled){
    const enabledCount=slots.filter(s=>s.enabled).length;
    if(enabledCount<=2) return;
    target.enabled=false;
  } else {
    target.enabled=true;
  }
  roomRef().update({slots});
}
function toggleSlotAiLock(idx){
  if(!isHost()) return;
  const slots=ui.room.slots.map(s=>({...s}));
  const target=slots[idx];
  if(target.claimedBy) return;
  target.aiLocked=!target.aiLocked;
  roomRef().update({slots});
}
function kickSlot(idx){
  if(!isHost()) return;
  const slots=ui.room.slots.map(s=>({...s}));
  const target=slots[idx];
  if(!target.claimedBy || target.claimedBy===MY_ID) return;
  target.claimedBy=null;
  target.playerDisplayName=null;
  target.ready=false;
  target.aiLocked=true;
  roomRef().update({slots});
}
```

- [ ] **Step 3: Guard `claimSlot()` against locked/disabled slots**

Find this exact block:

```javascript
function claimSlot(idx){
  const slots=ui.room.slots.map(s=>({...s}));
  slots.forEach(s=>{ if(s.claimedBy===MY_ID){ s.claimedBy=null; s.playerDisplayName=null; } });
  slots[idx].claimedBy=MY_ID;
  slots[idx].playerDisplayName=ui.nameInput||'ผู้เล่น';
  roomRef().update({slots});
}
```

Replace with:

```javascript
function claimSlot(idx){
  const slots=ui.room.slots.map(s=>({...s}));
  if(!slots[idx].enabled || slots[idx].aiLocked) return;
  slots.forEach(s=>{ if(s.claimedBy===MY_ID){ s.claimedBy=null; s.playerDisplayName=null; } });
  slots[idx].claimedBy=MY_ID;
  slots[idx].playerDisplayName=ui.nameInput||'ผู้เล่น';
  roomRef().update({slots});
}
```

- [ ] **Step 4: Manually verify with two tabs**

Using two browser tabs on the same room (tab A = host, tab B = second player — remember to set a distinct `localStorage.setItem('jadtang_pid','p_faketestplayer2')` on tab B before it loads the game, since both tabs share the same browser storage origin):

1. In tab A, click "ล็อคเป็น AI" on an unclaimed slot. In tab B, confirm that slot's "เข้าร่วมพรรคนี้" button is gone and it shows the "AI เท่านั้น" tag instead.
2. In tab A, click "ปลดล็อค AI" on that same slot — confirm tab B's join button reappears.
3. In tab B, claim a different slot. In tab A (host), click "เตะออก" on tab B's slot. Confirm: tab B's UI shows itself back to "ยังไม่ได้เลือกพรรค" (no `mine` slot), and both tabs show that slot as `aiLocked` (join button gone, "AI เท่านั้น" tag showing) without you touching the AI-lock toggle yourself.
4. Confirm the host never sees a "เตะออก" button on their own claimed slot.

- [ ] **Step 5: Commit**

```bash
git add "index (3).html"
git commit -m "Add AI-lock toggle and kick-to-AI for lobby slots"
```

---

### Task 4: Mandatory Ready gate before the host can start

**Files:**
- Modify: `index (3).html` — `PHASE_TEMPLATES.lobby`, `slotCard()`, `updateMySlotField()`; add `toggleMyReady()`

**Interfaces:**
- Consumes: `s.ready` (Task 1)
- Produces: `toggleMyReady()` — called by the button this task adds to `PHASE_TEMPLATES.lobby`.

- [ ] **Step 1: Add the Ready toggle button and gate the Start button on it**

Find this exact block:

```javascript
PHASE_TEMPLATES.lobby=function(){
  const r=ui.room;
  const mine=mySlot();
  const humanCount=claimedSlots().length;
  return `
  <div class="panel-dark" style="text-align:center;">
    <div class="term-badge">ห้องรอผู้เล่น</div>
    <h1 class="serif" style="font-size:26px;margin:8px 0 2px;">แบ่งพรรคการเมือง</h1>
    <div class="code-display">${r.roomCode}</div>
    <div style="font-size:12.5px;color:var(--muted-on-navy);">ส่งรหัสนี้ให้เพื่อน แล้วให้เพื่อนกด "เข้าร่วมห้องด้วยรหัส" ที่หน้าแรก</div>
  </div>
  <div class="doc">
    <h2>เลือกพรรคของคุณ</h2>
    <div class="doc-sub">ผู้เล่น ${humanCount} / ${r.slots.length} คน &middot; ที่เหลือจะให้ AI เล่นแทนเมื่อเริ่มเกม</div>
    ${r.slots.map((s,i)=>slotCard(s,i,mine)).join('')}

    ${mine?`
    <div style="margin-top:18px;border-top:1px solid var(--paper-line);padding-top:16px;">
      <label style="font-size:12.5px;font-weight:600;">ตั้งชื่อพรรคของคุณ</label>
      <input class="field" style="margin-top:6px;" value="${mine.name}" onchange="updateMySlotField('name',this.value)">
      <div style="margin-top:12px;">
        <label style="font-size:12.5px;font-weight:600;">สีประจำพรรค</label>
        <div class="color-swatches" style="margin-top:8px;">
          ${PARTY_TEMPLATES.map(t=>t.color).map(c=>`<button style="background:${c};" class="${mine.color===c?'picked':''}" onclick="updateMySlotField('color','${c}')"></button>`).join('')}
        </div>
      </div>
    </div>` : ''}

    <div class="btn-row">
      ${isHost()?`<button class="btn btn-stamp" ${humanCount<2?'disabled':''} onclick="startGame()">เริ่มเกม (${humanCount} ผู้เล่น)</button>`:`<div class="waiting-box" style="padding:6px;">รอโฮสต์เริ่มเกม...</div>`}
    </div>
  </div>
  `;
};
```

Replace with:

```javascript
PHASE_TEMPLATES.lobby=function(){
  const r=ui.room;
  const mine=mySlot();
  const humanCount=claimedSlots().length;
  const readyCount=claimedSlots().filter(s=>s.ready).length;
  return `
  <div class="panel-dark" style="text-align:center;">
    <div class="term-badge">ห้องรอผู้เล่น</div>
    <h1 class="serif" style="font-size:26px;margin:8px 0 2px;">แบ่งพรรคการเมือง</h1>
    <div class="code-display">${r.roomCode}</div>
    <div style="font-size:12.5px;color:var(--muted-on-navy);">ส่งรหัสนี้ให้เพื่อน แล้วให้เพื่อนกด "เข้าร่วมห้องด้วยรหัส" ที่หน้าแรก</div>
  </div>
  <div class="doc">
    <h2>เลือกพรรคของคุณ</h2>
    <div class="doc-sub">ผู้เล่น ${humanCount} คน &middot; พร้อมแล้ว ${readyCount}/${humanCount} &middot; ที่เหลือจะให้ AI เล่นแทนเมื่อเริ่มเกม</div>
    ${r.slots.map((s,i)=>slotCard(s,i,mine)).join('')}

    ${mine?`
    <div style="margin-top:18px;border-top:1px solid var(--paper-line);padding-top:16px;">
      <label style="font-size:12.5px;font-weight:600;">ตั้งชื่อพรรคของคุณ</label>
      <input class="field" style="margin-top:6px;" value="${mine.name}" onchange="updateMySlotField('name',this.value)">
      <div style="margin-top:12px;">
        <label style="font-size:12.5px;font-weight:600;">สีประจำพรรค</label>
        <div class="color-swatches" style="margin-top:8px;">
          ${PARTY_TEMPLATES.map(t=>t.color).map(c=>`<button style="background:${c};" class="${mine.color===c?'picked':''}" onclick="updateMySlotField('color','${c}')"></button>`).join('')}
        </div>
      </div>
      <div class="btn-row left" style="margin-top:14px;">
        <button class="btn ${mine.ready?'btn-ghost-dark':'btn-stamp'}" onclick="toggleMyReady()">${mine.ready?'ยังไม่พร้อม':'พร้อมแล้ว'}</button>
      </div>
    </div>` : ''}

    <div class="btn-row">
      ${isHost()?`<button class="btn btn-stamp" ${(humanCount<2||readyCount<humanCount)?'disabled':''} onclick="startGame()">เริ่มเกม (${humanCount} ผู้เล่น)</button>`:`<div class="waiting-box" style="padding:6px;">รอโฮสต์เริ่มเกม...</div>`}
    </div>
  </div>
  `;
};
```

- [ ] **Step 2: Add the ready tag to `slotCard()`**

Find this exact block (the enabled-branch from Task 3):

```javascript
  const enabledCount=ui.room.slots.filter(x=>x.enabled).length;
  const canDisable=host && !taken && enabledCount>2;
  return `<div class="slot-card ${taken?'taken':''}">
    <span class="dot" style="background:${s.color}"></span>
    <div class="info">
      <div class="sname">${s.name}</div>
      <div class="smeta">${taken?('ผู้เล่น: '+(s.playerDisplayName||'ไม่ทราบชื่อ')+(isMine?' (คุณ)':'')):(s.aiLocked?'ล็อคให้ AI เล่นแทน':'ว่าง — จะให้ AI เล่นแทนถ้าไม่มีใครเลือก')}</div>
    </div>
    ${!taken && !s.aiLocked?`<button class="btn btn-ghost-dark" onclick="claimSlot(${i})">เข้าร่วมพรรคนี้</button>`:''}
    ${taken?(isMine?`<span class="tag human">คุณ</span>`:`<span class="tag human">มีคนแล้ว</span>`):(s.aiLocked?`<span class="tag ai">AI เท่านั้น</span>`:'')}
```

Replace with:

```javascript
  const enabledCount=ui.room.slots.filter(x=>x.enabled).length;
  const canDisable=host && !taken && enabledCount>2;
  const readyTag=taken?`<span class="tag ${s.ready?'accepted':'pending'}">${s.ready?'พร้อมแล้ว':'ยังไม่พร้อม'}</span>`:'';
  return `<div class="slot-card ${taken?'taken':''}">
    <span class="dot" style="background:${s.color}"></span>
    <div class="info">
      <div class="sname">${s.name}</div>
      <div class="smeta">${taken?('ผู้เล่น: '+(s.playerDisplayName||'ไม่ทราบชื่อ')+(isMine?' (คุณ)':'')):(s.aiLocked?'ล็อคให้ AI เล่นแทน':'ว่าง — จะให้ AI เล่นแทนถ้าไม่มีใครเลือก')}</div>
    </div>
    ${readyTag}
    ${!taken && !s.aiLocked?`<button class="btn btn-ghost-dark" onclick="claimSlot(${i})">เข้าร่วมพรรคนี้</button>`:''}
    ${taken?(isMine?`<span class="tag human">คุณ</span>`:`<span class="tag human">มีคนแล้ว</span>`):(s.aiLocked?`<span class="tag ai">AI เท่านั้น</span>`:'')}
```

(The rest of the function — the AI-lock/kick/disable buttons and closing `</div>` — is unchanged from Task 3.)

- [ ] **Step 3: Add `toggleMyReady()` and reset `ready` on name/color edits**

Find this exact block:

```javascript
function updateMySlotField(field,val){
  const slots=ui.room.slots.map(s=>s.claimedBy===MY_ID?{...s,[field]:val}:s);
  roomRef().update({slots});
}
```

Replace with:

```javascript
function updateMySlotField(field,val){
  const slots=ui.room.slots.map(s=>s.claimedBy===MY_ID?{...s,[field]:val,ready:false}:s);
  roomRef().update({slots});
}
function toggleMyReady(){
  const slots=ui.room.slots.map(s=>s.claimedBy===MY_ID?{...s,ready:!s.ready}:s);
  roomRef().update({slots});
}
```

- [ ] **Step 4: Manually verify with two tabs**

Using two tabs on a fresh room (tab A = host claims a slot, tab B = second player claims another — distinct `jadtang_pid` on tab B as before):

1. Before either taps Ready, confirm in tab A the "เริ่มเกม" button is disabled and the doc-sub reads "พร้อมแล้ว 0/2".
2. Tab A clicks "พร้อมแล้ว" — confirm the button now reads "ยังไม่พร้อม" (toggled), the doc-sub updates to "พร้อมแล้ว 1/2", and tab B's view of tab A's slot now shows a green "พร้อมแล้ว" tag instead of yellow "ยังไม่พร้อม". Start button still disabled (only 1/2 ready).
3. Tab B clicks "พร้อมแล้ว" — confirm doc-sub reads "พร้อมแล้ว 2/2" and the host's "เริ่มเกม" button is now enabled.
4. Tab A edits its own party name (the "ตั้งชื่อพรรคของคุณ" input) — confirm tab A's ready state flips back to "พร้อมแล้ว" button label (i.e. `ready` reset to false), doc-sub drops to "พร้อมแล้ว 1/2", and the host's Start button becomes disabled again.
5. Tab A re-clicks Ready, confirm Start re-enables, then actually click "เริ่มเกม" and confirm the room transitions to `campaign` phase correctly (full sanity check that the Task 2 `startGame()` filter and this task's gate don't conflict).

- [ ] **Step 5: Commit**

```bash
git add "index (3).html"
git commit -m "Require every claimed player to mark Ready before the host can start"
```

---

### Task 5: Full regression pass across all three Group B features

**Files:**
- None (verification only)

- [ ] **Step 1: Play one full lobby setup start-to-finish with two human tabs**

Using two tabs (host + second player) on a fresh room:
1. Host disables 3 of the 6 parties (leaving 3 enabled), confirms the 2-party floor prevents disabling below 2.
2. Host AI-locks one of the remaining unclaimed enabled parties.
3. Both humans claim the two other enabled, non-locked parties.
4. Host kicks the second player, confirms they're unclaimed and their old slot is now AI-locked; second player re-claims a still-open enabled slot.
5. Both mark Ready; host confirms the Start button only enables once both are ready; edit a name after being ready and confirm it un-readies that slot.
6. Host clicks "เริ่มเกม" — confirm the resulting `campaign` phase's room doc has exactly the enabled-party count of slots (not 6), and that the AI-locked, never-claimed party is present in that array as an AI-controlled party (its `claimedBy` is `null`).
7. Mid-lobby, click "รีเซ็ตเกม" (Group A's reset button — available once you've reached `campaign`, since the reset button lives in `topbar()` which only renders past `lobby`) and confirm on return to lobby that the `enabled`/`aiLocked` choices from before are still in effect but every `ready` flag is back to false.

- [ ] **Step 2: Check for console errors**

Open devtools console in both tabs throughout Step 1. Expected: no uncaught exceptions.

- [ ] **Step 3: Final commit (only if Step 1/2 surfaced fixups)**

If no fixups were needed, this task requires no commit. If a bug surfaced, fix it, re-run Step 1/2, then:

```bash
git add "index (3).html"
git commit -m "Fix regression found in Group B end-to-end verification"
```
