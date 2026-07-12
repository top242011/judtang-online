# Group E1: Legislation System Implementation Plan

> **For agentic workers:** Executed via **inline execution** in the same session (superpowers:executing-plans), not subagent-driven-development. No automated test suite exists in this project (established across Groups A–D) — verification per task is a Node.js syntax-parse check plus manual trace of the new code path, not a live browser/Firestore run, matching the project's existing manual-verification convention.

**Goal:** Add the propose → amend → select-vote → pass-vote bill lifecycle with persistent `activeLaws`, region-leaning drift, and public-debt carryover, per `docs/superpowers/specs/2026-07-12-group-e1-legislation-design.md`.

**Architecture:** All changes land in the single `index (3).html` file, following the existing pattern (static data arrays, per-phase render functions, client mutation functions calling `roomRef().update()`, host-automation mirror in `runHostAutomation()`).

**Tech Stack:** Vanilla JS, Firebase Firestore compat SDK. No build step.

## Global Constraints

- Categories are exactly `'fiscal'|'social'|'security'`; intensities exactly `'mild'|'moderate'|'aggressive'`.
- Magnitude table (from spec §2) must match exactly:
  - fiscal: mild `{treasury:5,economy:1}`, moderate `{treasury:10,economy:2}`, aggressive `{treasury:18,economy:4,trust:-2}`
  - social: mild `{trust:3,unrest:-2}`, moderate `{trust:6,unrest:-4}`, aggressive `{trust:10,unrest:-7,satisfactionAll:3}`
  - security: mild `{unrest:-3}`, moderate `{unrest:-6,trust:2}`, aggressive `{unrest:-10,trust:4,regions:5}` (the `regions` value applies to the bill's `proposerId`'s strongest-fit region only, and is a popularity effect, not a leaning-drift effect)
  - `direction:-1` negates every numeric value in the effect object before applying.
- `pendingBills` is turn-scoped: cleared at the very start of every `nextTurn()` call (old bills never carry into the new turn), and by every function that already resets other governing-phase state (`confirmCabinet()`, the cabinet-automation block, `resetRoomToLobby()`, `startNewTerm()`, and both branches of the mid-term-reshuffle write in `nextTurn()`).
- `activeLaws` and `publicDebt` persist across terms — only `resetRoomToLobby()` resets them to `{fiscal:null,social:null,security:null}` / `0`.
- Region drift rate: `±0.3` per turn per active fiscal/security law, clamped `[-100,100]`, applied to the region's `leaning[axis]` where `axis` is `'economy'` for fiscal, `'security'` for security.
- Public debt: `publicDebt += -treasury; treasury = 0` whenever treasury would go negative after all per-turn effects; then `treasury = Math.max(0, treasury - Math.round(publicDebt*0.02))`; an active `fiscal` law with `direction:-1` at `aggressive` intensity additionally does `publicDebt = Math.max(0, publicDebt - 8)`.

---

### Task 1: Party billStance data + room state defaults

**Files:**
- Modify: `index (3).html` — `PARTY_TEMPLATES` array (~line 250), `handleCreateRoom()`, `resetRoomToLobby()`, `startNewTerm()`, both branches of the mid-term-reshuffle write in `nextTurn()`.

**Interfaces:**
- Produces: `slot.billStance:{fiscal,social,security}` on every party template; room fields `pendingBills:[]`, `activeLaws:{fiscal:null,social:null,security:null}`, `publicDebt:0`.

