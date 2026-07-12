# Group C2 No-Confidence Depth Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the no-confidence-motion response into a seat-weighted coalition vote (PM breaks ties) instead of a PM-only decision, and make its numeric consequences scale with the current government's track record of trust-losing decisions — per [docs/superpowers/specs/2026-07-12-group-c2-no-confidence-depth-design.md](../specs/2026-07-12-group-c2-no-confidence-depth-design.md).

**Architecture:** Same single-file `index (3).html`, no build step, Firestore as the only backend. This plan builds on Group A/B/C1 (merged into `main` at commit `0ed5391`). Scoped strictly to the `noconf` event — every other governing event keeps its existing single-decider flow untouched.

**Tech Stack:** Vanilla JS, Firebase JS SDK 10.7.1 (compat), Firestore. No automated test framework exists in this project. Pure functions (no DOM/Firestore dependency, given a plain room-shaped object) are verified with standalone `node -e` snippets before being wired in; everything touching Firestore/DOM is verified by hand in a browser against the real Firebase project.

## Global Constraints

- Do not introduce a build step, bundler, package.json, or any new dependency.
- All new user-facing strings must be in Thai, matching the existing tone.
- New room-state fields: `decisionHistory` (array, default `[]`), `noConfidenceVotes` (object, default `{}`). `decisionHistory` persists across turns within a government's term and only resets when a new government is confirmed (`confirmCabinet()` and its AI-automation equivalent) or the room is created/reset. `noConfidenceVotes` resets every time a `noconf` event resolves (`chooseEvent()`) and every time a fresh event is drawn (`nextTurn()`).
- Function names introduced across tasks must match exactly: `countBadDecisions`, `computeNoConfidenceEffect`, `castNoConfidenceVote`, `tallyNoConfidenceWinner`, `resolveNoConfidenceVote`, `noConfidenceVotingPanel`.
- This feature must NOT change behavior for any event other than `id==='noconf'` — every new code path must be gated on that check.
- Exact formulas (from the approved spec, do not change): choice 0 effect `trust = Math.max(0, base - badCount)`; choice 1 effect `trust = base - badCount*2`, where `badCount = countBadDecisions(decisionHistory)`.
- Seat-weighted tally: sum each coalition party's seats (from `r.lastElection.results`) into whichever choice index it voted for; higher total wins; exact tie resolves to whatever `r.pm` itself voted.

---

### Task 1: Room-state field defaults for `decisionHistory`/`noConfidenceVotes`

**Files:**
- Modify: `index (3).html` — `handleCreateRoom()`, `resetRoomToLobby()`, `confirmCabinet()`, the AI-cabinet-automation branch inside `runHostAutomation()`, `nextTurn()`

**Interfaces:**
- Produces: every room document has `decisionHistory` and `noConfidenceVotes` fields from creation onward, reset at the points listed in the spec's table. Tasks 2-5 assume these fields exist on every room doc they read.

- [ ] **Step 1: `handleCreateRoom()` — add both fields with empty defaults**

Find this exact block:

```javascript
    turn:1, turnsPerTerm:16, usedEventIds:[], currentEvent:null, currentEventOwnerSlot:null, currentEffectResult:null,
    collapsed:false, electionLock:false, pendingCollapse:false, deficitStreak:0, concededPm:[],
  };
```

Replace with:

```javascript
    turn:1, turnsPerTerm:16, usedEventIds:[], currentEvent:null, currentEventOwnerSlot:null, currentEffectResult:null,
    collapsed:false, electionLock:false, pendingCollapse:false, deficitStreak:0, concededPm:[],
    decisionHistory:[], noConfidenceVotes:{},
  };
```

- [ ] **Step 2: `resetRoomToLobby()` — add both fields with empty defaults**

Find this exact block:

```javascript
    turn:1, turnsPerTerm:16, usedEventIds:[], currentEvent:null, currentEventOwnerSlot:null, currentEffectResult:null,
    collapsed:false, electionLock:false, pendingCollapse:false, deficitStreak:0, concededPm:[],
  });
}
```

Replace with:

```javascript
    turn:1, turnsPerTerm:16, usedEventIds:[], currentEvent:null, currentEventOwnerSlot:null, currentEffectResult:null,
    collapsed:false, electionLock:false, pendingCollapse:false, deficitStreak:0, concededPm:[],
    decisionHistory:[], noConfidenceVotes:{},
  });
}
```

