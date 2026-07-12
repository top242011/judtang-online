# Group D Constitutional Court Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let the opposition petition a constitutional court against a minister or the PM over corruption they covered up, with a probability-weighted verdict that either vacates a ministry or triggers a mid-term PM reshuffle (reusing the existing coalition/cabinet formation flow without resetting the term clock) — plus an unconditional PM power to dissolve parliament at any time — per [docs/superpowers/specs/2026-07-12-group-d-constitutional-court-design.md](../specs/2026-07-12-group-d-constitutional-court-design.md).

**Architecture:** Same single-file `index (3).html`, no build step, Firestore as the only backend. This plan builds on Group A/B/C1/C2 (merged into `main` at commit `9680fc6`).

**Tech Stack:** Vanilla JS, Firebase JS SDK 10.7.1 (compat), Firestore. No automated test framework exists in this project. Pure functions (no DOM/Firestore dependency, given a plain data object) are verified with standalone `node -e` snippets before being wired in; everything touching Firestore/DOM is verified by hand in a browser against the real Firebase project.

## Global Constraints

- Do not introduce a build step, bundler, package.json, or any new dependency.
- All new user-facing strings must be in Thai, matching the existing tone.
- New room-state fields: `corruptionIncidents` (array, default `[]`, never reset by `startNewTerm()`/`confirmCabinet()` — persists across governments), `pendingPmDisqualification` (object or `null`, default `null`), `midTermReshuffle` (boolean, default `false`), `dissolvedByPm` (boolean, default `false`).
- Function names introduced across tasks must match exactly: `countGuiltyIncidents`, `computeCourtVerdictProbability`, `fileCourtPetition`, `resolveCourtCase`, `dissolveParliament`.
- `countGuiltyIncidents(corruptionIncidents, targetId)` counts only entries where `choiceIndex===1` ("ปกป้อง ไม่ตั้งกรรมการ") — `choiceIndex===0` never counts as grounds.
- Exact verdict formula (do not change): `pGuilty = clamp(0.4 + 0.15*countGuiltyIncidents(...), 0, 0.95)`.
- A guilty minister loses their portfolio for the rest of the term (no mid-term reassignment UI). A guilty PM does NOT end the term — it triggers a same-term PM reshuffle that preserves `turn`/`turnsPerTerm`/`usedEventIds` and only resets `decisionHistory` (new government, fresh track record).
- `dissolveParliament()` has no cost or condition — any PM can call it at any time during `governing` phase.
- `oppositionActions[partyId]` must always be updated via a merge (`{...(r.oppositionActions?.[partyId]||{}), newFlag:true}`), never a flat overwrite — this project already has a real bug here (`fileNoConfidence()` overwrites the whole per-party object) that this plan's Task 3 must fix as part of adding the second flag.

---

### Task 1: Room-state field defaults for the four new fields

**Files:**
- Modify: `index (3).html` — `handleCreateRoom()`, `resetRoomToLobby()`, `startNewTerm()`

**Interfaces:**
- Produces: every room document has `corruptionIncidents:[]`, `pendingPmDisqualification:null`, `midTermReshuffle:false`, `dissolvedByPm:false` from creation onward. Tasks 2-7 assume these fields exist on every room doc they read.

- [ ] **Step 1: `handleCreateRoom()` — add all four fields**

Find this exact block:

```javascript
    turn:1, turnsPerTerm:16, usedEventIds:[], currentEvent:null, currentEventOwnerSlot:null, currentEffectResult:null,
    collapsed:false, electionLock:false, pendingCollapse:false, deficitStreak:0, concededPm:[],
    decisionHistory:[], noConfidenceVotes:{},
  };
```

Replace with:

```javascript
    turn:1, turnsPerTerm:16, usedEventIds:[], currentEvent:null, currentEventOwnerSlot:null, currentEffectResult:null,
    collapsed:false, electionLock:false, pendingCollapse:false, deficitStreak:0, concededPm:[],
    decisionHistory:[], noConfidenceVotes:{},
    corruptionIncidents:[], pendingPmDisqualification:null, midTermReshuffle:false, dissolvedByPm:false,
  };
```

- [ ] **Step 2: `resetRoomToLobby()` — add all four fields**

Find this exact block:

```javascript
    turn:1, turnsPerTerm:16, usedEventIds:[], currentEvent:null, currentEventOwnerSlot:null, currentEffectResult:null,
    collapsed:false, electionLock:false, pendingCollapse:false, deficitStreak:0, concededPm:[],
    decisionHistory:[], noConfidenceVotes:{},
  });
}
```

Replace with:

```javascript
    turn:1, turnsPerTerm:16, usedEventIds:[], currentEvent:null, currentEventOwnerSlot:null, currentEffectResult:null,
    collapsed:false, electionLock:false, pendingCollapse:false, deficitStreak:0, concededPm:[],
    decisionHistory:[], noConfidenceVotes:{},
    corruptionIncidents:[], pendingPmDisqualification:null, midTermReshuffle:false, dissolvedByPm:false,
  });
}
```

