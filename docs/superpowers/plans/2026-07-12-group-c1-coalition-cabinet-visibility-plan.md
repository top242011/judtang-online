# Group C1 Coalition/Cabinet Visibility Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Show real player names in the coalition-negotiation screen, and give coalition partners a live read-only view of cabinet-portfolio allocation instead of a blank waiting box — per [docs/superpowers/specs/2026-07-12-group-c1-coalition-cabinet-visibility-design.md](../specs/2026-07-12-group-c1-coalition-cabinet-visibility-design.md).

**Architecture:** Same single-file `index (3).html`, no build step, Firestore as the only backend. This plan builds on Group A and Group B (merged into `main` at commit `3df77d4`) and introduces no new room-state fields — both features are read-only presentation over data that already exists (`playerDisplayName`, `r.cabinet`).

**Tech Stack:** Vanilla JS, Firebase JS SDK 10.7.1 (compat), Firestore. No automated test framework exists in this project — verification is manual, two browser tabs simulating two players, exactly as used in Group A and Group B.

## Global Constraints

- Do not introduce a build step, bundler, package.json, or any new dependency.
- Do not add, rename, or remove any room-state field — this plan is presentation-only.
- All new user-facing strings must be in Thai, matching the existing tone.
- The cabinet read-only view must not let a non-PM coalition partner change any assignment — no `<select>`, no "จัดสรรอัตโนมัติ" button, no "แต่งตั้งคณะรัฐมนตรี" button in that view. Decision authority stays PM-only; this plan only adds visibility.
- Function names introduced must match exactly across tasks: `ministryRow`, `partyWeightSummaryRows` (Task 2 defines both; both call sites — PM view and non-PM view — must use them, not duplicate the markup inline).

---

### Task 1: Show player names in the coalition-negotiation screen

**Files:**
- Modify: `index (3).html` — `PHASE_TEMPLATES.coalition`

**Interfaces:**
- Consumes: `s.playerDisplayName`, `pmSlot.playerDisplayName` (pre-existing fields, set by `claimSlot()`, unchanged by this task)

- [ ] **Step 1: Replace the generic "คน" tag with the actual player name in each party row**

Find this exact block:

```javascript
        <div class="pname">${s.name} <span class="tag ${tag.cls}">${tag.label}</span> ${s.claimedBy?'<span class="tag human">คน</span>':'<span class="tag ai">AI</span>'}</div>
```

Replace with:

```javascript
        <div class="pname">${s.name} <span class="tag ${tag.cls}">${tag.label}</span> ${s.claimedBy?`<span class="tag human">${s.playerDisplayName||'ไม่ทราบชื่อ'}</span>`:'<span class="tag ai">AI</span>'}</div>
```

- [ ] **Step 2: Show the PM's player name in the doc-sub line**

Find this exact block:

```javascript
    <div class="doc-sub">พรรคแกนนำ: <strong>${pmSlot.name}</strong>${pmSlot.claimedBy?'':' (AI ดำเนินการอัตโนมัติ)'} มี ${pmSeats} ที่นั่ง &middot; ต้องการรวม ${MAJORITY} ที่นั่ง</div>
```

Replace with:

```javascript
    <div class="doc-sub">พรรคแกนนำ: <strong>${pmSlot.name}</strong>${pmSlot.claimedBy?` (ผู้เล่น: ${pmSlot.playerDisplayName||'ไม่ทราบชื่อ'})`:' (AI ดำเนินการอัตโนมัติ)'} มี ${pmSeats} ที่นั่ง &middot; ต้องการรวม ${MAJORITY} ที่นั่ง</div>
```

- [ ] **Step 3: Manually verify**

```bash
cd "/Users/panupanpitak/Projects/judtang-online" && python3 -m http.server 8790
```

Open `http://localhost:8790/index%20(3).html` via the Claude_Browser MCP tools. Using two tabs (distinct `localStorage.setItem('jadtang_pid','p_faketestplayer2')` on the second before it loads), play through campaign/election until you reach `coalition` phase with both tabs' parties having seats and needing negotiation (set policies far apart between the two human parties so neither wins an outright majority). Confirm:
- The doc-sub line for the PM party shows "(ผู้เล่น: ‹your name›)" when the PM is a human-claimed party.
- Any other human-claimed party row shows that player's actual entered name instead of a generic "คน" tag.
- An AI-controlled (unclaimed) party row still shows the "AI" tag, unaffected.

- [ ] **Step 4: Commit**

```bash
git add "index (3).html"
git commit -m "Show real player names in the coalition-negotiation screen"
```

---

### Task 2: Read-only cabinet-allocation view for coalition partners

**Files:**
- Modify: `index (3).html` — `PHASE_TEMPLATES.cabinet`; add `ministryRow()` and `partyWeightSummaryRows()`

**Interfaces:**
- Produces: `ministryRow(m, r, parties, editable) -> string`, `partyWeightSummaryRows(r, parties, assignedWeight) -> string` — both called from both the PM and non-PM branches of `PHASE_TEMPLATES.cabinet` in this same task (no cross-task dependency, this task is self-contained).

- [ ] **Step 1: Add the two shared rendering helpers**