- [ ] **Step 3: `confirmCabinet()` — reset both fields (new government, fresh track record)**

Find this exact block:

```javascript
function confirmCabinet(){
  const r=ui.room;
  const coalitionParties=computeCoalitionSatisfaction(r,r.cabinet);
  const nextR={...r,coalitionParties,usedEventIds:[]};
  const {event,ownerSlot}=drawEventForRoom(nextR);
  stampAndThen('แต่งตั้งแล้ว',()=>{
    roomRef().update({phase:'governing',coalitionParties,turn:1,usedEventIds:[],currentEffectResult:null,currentEvent:event,currentEventOwnerSlot:ownerSlot});
  });
}
```

Replace with:

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

- [ ] **Step 4: AI-cabinet-automation branch — reset both fields (mirrors `confirmCabinet()` for an AI-controlled PM)**

Find this exact block:

```javascript
          const coalitionParties=computeCoalitionSatisfaction(r,cabinet);
          const nextR={...r,cabinet,coalitionParties,usedEventIds:[]};
          const {event,ownerSlot}=drawEventForRoom(nextR);
          roomRef().update({cabinet,coalitionParties,phase:'governing',turn:1,usedEventIds:[],currentEffectResult:null,currentEvent:event,currentEventOwnerSlot:ownerSlot}).catch(()=>{}).finally(releaseAutomation);
```

Replace with:

```javascript
          const coalitionParties=computeCoalitionSatisfaction(r,cabinet);
          const nextR={...r,cabinet,coalitionParties,usedEventIds:[]};
          const {event,ownerSlot}=drawEventForRoom(nextR);
          roomRef().update({cabinet,coalitionParties,phase:'governing',turn:1,usedEventIds:[],currentEffectResult:null,currentEvent:event,currentEventOwnerSlot:ownerSlot,decisionHistory:[],noConfidenceVotes:{}}).catch(()=>{}).finally(releaseAutomation);
```

- [ ] **Step 5: `nextTurn()` — reset `noConfidenceVotes` only (a fresh event needs a fresh vote tally; `decisionHistory` must NOT be touched here — it persists across turns)**

Find this exact block:

```javascript
  const crisis=computeDeficitCrisis(r.treasury,r.deficitStreak||0,r.national);
  const nextR={...r,turn:r.turn+1,national:crisis.national};
  const {event,ownerSlot}=drawEventForRoom(nextR);
  roomRef().update({turn:r.turn+1,national:crisis.national,deficitStreak:crisis.deficitStreak,currentEvent:event,currentEventOwnerSlot:ownerSlot,currentEffectResult:null,queuedEvent:null});
}
```

Replace with:

```javascript
  const crisis=computeDeficitCrisis(r.treasury,r.deficitStreak||0,r.national);
  const nextR={...r,turn:r.turn+1,national:crisis.national};
  const {event,ownerSlot}=drawEventForRoom(nextR);
  roomRef().update({turn:r.turn+1,national:crisis.national,deficitStreak:crisis.deficitStreak,currentEvent:event,currentEventOwnerSlot:ownerSlot,currentEffectResult:null,queuedEvent:null,noConfidenceVotes:{}});
}
```

- [ ] **Step 6: Manually verify**

```bash
cd "/Users/panupanpitak/Projects/judtang-online" && python3 -m http.server 8800
```

Open `http://localhost:8800/index%20(3).html` via the Claude_Browser MCP tools, create a room, and confirm via console:

```javascript
db.collection('jadtang_rooms').doc(ui.roomCode).get().then(s=>console.log(s.data().decisionHistory, s.data().noConfidenceVotes))
```

Expected: `[] {}`. Then drive the room through to `governing` phase (via real play or direct Firestore seeding as used in earlier Group A/B/C1 verification) and confirm `confirmCabinet()` (or the AI-cabinet path, if the PM ends up AI-controlled) left both fields at `[] {}` right as `governing` phase begins, and that calling `nextTurn()` from the console at any point resets `noConfidenceVotes` to `{}` while leaving `decisionHistory` unchanged (seed a non-empty `decisionHistory` first via the console to confirm it survives).

- [ ] **Step 7: Commit**

```bash
git add "index (3).html"
git commit -m "Add decisionHistory/noConfidenceVotes fields with resets at every room-state transition"
```