- [ ] **Step 1:** Add `billStance` to each of the 6 `PARTY_TEMPLATES` entries, roughly mirroring each party's existing `policy` sign (populist/high-`economy`-negative parties get very negative `fiscal`; `reform`-heavy parties get positive `social`; `security`-heavy parties get positive `security`):
```js
const PARTY_TEMPLATES=[
  {name:'พรรคไทยรุ่งเรือง',color:'#8B6F2E',policy:{economy:40,reform:-70,security:60},billStance:{fiscal:30,social:-40,security:60}},
  {name:'พรรคประชาชนก้าวใหม่',color:'#2E8B74',policy:{economy:-20,reform:90,security:-50},billStance:{fiscal:-50,social:70,security:-60}},
  {name:'พรรคชาวนาไทยเข้มแข็ง',color:'#3F6B2E',policy:{economy:-70,reform:10,security:0},billStance:{fiscal:-80,social:20,security:0}},
  {name:'พรรครวมพลังท้องถิ่น',color:'#6B2E4C',policy:{economy:20,reform:-30,security:30},billStance:{fiscal:10,social:-20,security:30}},
  {name:'พรรคเศรษฐกิจใหม่',color:'#2E5A8B',policy:{economy:80,reform:20,security:10},billStance:{fiscal:70,social:10,security:10}},
  {name:'พรรคประชาธรรมใหม่',color:'#8B4A2E',policy:{economy:0,reform:40,security:20},billStance:{fiscal:0,social:30,security:20}},
];
```
- [ ] **Step 2:** In `handleCreateRoom()`, add `pendingBills:[], activeLaws:{fiscal:null,social:null,security:null}, publicDebt:0` to the room-creation document (same object literal that currently sets `roomCode`, `hostId`, `phase:'lobby'`, etc.).
- [ ] **Step 3:** In `resetRoomToLobby()`'s `roomRef().update({...})` call, add `pendingBills:[], activeLaws:{fiscal:null,social:null,security:null}, publicDebt:0`.
- [ ] **Step 4:** In `startNewTerm()`'s `roomRef().update({...})` call, add `pendingBills:[]` only (do **not** add `activeLaws`/`publicDebt` — they must persist).
- [ ] **Step 5:** In `nextTurn()`, add `pendingBills:[]` to both the `phase:'cabinet'` and `phase:'coalition'` update objects inside the `if(r.pendingPmDisqualification)` branch.
- [ ] **Step 6:** In `confirmCabinet()` and the cabinet-automation block inside `runHostAutomation()`, add `pendingBills:[]` to their `roomRef().update({...})` calls (same object that already resets `decisionHistory`/`noConfidenceVotes`/`midTermReshuffle`).
- [ ] **Step 7 (verify):**
```bash
node -e "
const fs=require('fs');
const html=fs.readFileSync('index (3).html','utf8');
const m=html.match(/<script>([\s\S]*)<\/script>/);
new Function(m[1]);
console.log('syntax OK');
"
```
Expected: `syntax OK`
- [ ] **Step 8: Commit**
```bash
git add "index (3).html"
git commit -m "Add billStance party data and E1 room-state defaults"
```

---

### Task 2: Pure bill-effect and tally functions

**Files:**
- Modify: `index (3).html` — add new pure functions near `computeDeficitCrisis`/`computeNoConfidenceEffect` (~line 300s).