Find this exact block (the start of `PHASE_TEMPLATES.cabinet`):

```javascript
PHASE_TEMPLATES.cabinet=function(){
  const r=ui.room; const mine=mySlot();
  const pmSlot=r.slots.find(s=>s.id===r.pm);
  const isPM=mine&&mine.id===r.pm;
  const parties=r.coalitionSlots.map(id=>r.slots.find(s=>s.id===id));
  const assignedWeight={}; parties.forEach(p=>assignedWeight[p.id]=0);
  MINISTRIES.forEach(m=>{ const owner=r.cabinet[m.id]; if(owner) assignedWeight[owner]=(assignedWeight[owner]||0)+m.weight; });
  const allAssigned=MINISTRIES.every(m=>r.cabinet[m.id]);

  if(!isPM){
    return `${topbar()}${breadcrumb('cabinet')}<div class="doc"><h2>รอการแต่งตั้งคณะรัฐมนตรี</h2>
      <div class="waiting-box">${pmSlot.claimedBy?('พรรค '+pmSlot.name+' กำลังจัดสรรตำแหน่งรัฐมนตรี...'):'AI กำลังจัดสรรตำแหน่งรัฐมนตรี...'}</div></div>`;
  }
  return `
  ${topbar()}${breadcrumb('cabinet')}
  <div class="doc">
    <h2>คำสั่งแต่งตั้งคณะรัฐมนตรี</h2>
    <div class="doc-sub">จัดสรรตำแหน่งรัฐมนตรี ${MINISTRIES.length} กระทรวงให้พรรคร่วมรัฐบาล ★ แทนระดับความสำคัญของกระทรวง</div>
    ${MINISTRIES.map(m=>`<div class="ministry-row">
      <div class="mname">${m.name}</div>
      <div class="stars">${'★'.repeat(m.weight)}${'☆'.repeat(3-m.weight)}</div>
      <select onchange="assignMinistry('${m.id}',this.value)">
        <option value="">— เลือกพรรค —</option>
        ${parties.map(p=>`<option value="${p.id}" ${r.cabinet[m.id]===p.id?'selected':''}>${p.name}</option>`).join('')}
      </select>
    </div>`).join('')}
    <div class="btn-row left"><button class="btn btn-ghost-dark" onclick="autoSuggestCabinet()">จัดสรรอัตโนมัติตามสัดส่วนที่นั่ง</button></div>
    <div style="margin-top:16px;">
      ${parties.map(p=>{const share=(assignedWeight[p.id]||0); const seats=(r.lastElection.results.find(x=>x.id===p.id)?.seats||0);
        return `<div style="font-size:12.5px;margin-bottom:4px;"><span class="swatch" style="background:${p.color};display:inline-block;width:9px;height:9px;border-radius:50%;margin-right:6px;"></span>${p.name}: ${share}/${TOTAL_WEIGHT} หน่วยน้ำหนักกระทรวง (${seats} ที่นั่ง)</div>`;}).join('')}
    </div>
    <div class="btn-row"><button class="btn btn-stamp" ${allAssigned?'':'disabled'} onclick="confirmCabinet()">แต่งตั้งคณะรัฐมนตรี</button></div>
  </div>
  `;
};
```

Replace with:

```javascript
function ministryRow(m,r,parties,editable){
  const owner=r.cabinet[m.id];
  const ownerParty=owner?parties.find(p=>p.id===owner):null;
  return `<div class="ministry-row">
    <div class="mname">${m.name}</div>
    <div class="stars">${'★'.repeat(m.weight)}${'☆'.repeat(3-m.weight)}</div>
    ${editable?`<select onchange="assignMinistry('${m.id}',this.value)">
        <option value="">— เลือกพรรค —</option>
        ${parties.map(p=>`<option value="${p.id}" ${owner===p.id?'selected':''}>${p.name}</option>`).join('')}
      </select>`
      :`<div style="font-size:13px;color:${ownerParty?'var(--ink)':'var(--ink-soft)'};">${ownerParty?ownerParty.name:'ยังไม่ได้กำหนด'}</div>`}
  </div>`;
}
function partyWeightSummaryRows(r,parties,assignedWeight){
  return parties.map(p=>{
    const share=(assignedWeight[p.id]||0);
    const seats=(r.lastElection.results.find(x=>x.id===p.id)?.seats||0);
    return `<div style="font-size:12.5px;margin-bottom:4px;"><span class="swatch" style="background:${p.color};display:inline-block;width:9px;height:9px;border-radius:50%;margin-right:6px;"></span>${p.name}: ${share}/${TOTAL_WEIGHT} หน่วยน้ำหนักกระทรวง (${seats} ที่นั่ง)</div>`;
  }).join('');
}
PHASE_TEMPLATES.cabinet=function(){
  const r=ui.room; const mine=mySlot();
  const pmSlot=r.slots.find(s=>s.id===r.pm);
  const isPM=mine&&mine.id===r.pm;
  const parties=r.coalitionSlots.map(id=>r.slots.find(s=>s.id===id));
  const assignedWeight={}; parties.forEach(p=>assignedWeight[p.id]=0);
  MINISTRIES.forEach(m=>{ const owner=r.cabinet[m.id]; if(owner) assignedWeight[owner]=(assignedWeight[owner]||0)+m.weight; });
  const allAssigned=MINISTRIES.every(m=>r.cabinet[m.id]);

  if(!isPM){
    return `${topbar()}${breadcrumb('cabinet')}
    <div class="doc">
      <h2>การจัดสรรคณะรัฐมนตรี</h2>
      <div class="doc-sub">กำลังดำเนินการโดย ${pmSlot.name}${pmSlot.claimedBy?' (ผู้เล่น: '+(pmSlot.playerDisplayName||'ไม่ทราบชื่อ')+')':' (AI ดำเนินการอัตโนมัติ)'}</div>
      ${MINISTRIES.map(m=>ministryRow(m,r,parties,false)).join('')}
      <div style="margin-top:16px;">
        ${partyWeightSummaryRows(r,parties,assignedWeight)}
      </div>
      <div class="waiting-box">รอ ${pmSlot.name} ยืนยันคณะรัฐมนตรี...</div>
    </div>
    `;
  }
  return `
  ${topbar()}${breadcrumb('cabinet')}
  <div class="doc">
    <h2>คำสั่งแต่งตั้งคณะรัฐมนตรี</h2>
    <div class="doc-sub">จัดสรรตำแหน่งรัฐมนตรี ${MINISTRIES.length} กระทรวงให้พรรคร่วมรัฐบาล ★ แทนระดับความสำคัญของกระทรวง</div>
    ${MINISTRIES.map(m=>ministryRow(m,r,parties,true)).join('')}
    <div class="btn-row left"><button class="btn btn-ghost-dark" onclick="autoSuggestCabinet()">จัดสรรอัตโนมัติตามสัดส่วนที่นั่ง</button></div>
    <div style="margin-top:16px;">
      ${partyWeightSummaryRows(r,parties,assignedWeight)}
    </div>
    <div class="btn-row"><button class="btn btn-stamp" ${allAssigned?'':'disabled'} onclick="confirmCabinet()">แต่งตั้งคณะรัฐมนตรี</button></div>
  </div>
  `;
};
```

(This replaces the PM branch's inline ministry-row and weight-summary markup with calls to the two new helpers — the rendered output for the PM view is unchanged, only the source is de-duplicated. The non-PM branch changes from a blank waiting box to the same information rendered read-only.)

- [ ] **Step 2: Manually verify with two tabs**

Using the local server from Task 1 (or a fresh one on a different port if that one's no longer running), with two tabs on a room that has reached `cabinet` phase (host is PM, second player is a coalition partner — you may need to directly seed `phase:'cabinet'`, `coalitionSlots`, and `lastElection` via the Firestore console if steering a real campaign/coalition flow there is slow, exactly as done in earlier Group A/B verification):

1. In the non-PM tab, confirm you see: a heading "การจัดสรรคณะรัฐมนตรี", a doc-sub naming the PM party (and player name if human), all 12 ministries with their star ratings, and — for each ministry — either "ยังไม่ได้กำหนด" or the name of whichever party currently holds it. Confirm there is NO `<select>`, no "จัดสรรอัตโนมัติ" button, and no "แต่งตั้งคณะรัฐมนตรี" button anywhere in this view.
2. In the PM tab, assign a few ministries via the dropdowns (or click "จัดสรรอัตโนมัติตามสัดส่วนที่นั่ง"). Confirm the non-PM tab's view updates within about a second, without needing a page reload, showing the newly assigned party names.
3. Confirm the per-party weight summary line (`X/${TOTAL_WEIGHT} หน่วยน้ำหนักกระทรวง (Y ที่นั่ง)`) matches exactly between the PM view and the non-PM view at every point during the assignment process.
4. In the PM tab, finish assigning all 12 ministries and click "แต่งตั้งคณะรัฐมนตรี" — confirm both tabs transition to `governing` phase correctly (sanity check that this refactor didn't break the existing confirm flow).

- [ ] **Step 3: Commit**

```bash
git add "index (3).html"
git commit -m "Show coalition partners a live read-only view of cabinet allocation"
```

---

### Task 3: Regression pass across both Group C1 features

**Files:**
- None (verification only)

- [ ] **Step 1: Play through one full election → coalition → cabinet flow with two human tabs**

Using two tabs (host + second player), steer policies so neither wins an outright majority, negotiate a coalition (accept the invite from the non-PM tab), then proceed into cabinet:

1. Confirm player names show correctly throughout the coalition screen (Task 1) for both the PM row and the accepted-partner row.
2. Confirm the non-PM tab's read-only cabinet view (Task 2) tracks the PM's live assignments correctly all the way through to `governing` phase.
3. Confirm no console errors appear in either tab throughout (`mcp__Claude_Browser__read_console_messages`).

- [ ] **Step 2: Commit (only if a fixup was needed)**

If no fixups were needed, this task requires no commit. If a bug surfaced, fix it, re-run Step 1, then:

```bash
git add "index (3).html"
git commit -m "Fix regression found in Group C1 end-to-end verification"
```