- [ ] **Step 3: `startNewTerm()` — reset only `dissolvedByPm` (the other three fields must NOT be touched here per the spec's reset table)**

Find this exact block:

```javascript
  roomRef().update({phase:'campaign',term:r.term+1,slots,electionLock:false,invitations:{},pendingCollapse:false,oppositionActions:{},queuedEvent:null,deficitStreak:0});
}
```

Replace with:

```javascript
  roomRef().update({phase:'campaign',term:r.term+1,slots,electionLock:false,invitations:{},pendingCollapse:false,oppositionActions:{},queuedEvent:null,deficitStreak:0,dissolvedByPm:false});
}
```

- [ ] **Step 4: Manually verify**

```bash
cd "/Users/panupanpitak/Projects/judtang-online" && python3 -m http.server 8810
```

Open `http://localhost:8810/index%20(3).html` via the Claude_Browser MCP tools, create a room, and confirm via console:

```javascript
db.collection('jadtang_rooms').doc(ui.roomCode).get().then(s=>console.log(s.data().corruptionIncidents, s.data().pendingPmDisqualification, s.data().midTermReshuffle, s.data().dissolvedByPm))
```

Expected: `[] null false false`. Then click "รีเซ็ตเกม" (from `topbar()`, available once past lobby — seed `phase:'campaign'` via console first if needed) and confirm the same four values after reset. Then seed a non-empty `corruptionIncidents` array and a `midTermReshuffle:true` via console, call `startNewTerm()` directly, and confirm `dissolvedByPm` becomes `false` while `corruptionIncidents` and `midTermReshuffle` are UNCHANGED (still whatever you seeded) — this proves Step 3's narrower reset is correct.

- [ ] **Step 5: Commit**

```bash
git add "index (3).html"
git commit -m "Add corruptionIncidents/pendingPmDisqualification/midTermReshuffle/dissolvedByPm fields"
```

---

### Task 2: Widen the corruption event to implicate the PM; record `corruptionIncidents`

**Files:**
- Modify: `index (3).html` — `drawEventForRoom()`, `chooseEvent()`; add `countGuiltyIncidents()`

**Interfaces:**
- Consumes: `r.corruptionIncidents` (Task 1)
- Produces: `countGuiltyIncidents(corruptionIncidents, targetId) -> number`. Task 3 (eligibility check), Task 4 (`computeCourtVerdictProbability`), and Task 6 (UI) all call this.

- [ ] **Step 1: Validate the logic in isolation with Node**

```bash
node -e "
function countGuiltyIncidents(corruptionIncidents,targetId){
  return (corruptionIncidents||[]).filter(d=>d.targetId===targetId && d.choiceIndex===1).length;
}

const history=[
  {targetId:'s1',choiceIndex:1},
  {targetId:'s1',choiceIndex:0},
  {targetId:'s2',choiceIndex:1},
  {targetId:'s1',choiceIndex:1},
];

console.assert(countGuiltyIncidents(history,'s1')===2,'s1: expected 2 got '+countGuiltyIncidents(history,'s1'));
console.assert(countGuiltyIncidents(history,'s2')===1,'s2: expected 1 got '+countGuiltyIncidents(history,'s2'));
console.assert(countGuiltyIncidents(history,'s3')===0,'s3: expected 0 got '+countGuiltyIncidents(history,'s3'));
console.assert(countGuiltyIncidents(undefined,'s1')===0,'undefined history: expected 0');
console.assert(countGuiltyIncidents([],'s1')===0,'empty history: expected 0');

console.log('ALL PASS');
"
```

Expected output: `ALL PASS`.

- [ ] **Step 2: Add `countGuiltyIncidents()` near the other pure helpers**

Find this exact block:

```javascript
function countBadDecisions(decisionHistory){
  return (decisionHistory||[]).filter(d=>d.trustDelta<0).length;
}
```

Replace with:

```javascript
function countBadDecisions(decisionHistory){
  return (decisionHistory||[]).filter(d=>d.trustDelta<0).length;
}
function countGuiltyIncidents(corruptionIncidents,targetId){
  return (corruptionIncidents||[]).filter(d=>d.targetId===targetId && d.choiceIndex===1).length;
}
```

- [ ] **Step 3: Widen the corruption event's candidate ministry pool to include the PM's own ministries, and tag the event with `_corruptionTarget`**

Find this exact block:

```javascript
  let pool=EVENTS.filter(e=>!r.usedEventIds.includes(e.id));
  const partnerMinistries=MINISTRIES.filter(m=>r.cabinet[m.id] && r.cabinet[m.id]!==r.pm);
  if(partnerMinistries.length && !r.usedEventIds.includes('corruption')){
    const m=partnerMinistries[Math.floor(Math.random()*partnerMinistries.length)];
    const ownerId=r.cabinet[m.id];
    const owner=r.slots.find(s=>s.id===ownerId);
    pool=pool.concat([{
      id:'corruption', title:'เรื่องอื้อฉาวทุจริตใน'+m.name,
      desc:`มีรายงานข่าวการทุจริตจัดซื้อจัดจ้างใน${m.name} ซึ่งอยู่ภายใต้การกำกับดูแลของ${owner?owner.name:'พรรคร่วมรัฐบาล'} สื่อมวลชนจับตาอย่างใกล้ชิด`,
      choices:[
        {label:'ตั้งกรรมการสอบสวนอิสระ',desc:'สร้างความโปร่งใส แต่กระทบความสัมพันธ์กับพรรคร่วม',effect:{trust:8,treasury:-5,satisfactionParty:{id:ownerId,val:-15}}},
        {label:'ปกป้องพรรคร่วมรัฐบาล ไม่ตั้งกรรมการ',desc:'รักษาความสัมพันธ์ในรัฐบาล แต่เสี่ยงเสียความเชื่อมั่นสาธารณะ',effect:{trust:-15,unrest:8,satisfactionParty:{id:ownerId,val:10}}},
      ]
    }]);
  }
```

Replace with:

```javascript
  let pool=EVENTS.filter(e=>!r.usedEventIds.includes(e.id));
  const assignedMinistries=MINISTRIES.filter(m=>r.cabinet[m.id]);
  if(assignedMinistries.length && !r.usedEventIds.includes('corruption')){
    const m=assignedMinistries[Math.floor(Math.random()*assignedMinistries.length)];
    const ownerId=r.cabinet[m.id];
    const owner=r.slots.find(s=>s.id===ownerId);
    pool=pool.concat([{
      id:'corruption', title:'เรื่องอื้อฉาวทุจริตใน'+m.name,
      desc:`มีรายงานข่าวการทุจริตจัดซื้อจัดจ้างใน${m.name} ซึ่งอยู่ภายใต้การกำกับดูแลของ${owner?owner.name:'พรรคร่วมรัฐบาล'} สื่อมวลชนจับตาอย่างใกล้ชิด`,
      _corruptionTarget: ownerId,
      choices:[
        {label:'ตั้งกรรมการสอบสวนอิสระ',desc:'สร้างความโปร่งใส แต่กระทบความสัมพันธ์กับพรรคร่วม',effect:{trust:8,treasury:-5,satisfactionParty:{id:ownerId,val:-15}}},
        {label:'ปกป้องพรรคร่วมรัฐบาล ไม่ตั้งกรรมการ',desc:'รักษาความสัมพันธ์ในรัฐบาล แต่เสี่ยงเสียความเชื่อมั่นสาธารณะ',effect:{trust:-15,unrest:8,satisfactionParty:{id:ownerId,val:10}}},
      ]
    }]);
  }
```

Note: `ownerId` can now equal `r.pm` — the decision-maker (`ownerSlot`, set further down in this same function, unchanged) still always defaults to `r.pm` for corruption events, so this doesn't change who picks the response, only who can be implicated.

- [ ] **Step 4: Record a `corruptionIncidents` entry whenever a `corruption` event resolves**

Find this exact block:

```javascript
  update.decisionHistory=[...(r.decisionHistory||[]),{eventId:ev.id,choiceIndex:choiceIdx,trustDelta:eff.trust||0,turn:r.turn,term:r.term}];
  update.noConfidenceVotes={};
```

Replace with:

```javascript
  update.decisionHistory=[...(r.decisionHistory||[]),{eventId:ev.id,choiceIndex:choiceIdx,trustDelta:eff.trust||0,turn:r.turn,term:r.term}];
  update.noConfidenceVotes={};
  if(ev.id==='corruption'){
    update.corruptionIncidents=[...(r.corruptionIncidents||[]),{targetId:ev._corruptionTarget,term:r.term,turn:r.turn,choiceIndex:choiceIdx}];
  }
```

- [ ] **Step 5: Manually verify in browser**

Using the local server from Task 1, seed a room in `governing` phase where the PM's own ministry (not just a partner's) is the one selected for a corruption event — the easiest way is to seed `r.cabinet` so `r.pm` holds at least one ministry, then force a corruption event: `roomRef().update({currentEvent:{id:'corruption',title:'...', desc:'...', _corruptionTarget:ui.room.pm, choices:[{label:'ตั้งกรรมการสอบสวนอิสระ',desc:'...',effect:{trust:8,treasury:-5,satisfactionParty:{id:ui.room.pm,val:-15}}},{label:'ปกป้องพรรคร่วมรัฐบาล ไม่ตั้งกรรมการ',desc:'...',effect:{trust:-15,unrest:8,satisfactionParty:{id:ui.room.pm,val:10}}}]}, currentEffectResult:null})`. Pick the second choice (index 1) and confirm via console that `corruptionIncidents` gained an entry with `targetId` equal to the PM's own slot id and `choiceIndex:1`. Also confirm picking choice 0 records `choiceIndex:0`.