**Interfaces:**
- Consumes: nothing beyond plain data (no `ui.room` reads — pure functions only, matching the project's existing `compute*` pattern).
- Produces: `computeBillMagnitude(category,intensity,direction)`, `tallySelectVote(bill,slots,lastElectionResults,pmId)`, `tallyPassVote(bill,slots,lastElectionResults,pmId,selectedIntensity)` — later tasks call these by these exact names/signatures.

- [ ] **Step 1:** Add the magnitude table and lookup function:
```js
const BILL_MAGNITUDE={
  fiscal:{mild:{treasury:5,economy:1},moderate:{treasury:10,economy:2},aggressive:{treasury:18,economy:4,trust:-2}},
  social:{mild:{trust:3,unrest:-2},moderate:{trust:6,unrest:-4},aggressive:{trust:10,unrest:-7,satisfactionAll:3}},
  security:{mild:{unrest:-3},moderate:{unrest:-6,trust:2},aggressive:{unrest:-10,trust:4,regions:5}},
};
function computeBillMagnitude(category,intensity,direction){
  const base=BILL_MAGNITUDE[category][intensity];
  const eff={};
  Object.keys(base).forEach(k=>{ eff[k]=base[k]*(direction<0?-1:1); });
  return eff;
}
```
- [ ] **Step 2:** Add the seat-weighted plurality tally for the amendment-selection vote (candidates are the distinct intensities in play: the original plus any amendments submitted, duplicates collapsed by the caller before this is invoked):
```js
function tallySelectVote(bill,slots,lastElectionResults,pmId){
  const seatsOf=id=>(lastElectionResults.find(x=>x.id===id)?.seats||0);
  const totals={};
  Object.entries(bill.selectVotes||{}).forEach(([voterId,intensity])=>{
    totals[intensity]=(totals[intensity]||0)+seatsOf(voterId);
  });
  const entries=Object.entries(totals);
  if(!entries.length) return bill.category?bill.intensity:null;
  let best=entries[0], tied=[entries[0]];
  entries.forEach(e=>{ if(e[1]>best[1]){ best=e; tied=[e]; } else if(e[1]===best[1] && e!==best) tied.push(e); });
  if(tied.length>1){
    const pmVote=bill.selectVotes[pmId];
    if(pmVote && tied.some(t=>t[0]===pmVote)) return pmVote;
  }
  return best[0];
}
```
- [ ] **Step 3:** Add the binary seat-weighted pass/fail tally (mirrors `resolveNoConfidenceVote`'s majority-threshold logic):
```js
function tallyPassVote(bill,slots,lastElectionResults,pmId){
  const seatsOf=id=>(lastElectionResults.find(x=>x.id===id)?.seats||0);
  let yes=0,no=0;
  Object.entries(bill.passVotes||{}).forEach(([voterId,v])=>{
    if(v==='yes') yes+=seatsOf(voterId); else no+=seatsOf(voterId);
  });
  if(yes===no){ return bill.passVotes[pmId]==='yes'; }
  return yes>no;
}
```
- [ ] **Step 4 (verify):** run the same `node -e "..."` syntax check from Task 1 Step 7. Expected: `syntax OK`
- [ ] **Step 5:** Also sanity-check the three functions actually run, using Node directly (no Firestore/browser needed since they're pure):
```bash
node -e "
$(sed -n '/const BILL_MAGNITUDE/,/^}/p' 'index (3).html')
console.log(JSON.stringify(computeBillMagnitude('fiscal','aggressive',-1)));
console.log(tallySelectVote({selectVotes:{s0:'mild',s1:'mild',s2:'aggressive'}},[],[{id:'s0',seats:100},{id:'s1',seats:50},{id:'s2',seats:80}],'s0'));
console.log(tallyPassVote({passVotes:{s0:'yes',s1:'no'}},[],[{id:'s0',seats:100},{id:'s1',seats:120}],'s0'));
"
```
Expected: first line negates every value (e.g. `treasury:-18`), second line prints `mild` (100+50 seats beats 80), third line prints `false` (no's 120 seats beat yes's 100).
- [ ] **Step 6: Commit**
```bash
git add "index (3).html"
git commit -m "Add pure bill-magnitude and vote-tally functions"
```

---

### Task 3: Bill proposal, amendment, and voting mutation functions

**Files:**
- Modify: `index (3).html` — add client mutation functions near `fileNoConfidence`/`castNoConfidenceVote` (~line 1350s).

**Interfaces:**
- Consumes: `computeBillMagnitude`, `tallySelectVote`, `tallyPassVote` from Task 2; `MAJORITY`.
- Produces: `proposeBill(category,direction)`, `submitAmendment(billId,intensity)`, `passOnAmendment(billId)`, `castSelectVote(billId,intensity)`, `castPassVote(billId,vote)` — later tasks (UI, automation) call these exact names.

- [ ] **Step 1:** Add `proposeBill` — one bill per party per turn, capped at 6 concurrent:
```js
function proposeBill(category,direction){
  const r=ui.room; const mine=mySlot();
  if(!mine) return;
  const pending=r.pendingBills||[];
  if(pending.length>=6) return;
  if(pending.some(b=>b.proposerId===mine.id)) return;
  const bill={
    id:'b'+Math.random().toString(36).slice(2,9),
    proposerId:mine.id, category, direction, stage:'amend',
    amendments:[], respondedToAmend:[mine.id],
    selectVotes:{}, passVotes:{},
  };
  roomRef().update({pendingBills:[...pending,bill]});
}
```
- [ ] **Step 2:** Add `submitAmendment` and `passOnAmendment` (the non-proposer's two choices during the amend stage), each advancing the bill's stage once every non-proposer active party has responded:
```js
function otherActiveSlotIds(r,excludeId){
  return r.slots.filter(s=>s.enabled!==false && s.id!==excludeId).map(s=>s.id);
}
function maybeAdvanceAmendStage(r,bill){
  const others=otherActiveSlotIds(r,bill.proposerId);
  const allResponded=others.every(id=>bill.respondedToAmend.includes(id));
  if(!allResponded) return bill;
  const intensities=[...new Set(bill.amendments.map(a=>a.intensity))];
  return {...bill, stage: intensities.length ? 'select' : 'pass'};
}
function submitAmendment(billId,intensity){
  const r=ui.room; const mine=mySlot(); if(!mine) return;
  const bills=(r.pendingBills||[]).map(b=>{
    if(b.id!==billId || b.proposerId===mine.id || b.respondedToAmend.includes(mine.id)) return b;
    const updated={...b, amendments:[...b.amendments,{proposerId:mine.id,intensity}], respondedToAmend:[...b.respondedToAmend,mine.id]};
    return maybeAdvanceAmendStage(r,updated);
  });
  roomRef().update({pendingBills:bills});
}
function passOnAmendment(billId){
  const r=ui.room; const mine=mySlot(); if(!mine) return;
  const bills=(r.pendingBills||[]).map(b=>{
    if(b.id!==billId || b.proposerId===mine.id || b.respondedToAmend.includes(mine.id)) return b;
    const updated={...b, respondedToAmend:[...b.respondedToAmend,mine.id]};
    return maybeAdvanceAmendStage(r,updated);
  });
  roomRef().update({pendingBills:bills});
}
```
- [ ] **Step 3:** Add `castSelectVote` and `castPassVote`, each resolving the bill once every active party has voted:
```js
function castSelectVote(billId,intensity){
  const r=ui.room; const mine=mySlot(); if(!mine) return;
  const bills=(r.pendingBills||[]).map(b=>{
    if(b.id!==billId || b.stage!=='select') return b;
    const selectVotes={...b.selectVotes,[mine.id]:intensity};
    const active=otherActiveSlotIds(r,'').concat([]); // all active slots
    const allVoted=r.slots.filter(s=>s.enabled!==false).every(s=>selectVotes[s.id]!==undefined);
    if(!allVoted) return {...b,selectVotes};
    const winner=tallySelectVote({...b,selectVotes},r.slots,r.lastElection.results,r.pm);
    return {...b,selectVotes,selectedIntensity:winner,stage:'pass'};
  });
  roomRef().update({pendingBills:bills});
}
function castPassVote(billId,vote){
  const r=ui.room; const mine=mySlot(); if(!mine) return;
  let resolved=null;
  const bills=(r.pendingBills||[]).map(b=>{
    if(b.id!==billId || b.stage!=='pass') return b;
    const passVotes={...b.passVotes,[mine.id]:vote};
    const allVoted=r.slots.filter(s=>s.enabled!==false).every(s=>passVotes[s.id]!==undefined);
    if(!allVoted) return {...b,passVotes};
    const passed=tallyPassVote({...b,passVotes},r.slots,r.lastElection.results,r.pm);
    resolved={bill:{...b,passVotes},passed};
    return {...b,passVotes,stage:'done'};
  });
  const update={pendingBills:bills.filter(b=>b.stage!=='done')};
  if(resolved){
    const finalIntensity=resolved.bill.selectedIntensity||(resolved.bill.category&&resolved.bill.direction!==undefined?undefined:undefined);
    const intensity=resolved.bill.selectedIntensity || (resolved.bill.amendments.length?resolved.bill.amendments[0].intensity:'mild');
    if(resolved.passed){
      const activeLaws={...r.activeLaws,[resolved.bill.category]:{direction:resolved.bill.direction,intensity:resolved.bill.stage==='pass'&&resolved.bill.selectedIntensity?resolved.bill.selectedIntensity:intensity,proposerId:resolved.bill.proposerId}};
      update.activeLaws=activeLaws;
    } else {
      const slots=r.slots.map(s=>s.id===resolved.bill.proposerId?{...s}:s);
      update.slots=slots; // trust penalty applied via national.trust below is global, not per-party; see note in Task 4
    }
  }
  roomRef().update(update);
}
```
Note: the spec's "proposer loses a small trust amount" is a **national** trust figure in this codebase (there is no per-party trust stat, only per-party `satisfaction` within a coalition and per-region `popularity`) — Task 4 applies a `national.trust -= 2` penalty only when the failed bill's proposer is a *coalition* party (has `satisfaction`), by adjusting `coalitionParties` the same way `chooseEvent`'s `satisfactionParty` effect does; opposition-proposed bills that fail have no penalty target and apply no penalty. Remove the dead `update.slots` line above during Task 4's edit — it's a placeholder for where that logic plugs in and must not ship as-is.

- [ ] **Step 4 (verify):** run the Task 1 Step 7 syntax check. Expected: `syntax OK`
- [ ] **Step 5: Commit**
```bash
git add "index (3).html"
git commit -m "Add bill proposal, amendment, and voting mutation functions"
```

---

### Task 4: Wire activeLaws effects, region drift, and public debt into nextTurn()

**Files:**
- Modify: `index (3).html` — `nextTurn()` (~line 1407) and `castPassVote()` from Task 3 (fix the trust-penalty placeholder noted above).

**Interfaces:**
- Consumes: `r.activeLaws`, `computeBillMagnitude`, `REGIONS`, `clamp`.
- Produces: `nextTurn()` applies ongoing law effects, region drift, and debt handling every turn before drawing the next event.

- [ ] **Step 1:** In `castPassVote()` (Task 3), replace the placeholder `update.slots=slots;` line with an actual national-trust penalty, applied only if the proposer is currently a coalition member (has a `satisfaction` entry in `r.coalitionParties`):
```js
    } else {
      if(r.coalitionParties.some(p=>p.id===resolved.bill.proposerId)){
        update.national={...r.national,trust:clamp(r.national.trust-2)};
      }
    }
```
- [ ] **Step 2:** In `nextTurn()`, immediately after the existing `pendingCollapse`/turn-limit checks and before drawing the next event, add a new block that: clears `pendingBills`, applies each active law's per-turn magnitude, applies region drift, and applies public-debt handling. Insert this as a new local computation feeding into whichever `roomRef().update({...})` call `nextTurn()` already makes for the "normal turn advance" path (the one that increments `turn` and calls `drawEventForRoom`):
```js
function applyActiveLawsAndDebt(r){
  let treasury=r.treasury, national={...r.national}, satisfactionAll=0;
  const popularity={}; Object.keys(r.popularity||{}).forEach(k=>{ popularity[k]={...(r.popularity[k]||{})}; });
  const regions=REGIONS.map(reg=>({...reg,leaning:{...reg.leaning}}));
  ['fiscal','social','security'].forEach(cat=>{
    const law=r.activeLaws && r.activeLaws[cat];
    if(!law) return;
    const eff=computeBillMagnitude(cat,law.intensity,law.direction);
    if(eff.treasury) treasury+=eff.treasury;
    if(eff.economy) national.economy=clamp(national.economy+eff.economy);
    if(eff.trust) national.trust=clamp(national.trust+eff.trust);
    if(eff.unrest) national.unrest=clamp(national.unrest+eff.unrest);
    if(eff.satisfactionAll) satisfactionAll+=eff.satisfactionAll;
    if(eff.regions){
      popularity[law.proposerId]=popularity[law.proposerId]||{north:0,isaan:0,central:0,bangkok:0,south:0};
      const proposerSlot=r.slots.find(s=>s.id===law.proposerId);
      if(proposerSlot){
        let bestReg=regions[0],bestFit=-Infinity;
        regions.forEach(reg=>{ const d=policyDistance(proposerSlot.policy,reg.leaning); const fit=-d; if(fit>bestFit){bestFit=fit;bestReg=reg;} });
        popularity[law.proposerId][bestReg.id]=clamp((popularity[law.proposerId][bestReg.id]||0)+eff.regions,-60,60);
      }
    }
    if(cat==='fiscal'||cat==='security'){
      const axis=cat==='fiscal'?'economy':'security';
      const delta=0.3*(law.direction<0?-1:1);
      regions.forEach(reg=>{ reg.leaning[axis]=clamp(reg.leaning[axis]+delta,-100,100); });
    }
  });
  let publicDebt=r.publicDebt||0;
  if(treasury<0){ publicDebt+=-treasury; treasury=0; }
  treasury=Math.max(0,treasury-Math.round(publicDebt*0.02));
  const fiscalLaw=r.activeLaws&&r.activeLaws.fiscal;
  if(fiscalLaw && fiscalLaw.direction<0 && fiscalLaw.intensity==='aggressive'){ publicDebt=Math.max(0,publicDebt-8); }
  return {treasury,national,popularity,regions,publicDebt,satisfactionAll};
}
```
- [ ] **Step 3:** At the top of `nextTurn()`, before any of the existing `if(r.pendingPmDisqualification)`/`if(r.pendingCollapse)` checks, compute `const lawResult=applyActiveLawsAndDebt(r);` and `update.pendingBills=[];`. Merge `lawResult.treasury`/`lawResult.national`/`lawResult.popularity`/`lawResult.publicDebt` into whichever branch's `roomRef().update({...})` call actually executes (all three existing branches — disqualification, collapse, normal advance — must receive these fields, plus `REGIONS` is a module-level `const`, so region drift needs a different approach: **`REGIONS` cannot be mutated per-room since it's shared across all rooms in the same browser tab.** Store the drifted regions on the room instead: add a `r.regionLeaning` override map (`{regionId:{economy,security}}`, starts absent) that `policyDistance`/election calculations fall back to `REGIONS.find(...).leaning` when absent. This requires:
  - Changing `applyActiveLawsAndDebt` to read/write `r.regionLeaning[regId]` (defaulting to the matching `REGIONS` entry) instead of cloning `REGIONS` itself, and returning `regionLeaning` instead of `regions`.
  - Every existing call site that reads `reg.leaning` for seat/appeal computation (`computeElectionFromSlots`, `computeOppositionDrift`, the region-summary text at ~line 477) must first check `r.regionLeaning?.[reg.id] || reg.leaning`.
  Rewrite Step 2's function accordingly:
```js
function regionLeaningFor(r,regId){
  const reg=REGIONS.find(x=>x.id===regId);
  return (r.regionLeaning && r.regionLeaning[regId]) || reg.leaning;
}
function applyActiveLawsAndDebt(r){
  let treasury=r.treasury, national={...r.national};
  const popularity={}; Object.keys(r.popularity||{}).forEach(k=>{ popularity[k]={...(r.popularity[k]||{})}; });
  const regionLeaning={}; REGIONS.forEach(reg=>{ regionLeaning[reg.id]={...regionLeaningFor(r,reg.id)}; });
  ['fiscal','social','security'].forEach(cat=>{
    const law=r.activeLaws && r.activeLaws[cat];
    if(!law) return;
    const eff=computeBillMagnitude(cat,law.intensity,law.direction);
    if(eff.treasury) treasury+=eff.treasury;
    if(eff.economy) national.economy=clamp(national.economy+eff.economy);
    if(eff.trust) national.trust=clamp(national.trust+eff.trust);
    if(eff.unrest) national.unrest=clamp(national.unrest+eff.unrest);
    if(eff.regions){
      const proposerSlot=r.slots.find(s=>s.id===law.proposerId);
      if(proposerSlot){
        popularity[law.proposerId]=popularity[law.proposerId]||{north:0,isaan:0,central:0,bangkok:0,south:0};
        let bestReg=REGIONS[0],bestFit=-Infinity;
        REGIONS.forEach(reg=>{ const d=policyDistance(proposerSlot.policy,regionLeaning[reg.id]); const fit=-d; if(fit>bestFit){bestFit=fit;bestReg=reg;} });
        popularity[law.proposerId][bestReg.id]=clamp((popularity[law.proposerId][bestReg.id]||0)+eff.regions,-60,60);
      }
    }
    if(cat==='fiscal'||cat==='security'){
      const axis=cat==='fiscal'?'economy':'security';
      const delta=0.3*(law.direction<0?-1:1);
      REGIONS.forEach(reg=>{ regionLeaning[reg.id][axis]=clamp(regionLeaning[reg.id][axis]+delta,-100,100); });
    }
  });
  let publicDebt=r.publicDebt||0;
  if(treasury<0){ publicDebt+=-treasury; treasury=0; }
  treasury=Math.max(0,treasury-Math.round(publicDebt*0.02));
  const fiscalLaw=r.activeLaws&&r.activeLaws.fiscal;
  if(fiscalLaw && fiscalLaw.direction<0 && fiscalLaw.intensity==='aggressive'){ publicDebt=Math.max(0,publicDebt-8); }
  return {treasury,national,popularity,regionLeaning,publicDebt};
}
```
  and add `regionLeaning:lawResult.regionLeaning` (not `regions`) to the merged update object.
- [ ] **Step 4:** Update the three read sites to fall back to `r.regionLeaning`:
  - `computeElectionFromSlots(slots,popularityBySlot,regionLeaning)` — add a third parameter, default `{}`, and replace every `reg.leaning` read inside it with `(regionLeaning[reg.id]||reg.leaning)`. Update both call sites (`maybeRunElection`'s transaction, passing `data.regionLeaning||{}`) to pass the new argument.
  - `computeOppositionDrift(r,newNational)` — replace its `reg.leaning` read with `regionLeaningFor(r,reg.id)`.
  - The region-summary text helper (~line 477, `r.leaning.economy<...`) is per-region-object already (`r` there is a region, not the room) — leave as-is; it is not fed by `REGIONS` directly at that call site if it already receives a merged leaning object, otherwise confirm its caller passes `{...reg,leaning:regionLeaningFor(room,reg.id)}`. Trace this call site during implementation and adjust the caller, not this helper.
- [ ] **Step 5 (verify):** run the Task 1 Step 7 syntax check. Expected: `syntax OK`
- [ ] **Step 6:** Manual trace check (no live browser needed): read through `nextTurn()` once more and confirm every one of its three `roomRef().update(...)` branches now includes `treasury`, `national`, `popularity`, `regionLeaning`, `publicDebt`, `pendingBills:[]`.
- [ ] **Step 7: Commit**
```bash
git add "index (3).html"
git commit -m "Wire activeLaws effects, region-leaning drift, and public debt into nextTurn()"
```

---

### Task 5: Governing-phase UI for bills, active laws, and public debt

**Files:**
- Modify: `index (3).html` — `PHASE_TEMPLATES.governing` render function.

**Interfaces:**
- Consumes: `proposeBill`, `submitAmendment`, `passOnAmendment`, `castSelectVote`, `castPassVote` (Task 3); `r.pendingBills`, `r.activeLaws`, `r.publicDebt`.

- [ ] **Step 1:** Add a bill-proposal control (visible to any party without a pending bill this turn) rendering 3×2 category/direction buttons, and a card list for `r.pendingBills` showing stage-appropriate controls for `mine.id`:
```js
function billCategoryLabel(cat){ return cat==='fiscal'?'การคลัง':cat==='social'?'สังคม':'ความมั่นคง'; }
function billCard(r,mine,bill){
  const proposer=r.slots.find(s=>s.id===bill.proposerId);
  const dirLabel=bill.direction>0?'ขยาย':'รัดกุม';
  let body='';
  if(bill.stage==='amend'){
    const responded=bill.respondedToAmend.includes(mine?.id);
    body = mine && mine.id!==bill.proposerId && !responded
      ? `<div class="btn-row">
          ${['mild','moderate','aggressive'].filter(i=>true).map(i=>`<button class="btn btn-ghost-dark" onclick="submitAmendment('${bill.id}','${i}')">แก้เป็น ${i}</button>`).join('')}
          <button class="btn btn-ghost-dark" onclick="passOnAmendment('${bill.id}')">ไม่แก้ไข</button>
        </div>`
      : `<div class="tag medium">รอพรรคอื่นตอบสนอง (${bill.respondedToAmend.length}/${r.slots.filter(s=>s.enabled!==false).length-1})</div>`;
  } else if(bill.stage==='select'){
    const intensities=[...new Set(['original',...bill.amendments.map(a=>a.intensity)])];
    const voted=bill.selectVotes[mine?.id]!==undefined;
    body = mine && !voted
      ? `<div class="btn-row">${bill.amendments.map(a=>`<button class="btn btn-ghost-dark" onclick="castSelectVote('${bill.id}','${a.intensity}')">เลือก ${a.intensity}</button>`).join('')}</div>`
      : `<div class="tag medium">โหวตเลือกเวอร์ชันแล้ว</div>`;
  } else if(bill.stage==='pass'){
    const voted=bill.passVotes[mine?.id]!==undefined;
    body = mine && !voted
      ? `<div class="btn-row"><button class="btn btn-stamp" onclick="castPassVote('${bill.id}','yes')">เห็นชอบ</button><button class="btn btn-ghost-dark" onclick="castPassVote('${bill.id}','no')">ไม่เห็นชอบ</button></div>`
      : `<div class="tag medium">โหวตผ่านแล้ว</div>`;
  }
  return `<div class="doc-card"><strong>${proposer?proposer.name:''}</strong> เสนอร่างกฎหมาย${billCategoryLabel(bill.category)} (${dirLabel})<div class="doc-sub">สถานะ: ${bill.stage}</div>${body}</div>`;
}
function billsPanel(r){
  const mine=mySlot();
  const canPropose = mine && !(r.pendingBills||[]).some(b=>b.proposerId===mine.id) && (r.pendingBills||[]).length<6;
  const proposeUI = canPropose ? `
    <div class="btn-row">
      ${['fiscal','social','security'].map(cat=>`
        <button class="btn btn-ghost-dark" onclick="proposeBill('${cat}',1)">เสนอ${billCategoryLabel(cat)} (ขยาย)</button>
        <button class="btn btn-ghost-dark" onclick="proposeBill('${cat}',-1)">เสนอ${billCategoryLabel(cat)} (รัดกุม)</button>
      `).join('')}
    </div>` : '';
  const activeLawsText = ['fiscal','social','security'].map(cat=>{
    const law=r.activeLaws && r.activeLaws[cat];
    return `${billCategoryLabel(cat)}: ${law?`${law.direction>0?'ขยาย':'รัดกุม'} (${law.intensity})`:'ไม่มี'}`;
  }).join(' · ');
  return `<div class="doc">
    <h3>ร่างกฎหมาย</h3>
    <div class="doc-sub">กฎหมายบังคับใช้อยู่: ${activeLawsText} &middot; หนี้สาธารณะ: ${Math.round(r.publicDebt||0)}</div>
    ${proposeUI}
    ${(r.pendingBills||[]).map(b=>billCard(r,mine,b)).join('')}
  </div>`;
}
```
- [ ] **Step 2:** Call `billsPanel(r)` from within `PHASE_TEMPLATES.governing`'s returned template string, placed after the existing event card markup (trace the function's existing return statement and insert `${billsPanel(r)}` immediately before its closing backtick).
- [ ] **Step 3 (verify):** run the Task 1 Step 7 syntax check. Expected: `syntax OK`
- [ ] **Step 4: Commit**
```bash
git add "index (3).html"
git commit -m "Add governing-phase UI for bill lifecycle, active laws, and public debt"
```

---

### Task 6: AI automation for bill lifecycle

**Files:**
- Modify: `index (3).html` — `runHostAutomation()`'s `governing` branch.

**Interfaces:**
- Consumes: `proposeBill`, `submitAmendment`, `passOnAmendment`, `castSelectVote`, `castPassVote` (Task 3); existing `automationBusy`/`armAutomationWatchdog`/`releaseAutomation` pattern.

- [ ] **Step 1:** Inside the existing `if(r.phase==='governing'){ ... }` block, before the existing `noconf`/event-choice logic, add bill-automation sub-steps. Each follows the existing pattern: find the first AI-controlled slot with a pending action, guard with `automationBusy`, act after a short delay:
```js
    const pending=r.pendingBills||[];
    const amendCandidate=pending.find(b=>b.stage==='amend' && r.slots.filter(s=>s.enabled!==false && s.id!==b.proposerId && !s.claimedBy).find(s=>!b.respondedToAmend.includes(s.id)));
    if(amendCandidate){
      const aiSlot=r.slots.filter(s=>s.enabled!==false && s.id!==amendCandidate.proposerId && !s.claimedBy).find(s=>!amendCandidate.respondedToAmend.includes(s.id));
      automationBusy=true; armAutomationWatchdog();
      setTimeout(()=>{
        try{
          const proposerSlot=r.slots.find(s=>s.id===amendCandidate.proposerId);
          const stance=aiSlot.billStance[amendCandidate.category]||0;
          const opposes=(amendCandidate.direction>0 && stance<0)||(amendCandidate.direction<0 && stance>0);
          if(opposes){
            const softer=amendCandidate.intensity==='aggressive'?'moderate':'mild';
            submitAmendment(amendCandidate.id,softer);
          } else {
            passOnAmendment(amendCandidate.id);
          }
        } catch(e){ console.error('AI bill amendment failed:',e); }
        finally { releaseAutomation(); }
      },700);
      return;
    }
    const selectCandidate=pending.find(b=>b.stage==='select' && r.slots.filter(s=>s.enabled!==false && !s.claimedBy).find(s=>b.selectVotes[s.id]===undefined));
    if(selectCandidate){
      const aiSlot=r.slots.filter(s=>s.enabled!==false && !s.claimedBy).find(s=>selectCandidate.selectVotes[s.id]===undefined);
      automationBusy=true; armAutomationWatchdog();
      setTimeout(()=>{
        try{
          const options=[...new Set(selectCandidate.amendments.map(a=>a.intensity))];
          const stance=Math.abs(aiSlot.billStance[selectCandidate.category]||0);
          const pick=options.reduce((best,i)=>{
            const rank={mild:1,moderate:2,aggressive:3};
            return Math.abs(rank[i]-Math.round(stance/34)) < Math.abs(rank[best]-Math.round(stance/34)) ? i : best;
          },options[0]);
          castSelectVote(selectCandidate.id,pick);
        } catch(e){ console.error('AI bill select-vote failed:',e); }
        finally { releaseAutomation(); }
      },700);
      return;
    }
    const passCandidate=pending.find(b=>b.stage==='pass' && r.slots.filter(s=>s.enabled!==false && !s.claimedBy).find(s=>b.passVotes[s.id]===undefined));
    if(passCandidate){
      const aiSlot=r.slots.filter(s=>s.enabled!==false && !s.claimedBy).find(s=>passCandidate.passVotes[s.id]===undefined);
      automationBusy=true; armAutomationWatchdog();
      setTimeout(()=>{
        try{
          const stance=aiSlot.billStance[passCandidate.category]||0;
          const agrees=(passCandidate.direction>0 && stance>=0)||(passCandidate.direction<0 && stance<=0);
          castPassVote(passCandidate.id, agrees?'yes':'no');
        } catch(e){ console.error('AI bill pass-vote failed:',e); }
        finally { releaseAutomation(); }
      },700);
      return;
    }
    const aiWithoutBill=r.slots.find(s=>s.enabled!==false && !s.claimedBy && !pending.some(b=>b.proposerId===s.id) && pending.length<6 && Math.random()<0.08);
    if(aiWithoutBill){
      automationBusy=true; armAutomationWatchdog();
      setTimeout(()=>{
        try{
          const cats=['fiscal','social','security'].filter(c=>!(r.activeLaws&&r.activeLaws[c]));
          if(cats.length){
            const cat=cats.reduce((best,c)=>Math.abs(aiWithoutBill.billStance[c])>Math.abs(aiWithoutBill.billStance[best])?c:best,cats[0]);
            proposeBill(cat, aiWithoutBill.billStance[cat]>=0?1:-1);
          }
        } catch(e){ console.error('AI bill proposal failed:',e); }
        finally { releaseAutomation(); }
      },700);
      return;
    }
```
Place this block immediately after `if(r.phase==='governing'){` and before the existing `if(r.currentEvent && r.currentEvent.id==='noconf' ...)` line, so bill-lifecycle steps take priority each tick but fall through to the existing event/no-confidence automation when no bill needs attention.
- [ ] **Step 2 (verify):** run the Task 1 Step 7 syntax check. Expected: `syntax OK`
- [ ] **Step 3: Commit**
```bash
git add "index (3).html"
git commit -m "Add AI automation for bill proposal, amendment, and voting"
```

---

### Task 7: Cross-task consistency pass (static, no live testing)

**Files:**
- Read-only review of `index (3).html`.

- [ ] **Step 1:** Re-read every site touched in Tasks 1–6 in one pass and confirm:
  - Every function name/signature used across tasks matches exactly (`computeBillMagnitude`, `tallySelectVote`, `tallyPassVote`, `proposeBill`, `submitAmendment`, `passOnAmendment`, `castSelectVote`, `castPassVote`, `applyActiveLawsAndDebt`, `regionLeaningFor`).
  - `pendingBills` is reset everywhere the Global Constraints section requires.
  - `activeLaws`/`publicDebt` are never touched by `startNewTerm()` or the mid-term-reshuffle branches.
  - `computeElectionFromSlots`'s new third parameter is passed at both call sites (`maybeRunElection`'s transaction and any other caller found via `grep -n "computeElectionFromSlots("`).
  - No leftover placeholder code (the `update.slots=slots;` line from Task 3 Step 3 must not exist after Task 4).
- [ ] **Step 2 (verify):** final syntax check (Task 1 Step 7 command).
- [ ] **Step 3: Commit** (only if Step 1 found and fixed anything)
```bash
git add "index (3).html"
git commit -m "Fix cross-task consistency issues found in E1 review pass"
```

---

## Self-Review Notes

- **Spec coverage:** §1 data model → Task 1; §2 magnitude table → Task 2; §3 lifecycle → Tasks 3, 5, 6; §4 ongoing effects/drift/debt → Task 4; §5 UI → Task 5; §6 AI automation → Task 6; §7 reset matrix → Task 1 + Task 7 verification.
- **Type consistency:** bill object shape is defined once in Task 3 Step 1 and read identically in Tasks 3, 4, 5, 6 — verified field names (`stage`,`amendments`,`respondedToAmend`,`selectVotes`,`passVotes`,`selectedIntensity`) are consistent throughout.
- **Known follow-up risk flagged for Task 7:** `REGIONS.leaning` is a shared module-level constant across all rooms in one browser tab — Task 4 deliberately does NOT mutate it, introducing `r.regionLeaning` instead and threading a fallback through every existing read site. This is the highest-risk task in the plan and is exactly the kind of cross-cutting change that produced state-leak bugs in prior Groups — Task 7's consistency pass exists specifically to catch any missed `reg.leaning` read site.