---

### Task 2: History-based no-confidence effect — `countBadDecisions`, `computeNoConfidenceEffect`, wired into `chooseEvent()`

**Files:**
- Modify: `index (3).html` — add two helpers near `computeDeficitCrisis`; modify `chooseEvent()`; modify `PHASE_TEMPLATES.governing` to add the history badge

**Interfaces:**
- Consumes: `r.decisionHistory` (Task 1)
- Produces: `countBadDecisions(decisionHistory) -> number`, `computeNoConfidenceEffect(baseEffect, choiceIdx, decisionHistory) -> effect object`. Task 5 (AI voting) calls both.

- [ ] **Step 1: Validate the logic in isolation with Node**

```bash
node -e "
function countBadDecisions(decisionHistory){
  return (decisionHistory||[]).filter(d=>d.trustDelta<0).length;
}
function computeNoConfidenceEffect(baseEffect,choiceIdx,decisionHistory){
  const badCount=countBadDecisions(decisionHistory);
  const eff={...baseEffect};
  if(choiceIdx===0){
    eff.trust=Math.max(0,(eff.trust||0)-badCount);
  } else {
    eff.trust=(eff.trust||0)-badCount*2;
  }
  return eff;
}

// case 1: empty history -> no change
let c1=computeNoConfidenceEffect({treasury:-5,trust:5,satisfactionAll:10},0,[]);
console.assert(c1.trust===5,'case1: expected 5 got '+c1.trust);

// case 2: 3 bad decisions, choice 0 (defend) -> trust bonus shrinks
let c2=computeNoConfidenceEffect({treasury:-5,trust:5,satisfactionAll:10},0,[{trustDelta:-3},{trustDelta:-1},{trustDelta:5},{trustDelta:-2}]);
console.assert(countBadDecisions([{trustDelta:-3},{trustDelta:-1},{trustDelta:5},{trustDelta:-2}])===3,'case2 badCount');
console.assert(c2.trust===2,'case2: expected 5-3=2 got '+c2.trust);

// case 3: choice 0 never goes negative even with many bad decisions
let c3=computeNoConfidenceEffect({treasury:-5,trust:5,satisfactionAll:10},0,[{trustDelta:-1},{trustDelta:-1},{trustDelta:-1},{trustDelta:-1},{trustDelta:-1},{trustDelta:-1},{trustDelta:-1},{trustDelta:-1}]);
console.assert(c3.trust===0,'case3: expected floor 0 got '+c3.trust);

// case 4: 3 bad decisions, choice 1 (dismiss) -> extra penalty, double weight
let c4=computeNoConfidenceEffect({trust:-10,unrest:5,satisfactionAll:-15},1,[{trustDelta:-3},{trustDelta:-1},{trustDelta:-2}]);
console.assert(c4.trust===-16,'case4: expected -10-3*2=-16 got '+c4.trust);

// case 5: other effect fields pass through untouched
console.assert(c4.unrest===5 && c4.satisfactionAll===-15,'case5: other fields unchanged');

console.log('ALL PASS');
"
```

Expected output: `ALL PASS`.

- [ ] **Step 2: Add both helpers to `index (3).html`**

Find this exact block:

```javascript
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

Replace with:

```javascript
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
function countBadDecisions(decisionHistory){
  return (decisionHistory||[]).filter(d=>d.trustDelta<0).length;
}
function computeNoConfidenceEffect(baseEffect,choiceIdx,decisionHistory){
  const badCount=countBadDecisions(decisionHistory);
  const eff={...baseEffect};
  if(choiceIdx===0){
    eff.trust=Math.max(0,(eff.trust||0)-badCount);
  } else {
    eff.trust=(eff.trust||0)-badCount*2;
  }
  return eff;
}
```

- [ ] **Step 3: Wire into `chooseEvent()` — adjust effect and append to `decisionHistory`**

Find this exact block:

```javascript
function chooseEvent(choiceIdx){
  const r=ui.room;
  const ev=r.currentEvent;
  const choice=ev.choices[choiceIdx];
  const eff=choice.effect;
```

Replace with:

```javascript
function chooseEvent(choiceIdx){
  const r=ui.room;
  const ev=r.currentEvent;
  const choice=ev.choices[choiceIdx];
  let eff=choice.effect;
  if(ev.id==='noconf'){
    eff=computeNoConfidenceEffect(eff,choiceIdx,r.decisionHistory);
  }
```

Find this exact block (near the end of `chooseEvent()`):

```javascript
  update.popularity=popularity;
  update.currentEffectResult={text:`เลือก: ${choice.label} — ${choice.desc}`,tags};
  update.usedEventIds=[...r.usedEventIds,ev.id];
  if(ev._filedBy) update.queuedEvent=null;
```

Replace with:

```javascript
  update.popularity=popularity;
  update.currentEffectResult={text:`เลือก: ${choice.label} — ${choice.desc}`,tags};
  update.usedEventIds=[...r.usedEventIds,ev.id];
  update.decisionHistory=[...(r.decisionHistory||[]),{eventId:ev.id,choiceIndex:choiceIdx,trustDelta:eff.trust||0,turn:r.turn,term:r.term}];
  update.noConfidenceVotes={};
  if(ev._filedBy) update.queuedEvent=null;
```

- [ ] **Step 4: Add the history badge to `PHASE_TEMPLATES.governing`**

Find this exact block:

```javascript
        <p style="font-size:14px;line-height:1.7;">${ev.desc}</p>
        ${result?`
```

Replace with:

```javascript
        <p style="font-size:14px;line-height:1.7;">${ev.desc}</p>
        ${ev.id==='noconf'&&!result?`<div class="tag far" style="display:inline-block;margin-bottom:10px;">ประวัติการตัดสินใจที่กระทบความเชื่อมั่นเชิงลบ: ${countBadDecisions(r.decisionHistory)} ครั้ง</div>`:''}
        ${result?`
```

- [ ] **Step 5: Manually verify in browser**

Using the local server from Task 1, drive a room to `governing` phase as the sole human PM (others AI). Play a few turns choosing options with negative `trust` effects to build up `decisionHistory` with at least 3 entries where `trustDelta<0` — confirm via console:

```javascript
db.collection('jadtang_rooms').doc(ui.roomCode).get().then(s=>console.log(s.data().decisionHistory))
```

Then force a `noconf` event onto the room via console (e.g. `roomRef().update({currentEvent:{...EVENTS.find(e=>e.id==='noconf')},currentEffectResult:null})`) and confirm: the badge shows the correct bad-decision count; picking either choice produces a `currentEffectResult` whose `ความเชื่อมั่น` tag reflects the history-adjusted number (not the raw `5`/`-10` base), matching the Step 1 Node formulas; and `decisionHistory` gained one more entry for this noconf resolution too.

- [ ] **Step 6: Commit**

```bash
git add "index (3).html"
git commit -m "Scale no-confidence response effect by the government's decision history"
```

---

### Task 3: Coalition vote-casting and seat-weighted tally

**Files:**
- Modify: `index (3).html` — add `castNoConfidenceVote()`, `tallyNoConfidenceWinner()`, `resolveNoConfidenceVote()` near `fileNoConfidence()`

**Interfaces:**
- Consumes: `r.noConfidenceVotes`, `r.coalitionSlots`, `r.lastElection.results`, `r.pm` (all pre-existing or from Task 1); calls the pre-existing `chooseEvent(choiceIdx)`
- Produces: `castNoConfidenceVote(choiceIdx, slotId?)`, `tallyNoConfidenceWinner(r) -> 0|1`, `resolveNoConfidenceVote()`. Task 4 (UI) calls `castNoConfidenceVote`; Task 5 (AI automation) calls `castNoConfidenceVote` and `resolveNoConfidenceVote`.

- [ ] **Step 1: Validate `tallyNoConfidenceWinner` in isolation with Node**

```bash
node -e "
function tallyNoConfidenceWinner(r){
  const votes=r.noConfidenceVotes||{};
  let seatsFor=[0,0];
  r.coalitionSlots.forEach(id=>{
    const idx=votes[id];
    if(idx===undefined) return;
    const seats=(r.lastElection.results.find(x=>x.id===id)?.seats||0);
    seatsFor[idx]+=seats;
  });
  if(seatsFor[0]>seatsFor[1]) return 0;
  if(seatsFor[1]>seatsFor[0]) return 1;
  return votes[r.pm];
}

const results=[{id:'s0',seats:60},{id:'s1',seats:30},{id:'s2',seats:10}];

// case 1: choice 0 wins on seats
let r1={pm:'s0',coalitionSlots:['s0','s1','s2'],noConfidenceVotes:{s0:0,s1:0,s2:1},lastElection:{results}};
console.assert(tallyNoConfidenceWinner(r1)===0,'case1: expected 0 got '+tallyNoConfidenceWinner(r1));

// case 2: choice 1 wins on seats
let r2={pm:'s0',coalitionSlots:['s0','s1','s2'],noConfidenceVotes:{s0:1,s1:1,s2:0},lastElection:{results}};
console.assert(tallyNoConfidenceWinner(r2)===1,'case2: expected 1 got '+tallyNoConfidenceWinner(r2));

// case 3: exact tie -> PM's own vote decides
const tieResults=[{id:'s0',seats:30},{id:'s1',seats:30}];
let r3a={pm:'s0',coalitionSlots:['s0','s1'],noConfidenceVotes:{s0:1,s1:0},lastElection:{results:tieResults}};
console.assert(tallyNoConfidenceWinner(r3a)===1,'case3a: PM voted 1, expected 1 got '+tallyNoConfidenceWinner(r3a));
let r3b={pm:'s0',coalitionSlots:['s0','s1'],noConfidenceVotes:{s0:0,s1:1},lastElection:{results:tieResults}};
console.assert(tallyNoConfidenceWinner(r3b)===0,'case3b: PM voted 0, expected 0 got '+tallyNoConfidenceWinner(r3b));

// case 4: single-party majority government (PM only) -> PM's own vote always wins
let r4={pm:'s0',coalitionSlots:['s0'],noConfidenceVotes:{s0:1},lastElection:{results}};
console.assert(tallyNoConfidenceWinner(r4)===1,'case4: expected 1 got '+tallyNoConfidenceWinner(r4));

console.log('ALL PASS');
"
```

Expected output: `ALL PASS`.

- [ ] **Step 2: Add all three functions to `index (3).html`**

Find this exact block (the end of `fileNoConfidence()`):

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

Replace with (unchanged `fileNoConfidence`, adds the three new functions right after):

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
function castNoConfidenceVote(choiceIdx,slotId){
  const r=ui.room;
  const mine=mySlot();
  const targetId=slotId||(mine&&mine.id);
  if(!targetId) return;
  if(!r.coalitionSlots.includes(targetId)) return;
  if((r.noConfidenceVotes||{})[targetId]!==undefined) return;
  const noConfidenceVotes={...(r.noConfidenceVotes||{}),[targetId]:choiceIdx};
  roomRef().update({noConfidenceVotes});
}
function tallyNoConfidenceWinner(r){
  const votes=r.noConfidenceVotes||{};
  let seatsFor=[0,0];
  r.coalitionSlots.forEach(id=>{
    const idx=votes[id];
    if(idx===undefined) return;
    const seats=(r.lastElection.results.find(x=>x.id===id)?.seats||0);
    seatsFor[idx]+=seats;
  });
  if(seatsFor[0]>seatsFor[1]) return 0;
  if(seatsFor[1]>seatsFor[0]) return 1;
  return votes[r.pm];
}
function resolveNoConfidenceVote(){
  const r=ui.room;
  chooseEvent(tallyNoConfidenceWinner(r));
}
```

- [ ] **Step 3: Manually verify in browser**

Using the local server, seed a room in `governing` phase with a `noconf` event current, a multi-party coalition, and an empty `noConfidenceVotes`. From the console (or as two simulated players, one per tab), call `castNoConfidenceVote(0)` as one coalition party and `castNoConfidenceVote(1)` as another (use the `slotId` second argument to cast on behalf of a party you're not currently controlling, to speed up the test without needing extra tabs). Confirm:
- Each call updates `noConfidenceVotes` correctly and refuses a second vote from the same party (call `castNoConfidenceVote` again for a party that already voted — confirm the room doc is unchanged).
- Once all coalition parties have voted, call `resolveNoConfidenceVote()` directly from the console and confirm `chooseEvent` actually ran (check `currentEffectResult` is now populated, `noConfidenceVotes` reset to `{}`, and `decisionHistory` gained an entry).

- [ ] **Step 4: Commit**

```bash
git add "index (3).html"
git commit -m "Add seat-weighted coalition vote casting and tally for no-confidence motions"
```

---

### Task 4: Voting UI — `noConfidenceVotingPanel` wired into the governing screen

**Files:**
- Modify: `index (3).html` — add `noConfidenceVotingPanel()` near `gauge()`; modify `PHASE_TEMPLATES.governing`

**Interfaces:**
- Consumes: `castNoConfidenceVote(choiceIdx)` (Task 3, called with no `slotId` — the panel is only ever rendered for the current player casting their own vote)
- Produces: nothing new consumed by later tasks — this is the last piece needed for a human to fully interact with the feature via the UI.

- [ ] **Step 1: Add `noConfidenceVotingPanel()`**

Find this exact block (the end of `gauge()`):

```javascript
function gauge(label,value,inverse){
  value=clamp(value,0,100);
  let cls='good';
  if(inverse){ cls = value>65?'bad':value>35?'warn':'good'; } else { cls = value<35?'bad':value<65?'warn':'good'; }
  return `<div class="gauge"><div class="gauge-top"><span class="lbl">${label}</span><span class="val">${fmt(value)}</span></div><div class="gauge-bar"><div class="gauge-fill ${cls}" style="width:${value}%"></div></div></div>`;
}
```

Replace with (unchanged `gauge`, adds `noConfidenceVotingPanel` right after):

```javascript
function gauge(label,value,inverse){
  value=clamp(value,0,100);
  let cls='good';
  if(inverse){ cls = value>65?'bad':value>35?'warn':'good'; } else { cls = value<35?'bad':value<65?'warn':'good'; }
  return `<div class="gauge"><div class="gauge-top"><span class="lbl">${label}</span><span class="val">${fmt(value)}</span></div><div class="gauge-bar"><div class="gauge-fill ${cls}" style="width:${value}%"></div></div></div>`;
}
function noConfidenceVotingPanel(r,mine,ev){
  const votes=r.noConfidenceVotes||{};
  const iAmVoter=mine&&r.coalitionSlots.includes(mine.id);
  const myVote=mine?votes[mine.id]:undefined;
  let seatsFor=[0,0];
  r.coalitionSlots.forEach(id=>{
    const idx=votes[id];
    if(idx===undefined) return;
    const seats=(r.lastElection.results.find(x=>x.id===id)?.seats||0);
    seatsFor[idx]+=seats;
  });
  const rows=r.coalitionSlots.map(id=>{
    const s=r.slots.find(x=>x.id===id);
    const idx=votes[id];
    const label=idx===undefined?'ยังไม่โหวต':ev.choices[idx].label;
    const cls=idx===undefined?'pending':(idx===0?'accepted':'declined');
    return `<div class="party-row"><span class="dot" style="background:${s.color}"></span><div class="info"><div class="pname">${s.name}${s.claimedBy?' ('+(s.playerDisplayName||'ไม่ทราบชื่อ')+')':' (AI)'}</div></div><span class="tag ${cls}">${label}</span></div>`;
  }).join('');
  return `
    ${iAmVoter&&myVote===undefined?`
      ${ev.choices.map((c,i)=>`<button class="event-choice" onclick="castNoConfidenceVote(${i})"><span class="clabel">${c.label}</span><span class="cdesc">${c.desc}</span></button>`).join('')}
    `:''}
    <div style="margin-top:14px;">${rows}</div>
    <div class="majority-line" style="margin-top:10px;">คะแนนเสียงพรรคร่วม: ${ev.choices[0].label} = <strong class="mono">${seatsFor[0]}</strong> ที่นั่ง &middot; ${ev.choices[1].label} = <strong class="mono">${seatsFor[1]}</strong> ที่นั่ง</div>
  `;
}
```

- [ ] **Step 2: Wire it into `PHASE_TEMPLATES.governing`**

Find this exact block:

```javascript
        `:(isMyTurn?`
          ${ev.choices.map((c,i)=>`<button class="event-choice" onclick="chooseEvent(${i})"><span class="clabel">${c.label}</span><span class="cdesc">${c.desc}</span></button>`).join('')}
        `:`<div class="waiting-box">รอ ${ownerSlot.name}${ownerSlot.claimedBy?'':' (AI)'} ตัดสินใจ...</div>`)}
      </div>
```

Replace with:

```javascript
        `:(ev.id==='noconf'?noConfidenceVotingPanel(r,mine,ev):(isMyTurn?`
          ${ev.choices.map((c,i)=>`<button class="event-choice" onclick="chooseEvent(${i})"><span class="clabel">${c.label}</span><span class="cdesc">${c.desc}</span></button>`).join('')}
        `:`<div class="waiting-box">รอ ${ownerSlot.name}${ownerSlot.claimedBy?'':' (AI)'} ตัดสินใจ...</div>`))}
      </div>
```

- [ ] **Step 3: Manually verify with two tabs**

Using two tabs on a room with a 2+ human-party coalition and a current `noconf` event (seed via console as in earlier tasks if steering there organically is slow):

1. Confirm each coalition-party row shows a "ยังไม่โหวต" (pending, yellow) tag initially, plus the party's player name if human.
2. In tab A, click one of the two choice buttons. Confirm: tab A's own row now shows the chosen label in an "accepted" (green) or "declined" (red) tag depending on which button was clicked, the choice buttons disappear for tab A (already voted), and tab B sees tab A's vote update within about a second without reloading.
3. Confirm the seat-tally line (`คะแนนเสียงพรรคร่วม: ... = N ที่นั่ง`) updates correctly as votes come in.
4. In tab B, cast the second vote — confirm the event auto-resolves shortly after (via Task 3's `resolveNoConfidenceVote`, triggered through the existing snapshot-driven `runHostAutomation()` path if tab A happens to be host, or note if this doesn't yet resolve because Task 5's AI-facing automation trigger hasn't been added — that's expected at this point in the plan since Task 5 hasn't run yet; if BOTH parties are human and the resolution still doesn't fire, that's a genuine gap to flag, since `resolveNoConfidenceVote` needs *some* trigger even for all-human coalitions).

Note: at this point in the plan (before Task 5), there is no automatic call to `resolveNoConfidenceVote()` anywhere — nothing currently calls it except your manual Task 3 console test. If your two-tab test in this step never resolves the event on its own, that is expected — the automatic trigger is Task 5's job. Don't treat that as a Task 4 bug; just confirm the vote-casting UI itself (rows, tags, tallies, button disappearance) behaves correctly, and manually call `resolveNoConfidenceVote()` from the console to confirm the eventual transition to a resolved `result` renders correctly.

- [ ] **Step 4: Commit**

```bash
git add "index (3).html"
git commit -m "Add live coalition vote-tally UI for no-confidence motions"
```

---

### Task 5: AI coalition partners auto-vote; automatic resolution once everyone has voted

**Files:**
- Modify: `index (3).html` — the `governing` branch inside `runHostAutomation()`

**Interfaces:**
- Consumes: `castNoConfidenceVote(choiceIdx, slotId)` and `resolveNoConfidenceVote()` (Task 3), `computeNoConfidenceEffect(baseEffect, choiceIdx, decisionHistory)` (Task 2)

- [ ] **Step 1: Add the no-confidence voting/resolution branch ahead of the existing single-owner automation**

Find this exact block:

```javascript
  if(r.phase==='governing'){
    const ownerSlot=r.slots.find(s=>s.id===r.currentEventOwnerSlot);
    if(ownerSlot && !ownerSlot.claimedBy){
      automationBusy=true;
      armAutomationWatchdog();
      setTimeout(()=>{
        try{
          if(!r.currentEffectResult){
            const choices=r.currentEvent.choices;
            const scored=choices.map((c,i)=>({i,score:(c.effect.trust||0)*2-(c.effect.treasury<0?Math.abs(c.effect.treasury)*0.3:0)+(Math.random()*10-5)}));
            scored.sort((a,b)=>b.score-a.score);
            chooseEvent(scored[0].i);
          } else {
            nextTurn();
          }
        } catch(e){ console.error('AI governing automation failed:',e); }
        finally { releaseAutomation(); }
      },900);
    }
  }
}
```

Replace with:

```javascript
  if(r.phase==='governing'){
    if(r.currentEvent && r.currentEvent.id==='noconf' && !r.currentEffectResult){
      const votes=r.noConfidenceVotes||{};
      const missingAiVoter=r.coalitionSlots.map(id=>r.slots.find(s=>s.id===id)).find(s=>s&&!s.claimedBy&&votes[s.id]===undefined);
      if(missingAiVoter){
        automationBusy=true;
        armAutomationWatchdog();
        setTimeout(()=>{
          try{
            const choices=r.currentEvent.choices;
            const scored=choices.map((c,i)=>{
              const eff=computeNoConfidenceEffect(c.effect,i,r.decisionHistory);
              return {i,score:(eff.trust||0)*2-(eff.treasury<0?Math.abs(eff.treasury)*0.3:0)+(Math.random()*10-5)};
            });
            scored.sort((a,b)=>b.score-a.score);
            castNoConfidenceVote(scored[0].i,missingAiVoter.id);
          } catch(e){ console.error('AI no-confidence vote failed:',e); }
          finally { releaseAutomation(); }
        },700);
        return;
      }
      const allVoted=r.coalitionSlots.every(id=>votes[id]!==undefined);
      if(allVoted){
        automationBusy=true;
        armAutomationWatchdog();
        setTimeout(()=>{
          try{ resolveNoConfidenceVote(); }
          catch(e){ console.error('No-confidence vote resolution failed:',e); }
          finally { releaseAutomation(); }
        },500);
      }
      return;
    }
    const ownerSlot=r.slots.find(s=>s.id===r.currentEventOwnerSlot);
    if(ownerSlot && !ownerSlot.claimedBy){
      automationBusy=true;
      armAutomationWatchdog();
      setTimeout(()=>{
        try{
          if(!r.currentEffectResult){
            const choices=r.currentEvent.choices;
            const scored=choices.map((c,i)=>({i,score:(c.effect.trust||0)*2-(c.effect.treasury<0?Math.abs(c.effect.treasury)*0.3:0)+(Math.random()*10-5)}));
            scored.sort((a,b)=>b.score-a.score);
            chooseEvent(scored[0].i);
          } else {
            nextTurn();
          }
        } catch(e){ console.error('AI governing automation failed:',e); }
        finally { releaseAutomation(); }
      },900);
    }
  }
}
```

- [ ] **Step 2: Manually verify with an all-AI and a mixed coalition**

Using the local server:

1. **All-AI coalition partners:** seed a room in `governing` phase with a `noconf` event current, `pm` claimed by a human, and 2+ AI-controlled (unclaimed) coalition partners. Don't touch anything — poll the room doc every second or two:
   ```javascript
   setInterval(()=>db.collection('jadtang_rooms').doc(ui.roomCode).get().then(s=>console.log(s.data().noConfidenceVotes, s.data().currentEffectResult)),1000)
   ```
   Confirm votes appear for the AI parties on their own within a couple of automation cycles, and once all coalition slots (including the human PM — you'll need to cast the PM's own vote yourself for this to complete) have voted, `currentEffectResult` populates automatically without you calling `resolveNoConfidenceVote()` yourself.
2. **Mixed coalition (repeat Task 4's two-tab test):** with both parties human, confirm that once both have voted, the event now resolves automatically (this closes the gap Task 4's Step 3 flagged as "expected, not yet automatic").
3. Confirm no console errors in either scenario.

- [ ] **Step 3: Commit**

```bash
git add "index (3).html"
git commit -m "Have AI coalition partners auto-vote and auto-resolve no-confidence motions"
```

---

### Task 6: Full regression pass across Group C2

**Files:**
- None (verification only)

- [ ] **Step 1: Play one full election → coalition → cabinet → governing flow with two human tabs, exercising both features together**

Using two tabs (host + second player), form a 2-party coalition, reach `governing`, and:
1. Play a few non-`noconf` events choosing options with negative trust effects to build `decisionHistory` with several bad entries — confirm those events still work exactly as before (single owner decides, no voting UI appears for them).
2. Trigger (or seed) a `noconf` event — confirm the history badge shows the correct count, both coalition parties can vote and see each other's votes live, the seat tally is correct, and the effect that lands (once resolved) reflects the history-adjusted numbers from Task 2's formula given the actual `badCount` at that point.
3. Force an exact-tie scenario (adjust `lastElection.results`/coalition composition via console to make both parties' seats equal) and confirm the PM's own vote decides the outcome.
4. Confirm the PM alone can click "ดำเนินการต่อ" after the result to advance the turn, exactly as before this branch.
5. Confirm no console errors throughout (`mcp__Claude_Browser__read_console_messages`).

- [ ] **Step 2: Commit (only if a fixup was needed)**

If no fixups were needed, this task requires no commit. If a bug surfaced, fix it, re-run Step 1, then:

```bash
git add "index (3).html"
git commit -m "Fix regression found in Group C2 end-to-end verification"
```