- [ ] **Step 6: Commit**

```bash
git add "index (3).html"
git commit -m "Let corruption events implicate the PM's own ministries and record corruptionIncidents"
```

---

### Task 3: Fix the `oppositionActions` merge bug; add `fileCourtPetition()`

**Files:**
- Modify: `index (3).html` — `fileNoConfidence()`; add `fileCourtPetition()`

**Interfaces:**
- Consumes: `countGuiltyIncidents()` (Task 2)
- Produces: `fileCourtPetition(targetId)` — Task 6 (UI) calls this.

- [ ] **Step 1: Fix `fileNoConfidence()`'s overwrite bug and add `fileCourtPetition()` right after it**

Find this exact block:

```javascript
function fileNoConfidence(){
  const r=ui.room; const mine=mySlot();
  if(!mine || r.coalitionSlots.includes(mine.id)) return;
  if(r.oppositionActions?.[mine.id]?.noConfidenceUsedThisTerm) return;
  if(r.queuedEvent || (r.currentEvent && r.currentEvent._filedBy)) return;
  const oppositionActions={...(r.oppositionActions||{}),[mine.id]:{noConfidenceUsedThisTerm:true}};
  stampAndThen('ยื่นญัตติแล้ว',()=>{
    roomRef().update({oppositionActions,queuedEvent:{filedBy:mine.id}});
  });
}
```

Replace with:

```javascript
function fileNoConfidence(){
  const r=ui.room; const mine=mySlot();
  if(!mine || r.coalitionSlots.includes(mine.id)) return;
  if(r.oppositionActions?.[mine.id]?.noConfidenceUsedThisTerm) return;
  if(r.queuedEvent || (r.currentEvent && r.currentEvent._filedBy)) return;
  const oppositionActions={...(r.oppositionActions||{}),[mine.id]:{...(r.oppositionActions?.[mine.id]||{}),noConfidenceUsedThisTerm:true}};
  stampAndThen('ยื่นญัตติแล้ว',()=>{
    roomRef().update({oppositionActions,queuedEvent:{filedBy:mine.id,type:'noconf'}});
  });
}
function fileCourtPetition(targetId){
  const r=ui.room; const mine=mySlot();
  if(!mine || r.coalitionSlots.includes(mine.id)) return;
  if(r.oppositionActions?.[mine.id]?.courtPetitionUsedThisTerm) return;
  if(r.queuedEvent || (r.currentEvent && r.currentEvent._filedBy)) return;
  if(countGuiltyIncidents(r.corruptionIncidents,targetId)<1) return;
  const oppositionActions={...(r.oppositionActions||{}),[mine.id]:{...(r.oppositionActions?.[mine.id]||{}),courtPetitionUsedThisTerm:true}};
  stampAndThen('ยื่นคำร้องแล้ว',()=>{
    roomRef().update({oppositionActions,queuedEvent:{filedBy:mine.id,targetId,type:'courtcase'}});
  });
}
```

(Note: `fileNoConfidence()` now explicitly sets `type:'noconf'` on the queued event — this is a harmless clarifying addition; `drawEventForRoom()` in Task 4 will branch on this field, defaulting to the no-confidence path when `type` isn't `'courtcase'`.)

- [ ] **Step 2: Manually verify in browser**

Using the local server, seed a room in `governing` phase with a claimed opposition party and a `corruptionIncidents` entry making that party eligible (`targetId` = some slot, `choiceIndex:1`). As that opposition player:

1. Call `fileNoConfidence()` — confirm `oppositionActions[mine.id]` now has `noConfidenceUsedThisTerm:true`.
2. Reset `queuedEvent` to `null` via console (simulating the motion having already resolved) so you can test the second action independently, then call `fileCourtPetition(targetId)` with the eligible target's id — confirm `oppositionActions[mine.id]` now has BOTH `noConfidenceUsedThisTerm:true` AND `courtPetitionUsedThisTerm:true` (proving the merge fix works — before the fix, this second call would have wiped out the first flag).
3. Call `fileCourtPetition(someOtherPartyIdWithNoIncidents)` (a target with zero qualifying incidents) — confirm it's a no-op (guard blocks it, no Firestore write).

- [ ] **Step 3: Commit**

```bash
git add "index (3).html"
git commit -m "Fix oppositionActions overwrite bug and add fileCourtPetition()"
```

---

### Task 4: The `courtcase` event — construction, resolution, and verdict probability

**Files:**
- Modify: `index (3).html` — `drawEventForRoom()`, `chooseEvent()`; add `computeCourtVerdictProbability()`, `resolveCourtCase()`

**Interfaces:**
- Consumes: `countGuiltyIncidents()` (Task 2), `computeCoalitionSatisfaction()` (pre-existing)
- Produces: `computeCourtVerdictProbability(corruptionIncidents, targetId) -> number`, `resolveCourtCase()`. Task 5 doesn't call these directly but depends on `resolveCourtCase()` setting `pendingPmDisqualification` correctly.

- [ ] **Step 1: Validate `computeCourtVerdictProbability` in isolation with Node**

```bash
node -e "
function clamp(v,min=0,max=100){return Math.max(min,Math.min(max,v));}
function countGuiltyIncidents(corruptionIncidents,targetId){
  return (corruptionIncidents||[]).filter(d=>d.targetId===targetId && d.choiceIndex===1).length;
}
function computeCourtVerdictProbability(corruptionIncidents,targetId){
  return clamp(0.4+0.15*countGuiltyIncidents(corruptionIncidents,targetId),0,0.95);
}

console.assert(computeCourtVerdictProbability([],'s1')===0.4,'0 incidents: expected 0.4');
console.assert(computeCourtVerdictProbability([{targetId:'s1',choiceIndex:1}],'s1')===0.55,'1 incident: expected 0.55');
console.assert(computeCourtVerdictProbability([{targetId:'s1',choiceIndex:1},{targetId:'s1',choiceIndex:1}],'s1')===0.7,'2 incidents: expected 0.7');
// clamp ceiling: 0.4 + 0.15*10 = 1.9, must clamp to 0.95
const many=Array.from({length:10},()=>({targetId:'s1',choiceIndex:1}));
console.assert(computeCourtVerdictProbability(many,'s1')===0.95,'ceiling: expected 0.95 got '+computeCourtVerdictProbability(many,'s1'));

console.log('ALL PASS');
"
```

Expected output: `ALL PASS`.

- [ ] **Step 2: Add `computeCourtVerdictProbability()` near `countGuiltyIncidents()`**

Find this exact block:

```javascript
function countGuiltyIncidents(corruptionIncidents,targetId){
  return (corruptionIncidents||[]).filter(d=>d.targetId===targetId && d.choiceIndex===1).length;
}
```

Replace with:

```javascript
function countGuiltyIncidents(corruptionIncidents,targetId){
  return (corruptionIncidents||[]).filter(d=>d.targetId===targetId && d.choiceIndex===1).length;
}
function computeCourtVerdictProbability(corruptionIncidents,targetId){
  return clamp(0.4+0.15*countGuiltyIncidents(corruptionIncidents,targetId),0,0.95);
}
```

- [ ] **Step 3: Add the `courtcase` branch to `drawEventForRoom()`**

Find this exact block:

```javascript
function drawEventForRoom(r){
  if(r.queuedEvent){
    const filerSlot=r.slots.find(s=>s.id===r.queuedEvent.filedBy);
    const base=EVENTS.find(e=>e.id==='noconf');
    const event={
      ...base,
      title:'ญัตติอภิปรายไม่ไว้วางใจ (ยื่นโดย'+(filerSlot?filerSlot.name:'ฝ่ายค้าน')+')',
      _filedBy:r.queuedEvent.filedBy,
    };
    return {event,ownerSlot:r.pm};
  }
```

Replace with:

```javascript
function drawEventForRoom(r){
  if(r.queuedEvent){
    const filerSlot=r.slots.find(s=>s.id===r.queuedEvent.filedBy);
    if(r.queuedEvent.type==='courtcase'){
      const targetSlot=r.slots.find(s=>s.id===r.queuedEvent.targetId);
      const event={
        id:'courtcase',
        title:'ศาลรัฐธรรมนูญวินิจฉัยคดี'+(targetSlot?targetSlot.name:''),
        desc:`${filerSlot?filerSlot.name:'ฝ่ายค้าน'}ยื่นคำร้องต่อศาลรัฐธรรมนูญให้วินิจฉัยกรณีทุจริตของ${targetSlot?targetSlot.name:'พรรคที่ถูกกล่าวหา'} ศาลรับคำร้องไว้พิจารณาแล้ว`,
        choices:[{label:'รับทราบคำวินิจฉัยของศาล',desc:'ศาลได้พิจารณาและมีคำวินิจฉัยแล้ว',effect:{}}],
        _filedBy:r.queuedEvent.filedBy,
        _courtPetition:{targetId:r.queuedEvent.targetId},
      };
      return {event,ownerSlot:r.pm};
    }
    const base=EVENTS.find(e=>e.id==='noconf');
    const event={
      ...base,
      title:'ญัตติอภิปรายไม่ไว้วางใจ (ยื่นโดย'+(filerSlot?filerSlot.name:'ฝ่ายค้าน')+')',
      _filedBy:r.queuedEvent.filedBy,
    };
    return {event,ownerSlot:r.pm};
  }
```

- [ ] **Step 4: Delegate `chooseEvent()` to `resolveCourtCase()` for `courtcase` events, and add `resolveCourtCase()`**

Find this exact block:

```javascript
function chooseEvent(choiceIdx){
  const r=ui.room;
  const ev=r.currentEvent;
  const choice=ev.choices[choiceIdx];
  let eff=choice.effect;
```

Replace with:

```javascript
function chooseEvent(choiceIdx){
  const r=ui.room;
  const ev=r.currentEvent;
  if(ev.id==='courtcase'){ resolveCourtCase(); return; }
  const choice=ev.choices[choiceIdx];
  let eff=choice.effect;
```

Find this exact block (the end of `resolveNoConfidenceVote()`):

```javascript
function resolveNoConfidenceVote(){
  const r=ui.room;
  chooseEvent(tallyNoConfidenceWinner(r));
}
```

Replace with (unchanged `resolveNoConfidenceVote`, adds `resolveCourtCase` right after):

```javascript
function resolveNoConfidenceVote(){
  const r=ui.room;
  chooseEvent(tallyNoConfidenceWinner(r));
}
function resolveCourtCase(){
  const r=ui.room;
  const ev=r.currentEvent;
  const targetId=ev._courtPetition.targetId;
  const filedBy=ev._filedBy;
  const targetSlot=r.slots.find(s=>s.id===targetId);
  const guilty=Math.random()<computeCourtVerdictProbability(r.corruptionIncidents,targetId);
  const update={usedEventIds:r.usedEventIds,queuedEvent:null};
  if(guilty){
    if(targetId===r.pm){
      update.pendingPmDisqualification={targetId};
      update.currentEffectResult={text:`ศาลรัฐธรรมนูญวินิจฉัยว่า ${targetSlot?targetSlot.name:''} มีความผิดจริง ต้องพ้นจากตำแหน่งนายกรัฐมนตรีทันที`,tags:[{sign:'neg',text:'นายกรัฐมนตรีพ้นจากตำแหน่ง'}]};
    } else {
      const cabinet={...r.cabinet};
      Object.keys(cabinet).forEach(mid=>{ if(cabinet[mid]===targetId) delete cabinet[mid]; });
      update.cabinet=cabinet;
      update.coalitionParties=computeCoalitionSatisfaction({...r,cabinet},cabinet);
      update.currentEffectResult={text:`ศาลรัฐธรรมนูญวินิจฉัยว่า ${targetSlot?targetSlot.name:''} มีความผิดจริง เสียตำแหน่งรัฐมนตรีทันที`,tags:[{sign:'neg',text:`${targetSlot?targetSlot.name:''} เสียตำแหน่งรัฐมนตรี`}]};
    }
  } else {
    const filerSlot=r.slots.find(s=>s.id===filedBy);
    const popularity={}; Object.keys(r.popularity||{}).forEach(k=>{ popularity[k]={...(r.popularity[k]||{})}; });
    popularity[filedBy]=popularity[filedBy]||{north:0,isaan:0,central:0,bangkok:0,south:0};
    REGIONS.forEach(reg=>{ popularity[filedBy][reg.id]=clamp((popularity[filedBy][reg.id]||0)-8,-60,60); });
    update.popularity=popularity;
    update.currentEffectResult={text:`ศาลรัฐธรรมนูญยกคำร้อง — ${targetSlot?targetSlot.name:''} ไม่มีความผิด`,tags:[{sign:'neg',text:`ความนิยม ${filerSlot?filerSlot.name:'ผู้ยื่น'} -8`}]};
  }
  roomRef().update(update);
}
```

- [ ] **Step 5: Manually verify in browser**

Using the local server, seed a room in `governing` phase and a `corruptionIncidents` entry for a target minister (2+ guilty incidents, to push `pGuilty` well above 0.5 so a guilty verdict is likely though not certain — you may need to retry a few times given the randomness). Force `queuedEvent:{filedBy:'<opposition slot id>', targetId:'<minister slot id>', type:'courtcase'}` then call `drawEventForRoom(ui.room)` from the console to inspect the constructed event shape (don't apply it yet — just call it standalone to check `event.title`/`event.desc`/`event._courtPetition` look right), then actually seed `currentEvent` to that shape and click the single "รับทราบคำวินิจฉัยของศาล" choice. Confirm:
- **Minister target, guilty:** `cabinet` no longer has that minister assigned to the target; `coalitionParties` recomputed.
- **Minister target, not guilty** (retry with a target that has 0 incidents seeded temporarily, forcing pGuilty=0.4, and re-roll until you observe a non-guilty outcome, or just inspect the code path by reading — both paths should be exercised across your testing): filer's popularity drops by 8 in every region.
- **PM target, guilty:** `pendingPmDisqualification` is set to `{targetId: <pm id>}`; `currentEffectResult` text mentions the PM losing office.
- In every case, `queuedEvent` is `null` afterward.

- [ ] **Step 6: Commit**

```bash
git add "index (3).html"
git commit -m "Add courtcase event construction, verdict probability, and resolution"
```

---

### Task 5: Mid-term PM reshuffle after a guilty verdict

**Files:**
- Modify: `index (3).html` — `nextTurn()`, `confirmCabinet()`, the AI-cabinet-automation branch inside `runHostAutomation()`

**Interfaces:**
- Consumes: `r.pendingPmDisqualification` (Task 4 sets it), `nextPmCandidate()` (pre-existing, from Group A)

- [ ] **Step 1: `nextTurn()` — handle `pendingPmDisqualification` before the existing `pendingCollapse`/turn-limit checks**

Find this exact block:

```javascript
function nextTurn(){
  const r=ui.room;
  if(r.pendingCollapse){
```

Replace with:

```javascript
function nextTurn(){
  const r=ui.room;
  if(r.pendingPmDisqualification){
    const disqualifiedId=r.pendingPmDisqualification.targetId;
    const concededPm=[disqualifiedId];
    const next=nextPmCandidate({...r,concededPm});
    if(!next){
      roomRef().update({collapsed:true,phase:'term_summary',pendingPmDisqualification:null});
      return;
    }
    if(next.seats>=MAJORITY){
      const coalitionParties=[{id:next.id,satisfaction:100}];
      roomRef().update({phase:'cabinet',pm:next.id,coalitionSlots:[next.id],coalitionParties,cabinet:{},invitations:{},concededPm,pendingPmDisqualification:null,midTermReshuffle:true});
    } else {
      roomRef().update({phase:'coalition',pm:next.id,coalitionSlots:[],coalitionParties:[],invitations:{},cabinet:{},concededPm,pendingPmDisqualification:null,midTermReshuffle:true});
    }
    return;
  }
  if(r.pendingCollapse){
```

- [ ] **Step 2: `confirmCabinet()` — preserve `turn`/`usedEventIds` when `midTermReshuffle` is set**

Find this exact block:

```javascript
function confirmCabinet(){
  const r=ui.room;
  const coalitionParties=computeCoalitionSatisfaction(r,r.cabinet);
  const nextR={...r,coalitionParties,usedEventIds:[]};
  const {event,ownerSlot}=drawEventForRoom(nextR);
  stampAndThen('แต่งตั้งแล้ว',()=>{
    roomRef().update({phase:'governing',coalitionParties,turn:1,usedEventIds:[],currentEffectResult:null,currentEvent:event,currentEventOwnerSlot:ownerSlot,decisionHistory:[],noConfidenceVotes:{}});
  });
}
```

Replace with:

```javascript
function confirmCabinet(){
  const r=ui.room;
  const isReshuffle=!!r.midTermReshuffle;
  const coalitionParties=computeCoalitionSatisfaction(r,r.cabinet);
  const nextR={...r,coalitionParties,usedEventIds:isReshuffle?r.usedEventIds:[]};
  const {event,ownerSlot}=drawEventForRoom(nextR);
  stampAndThen('แต่งตั้งแล้ว',()=>{
    roomRef().update({
      phase:'governing', coalitionParties,
      turn: isReshuffle ? r.turn : 1,
      usedEventIds: isReshuffle ? r.usedEventIds : [],
      currentEffectResult:null, currentEvent:event, currentEventOwnerSlot:ownerSlot,
      decisionHistory:[], noConfidenceVotes:{}, midTermReshuffle:false,
    });
  });
}
```

- [ ] **Step 3: AI-cabinet-automation branch — same `isReshuffle` handling for when the reshuffled-in PM is AI-controlled**

Find this exact block:

```javascript
          const coalitionParties=computeCoalitionSatisfaction(r,cabinet);
          const nextR={...r,cabinet,coalitionParties,usedEventIds:[]};
          const {event,ownerSlot}=drawEventForRoom(nextR);
          roomRef().update({cabinet,coalitionParties,phase:'governing',turn:1,usedEventIds:[],currentEffectResult:null,currentEvent:event,currentEventOwnerSlot:ownerSlot,decisionHistory:[],noConfidenceVotes:{}}).catch(()=>{}).finally(releaseAutomation);
```

Replace with:

```javascript
          const isReshuffle=!!r.midTermReshuffle;
          const coalitionParties=computeCoalitionSatisfaction(r,cabinet);
          const nextR={...r,cabinet,coalitionParties,usedEventIds:isReshuffle?r.usedEventIds:[]};
          const {event,ownerSlot}=drawEventForRoom(nextR);
          roomRef().update({
            cabinet,coalitionParties,phase:'governing',
            turn: isReshuffle ? r.turn : 1,
            usedEventIds: isReshuffle ? r.usedEventIds : [],
            currentEffectResult:null,currentEvent:event,currentEventOwnerSlot:ownerSlot,
            decisionHistory:[],noConfidenceVotes:{},midTermReshuffle:false,
          }).catch(()=>{}).finally(releaseAutomation);
```

- [ ] **Step 4: Manually verify in browser**

Using the local server, drive a room to `governing` phase mid-term (e.g. `turn:10` of `turnsPerTerm:16`), seed `pendingPmDisqualification:{targetId:'<current pm id>'}` and a `currentEffectResult` (so the "ดำเนินการต่อ" button shows), then click it (or call `nextTurn()` directly). Confirm:
- If the next candidate has an outright majority: room jumps straight to `phase:'cabinet'` with `midTermReshuffle:true`, `pm` set to the new candidate, `concededPm` containing the disqualified id.
- Otherwise: room goes to `phase:'coalition'` with the same fields set.
- Complete the coalition/cabinet flow (invite parties if needed, assign ministries, click "แต่งตั้งคณะรัฐมนตรี" or let AI automation handle it if the new PM is AI) and confirm `turn` is STILL `10` (not reset to `1`) and `usedEventIds` still contains whatever it had before the reshuffle, while `decisionHistory` is `[]` (freshly reset) and `midTermReshuffle` is back to `false`.
- Separately, seed a scenario where `nextPmCandidate` would return `null` (e.g. a synthetic `lastElection.results` where only the disqualified party has any seats at all) and confirm the fallback correctly sets `collapsed:true, phase:'term_summary'`.

- [ ] **Step 5: Commit**

```bash
git add "index (3).html"
git commit -m "Reuse coalition/cabinet formation for a mid-term PM reshuffle after disqualification"
```

---

### Task 6: UI — court petition section in the opposition panel

**Files:**
- Modify: `index (3).html` — `PHASE_TEMPLATES.governing`

**Interfaces:**
- Consumes: `countGuiltyIncidents()` (Task 2), `fileCourtPetition(targetId)` (Task 3)

- [ ] **Step 1: Add the eligibility/gating variables and the new UI section**

Find this exact block:

```javascript
  const usedMotion = mine && r.oppositionActions?.[mine.id]?.noConfidenceUsedThisTerm;
  const motionPending = !!r.queuedEvent || (ev && ev._filedBy);
  const canFileMotion = isOpposition && !usedMotion && !motionPending;
```

Replace with:

```javascript
  const usedMotion = mine && r.oppositionActions?.[mine.id]?.noConfidenceUsedThisTerm;
  const motionPending = !!r.queuedEvent || (ev && ev._filedBy);
  const canFileMotion = isOpposition && !usedMotion && !motionPending;
  const eligibleCourtTargets = r.slots.filter(s=>countGuiltyIncidents(r.corruptionIncidents,s.id)>=1);
  const usedCourtPetition = mine && r.oppositionActions?.[mine.id]?.courtPetitionUsedThisTerm;
  const canFileCourtPetition = isOpposition && !usedCourtPetition && !motionPending;
```

Find this exact block:

```javascript
      ${isOpposition?`
      <div class="panel-dark">
        <h3 style="margin-top:0;font-size:14px;">บทบาทฝ่ายค้าน</h3>
        <div style="font-size:12.5px;color:var(--muted-on-navy);margin-bottom:10px;">พรรคของคุณสะสมความนิยมเองจากผลงานรัฐบาล — ยิ่งรัฐบาลบริหารแย่ พรรคคุณยิ่งได้เปรียบตอนเลือกตั้งครั้งหน้า</div>
        <button class="btn btn-ghost" style="width:100%;" ${canFileMotion?'':'disabled'} onclick="fileNoConfidence()">ยื่นอภิปรายไม่ไว้วางใจ</button>
        <div style="font-size:11px;color:var(--muted-on-navy);margin-top:6px;">${usedMotion?'ใช้สิทธิ์ในสมัยนี้ไปแล้ว':(motionPending?'มีญัตติค้างอยู่ในคิว':'ใช้ได้ 1 ครั้งต่อสมัย — บีบให้รัฐบาลตอบคำถามในสภา')}</div>
      </div>`:''}
```

Replace with:

```javascript
      ${isOpposition?`
      <div class="panel-dark">
        <h3 style="margin-top:0;font-size:14px;">บทบาทฝ่ายค้าน</h3>
        <div style="font-size:12.5px;color:var(--muted-on-navy);margin-bottom:10px;">พรรคของคุณสะสมความนิยมเองจากผลงานรัฐบาล — ยิ่งรัฐบาลบริหารแย่ พรรคคุณยิ่งได้เปรียบตอนเลือกตั้งครั้งหน้า</div>
        <button class="btn btn-ghost" style="width:100%;" ${canFileMotion?'':'disabled'} onclick="fileNoConfidence()">ยื่นอภิปรายไม่ไว้วางใจ</button>
        <div style="font-size:11px;color:var(--muted-on-navy);margin-top:6px;">${usedMotion?'ใช้สิทธิ์ในสมัยนี้ไปแล้ว':(motionPending?'มีญัตติค้างอยู่ในคิว':'ใช้ได้ 1 ครั้งต่อสมัย — บีบให้รัฐบาลตอบคำถามในสภา')}</div>
        <div style="margin-top:14px;border-top:1px solid var(--navy-line);padding-top:12px;">
          <div style="font-size:12.5px;color:var(--muted-on-navy);margin-bottom:8px;">ยื่นคำร้องต่อศาลรัฐธรรมนูญวินิจฉัยกรณีทุจริตที่มีมูล</div>
          ${eligibleCourtTargets.length?eligibleCourtTargets.map(t=>`<button class="btn btn-ghost" style="width:100%;margin-bottom:6px;" ${canFileCourtPetition?'':'disabled'} onclick="fileCourtPetition('${t.id}')">ยื่นคำร้องต่อ ${t.name}</button>`).join(''):'<div style="font-size:11px;color:var(--muted-on-navy);">ยังไม่มีพรรคใดมีมูลทุจริตให้ยื่นคำร้องได้</div>'}
          ${eligibleCourtTargets.length?`<div style="font-size:11px;color:var(--muted-on-navy);margin-top:4px;">${usedCourtPetition?'ใช้สิทธิ์ในสมัยนี้ไปแล้ว':(motionPending?'มีคำร้อง/ญัตติค้างอยู่ในคิว':'ใช้ได้ 1 ครั้งต่อสมัย')}</div>`:''}
        </div>
      </div>`:''}
```

- [ ] **Step 2: Manually verify in browser**

Using two tabs (one opposition, one PM/government), on a room with no `corruptionIncidents` yet, confirm the opposition tab shows "ยังไม่มีพรรคใดมีมูลทุจริตให้ยื่นคำร้องได้" and no buttons. Then seed a `corruptionIncidents` entry making one party eligible (or resolve a real corruption event choosing "ปกป้อง ไม่ตั้งกรรมการ") and confirm a "ยื่นคำร้องต่อ [ชื่อพรรค]" button appears for the opposition player, clicking it queues the courtcase event (confirm via console: `r.queuedEvent.type==='courtcase'`), and the button becomes disabled/shows "ใช้สิทธิ์ในสมัยนี้ไปแล้ว" after use while the no-confidence button above remains independently usable (proving Task 3's merge fix works end-to-end through the UI, not just via direct function calls).

- [ ] **Step 3: Commit**

```bash
git add "index (3).html"
git commit -m "Add court-petition UI to the opposition panel"
```

---

### Task 7: Dissolve Parliament — unconditional PM power

**Files:**
- Modify: `index (3).html` — `PHASE_TEMPLATES.governing`, `PHASE_TEMPLATES.term_summary`; add `dissolveParliament()`

**Interfaces:**
- Produces: `dissolveParliament()` — called by the button this task adds.

- [ ] **Step 1: Add `dissolveParliament()`**

Find this exact block (the end of `resolveCourtCase()` from Task 4):

```javascript
  roomRef().update(update);
}

function fileNoConfidence(){
```

Replace with:

```javascript
  roomRef().update(update);
}

function dissolveParliament(){
  const r=ui.room; const mine=mySlot();
  if(!mine || mine.id!==r.pm) return;
  if(!confirm('ยืนยันยุบสภา? สมัยนี้จะสิ้นสุดทันทีและเข้าสู่การเลือกตั้งใหม่')) return;
  stampAndThen('ยุบสภาแล้ว',()=>{
    roomRef().update({phase:'term_summary',collapsed:false,dissolvedByPm:true});
  });
}

function fileNoConfidence(){
```

- [ ] **Step 2: Add the "ยุบสภา" button to the coalition-partners panel (PM only)**

Find this exact block:

```javascript
      <div class="panel-dark"><h3 style="margin-top:0;font-size:14px;">พรรคร่วมรัฐบาล</h3>
        ${r.coalitionParties.filter(p=>p.id!==r.pm).map(p=>{const s=r.slots.find(x=>x.id===p.id); return gauge(s.name,p.satisfaction);}).join('') || '<div style="font-size:12.5px;color:var(--muted-on-navy);">ปกครองเสียงข้างมากเดี่ยว</div>'}
      </div>
```

Replace with:

```javascript
      <div class="panel-dark"><h3 style="margin-top:0;font-size:14px;">พรรคร่วมรัฐบาล</h3>
        ${r.coalitionParties.filter(p=>p.id!==r.pm).map(p=>{const s=r.slots.find(x=>x.id===p.id); return gauge(s.name,p.satisfaction);}).join('') || '<div style="font-size:12.5px;color:var(--muted-on-navy);">ปกครองเสียงข้างมากเดี่ยว</div>'}
        ${mine && mine.id===r.pm?`<button class="btn btn-ghost" style="width:100%;margin-top:10px;" onclick="dissolveParliament()">ยุบสภา</button>`:''}
      </div>
```

- [ ] **Step 3: Give `term_summary` a third, distinct message for a PM-initiated dissolution**

Find this exact block:

```javascript
    <div class="term-badge">${collapsed?'รัฐบาลสิ้นสุดก่อนกำหนด':'ครบวาระการดำรงตำแหน่ง'}</div>
    <h1 class="serif" style="font-size:24px;margin:10px 0;">${collapsed?'รัฐบาลล่ม — พรรคร่วมถอนตัว':'สิ้นสุดสมัยที่ '+r.term}</h1>
  </div>
  <div class="doc">
    <h2>สรุปผลการบริหารประเทศ</h2>
    <div class="doc-grid">
      <div>
        <p><strong>เศรษฐกิจ:</strong> ${fmt(r.national.economy)}/100</p>
        <p><strong>ความเชื่อมั่นประชาชน:</strong> ${fmt(r.national.trust)}/100</p>
        <p><strong>ความไม่สงบ:</strong> ${fmt(r.national.unrest)}/100</p>
        <p><strong>งบประมาณแผ่นดินคงเหลือ:</strong> ฿${fmt(r.treasury)} ล้านบาท</p>
      </div>
      <div>
        <p style="font-size:13px;color:var(--ink-soft);">${collapsed?'ความไม่พอใจของพรรคร่วมรัฐบาลสะสมจนถอนตัว ทำให้เสียงในสภาไม่ถึงกึ่งหนึ่ง รัฐบาลจึงสิ้นสุดลงก่อนครบวาระ':'รัฐบาลบริหารประเทศครบวาระ มีการยุบสภาเพื่อจัดการเลือกตั้งใหม่'}</p>
      </div>
    </div>
```

Replace with:

```javascript
    <div class="term-badge">${r.dissolvedByPm?'PM ประกาศยุบสภา':(collapsed?'รัฐบาลสิ้นสุดก่อนกำหนด':'ครบวาระการดำรงตำแหน่ง')}</div>
    <h1 class="serif" style="font-size:24px;margin:10px 0;">${r.dissolvedByPm?'ยุบสภา — เข้าสู่การเลือกตั้งใหม่':(collapsed?'รัฐบาลล่ม — พรรคร่วมถอนตัว':'สิ้นสุดสมัยที่ '+r.term)}</h1>
  </div>
  <div class="doc">
    <h2>สรุปผลการบริหารประเทศ</h2>
    <div class="doc-grid">
      <div>
        <p><strong>เศรษฐกิจ:</strong> ${fmt(r.national.economy)}/100</p>
        <p><strong>ความเชื่อมั่นประชาชน:</strong> ${fmt(r.national.trust)}/100</p>
        <p><strong>ความไม่สงบ:</strong> ${fmt(r.national.unrest)}/100</p>
        <p><strong>งบประมาณแผ่นดินคงเหลือ:</strong> ฿${fmt(r.treasury)} ล้านบาท</p>
      </div>
      <div>
        <p style="font-size:13px;color:var(--ink-soft);">${r.dissolvedByPm?'นายกรัฐมนตรีใช้อำนาจยุบสภาก่อนครบวาระ เพื่อจัดการเลือกตั้งใหม่ตามที่เห็นสมควร':(collapsed?'ความไม่พอใจของพรรคร่วมรัฐบาลสะสมจนถอนตัว ทำให้เสียงในสภาไม่ถึงกึ่งหนึ่ง รัฐบาลจึงสิ้นสุดลงก่อนครบวาระ':'รัฐบาลบริหารประเทศครบวาระ มีการยุบสภาเพื่อจัดการเลือกตั้งใหม่')}</p>
      </div>
    </div>
```

- [ ] **Step 4: Manually verify in browser**

Using the local server, as PM during `governing` phase (any turn number), click "ยุบสภา", confirm the dialog, confirm. Confirm the room shows the term_summary screen with badge "PM ประกาศยุบสภา" and heading "ยุบสภา — เข้าสู่การเลือกตั้งใหม่" (not the "ครบวาระ"/"รัฐบาลล่ม" text). Click "จัดการเลือกตั้งใหม่" and confirm a normal new campaign starts, and that `dissolvedByPm` is now `false` (so a later NORMAL term-end or coalition-collapse in this new term shows its own correct message, not a stale "ยุบสภา" one).

- [ ] **Step 5: Commit**

```bash
git add "index (3).html"
git commit -m "Add unconditional PM power to dissolve parliament"
```

---

### Task 8: Full regression pass across Group D

**Files:**
- None (verification only)

- [ ] **Step 1: Play one continuous flow with two human tabs exercising every piece together**

Using two tabs (host/PM + second player as opposition or coalition partner):
1. Play several turns normally — confirm ordinary events, no-confidence motions (from Group C2), and everything else pre-existing still work unaffected.
2. Get a corruption event to occur against the PM's own ministry at least once (may need several turns/retries, or seed directly via console per earlier tasks' pattern) and choose "ปกป้อง ไม่ตั้งกรรมการ" — confirm the opposition tab's court-petition button appears for the PM.
3. File a court petition against the PM. Whichever verdict lands:
   - If not guilty: confirm the filer's popularity dropped in every region, and the game continues normally in `governing` phase.
   - If guilty: confirm the mid-term reshuffle correctly preserves `turn`/`usedEventIds`, resets `decisionHistory`, and a new PM/cabinet forms without a full new election.
4. Get (or seed) a second corruption event against a coalition-partner minister, choose "ปกป้อง ไม่ตั้งกรรมการ" again, file a court petition against that minister, and confirm a guilty verdict correctly vacates their ministry without touching PM status.
5. As PM, click "ยุบสภา" mid-term and confirm the term ends immediately with the correct distinct message, and a fresh campaign starts correctly afterward.
6. Confirm no console errors throughout (`mcp__Claude_Browser__read_console_messages`).

- [ ] **Step 2: Commit (only if a fixup was needed)**

If no fixups were needed, this task requires no commit. If a bug surfaced, fix it, re-run Step 1, then:

```bash
git add "index (3).html"
git commit -m "Fix regression found in Group D end-to-end verification"
```
