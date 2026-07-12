# Group D: ศาลรัฐธรรมนูญวินิจฉัย + อำนาจยุบสภา — Design Spec

**วันที่:** 2026-07-12
**ขอบเขต:** เพิ่มกลไก "ศาลรัฐธรรมนูญวินิจฉัย" (#4 จากรายการเดิม) ในเกม "จัดตั้ง ออนไลน์" ([index (3).html](../../../index%20(3).html)) — ต่อยอดจาก Group A/B/C1/C2 ที่ merge เข้า `main` แล้ว (commit `9680fc6`) พร้อมฟีเจอร์ย่อยที่เกิดขึ้นระหว่างคุยดีไซน์: **อำนาจยุบสภาของ PM**

**ไม่รวมในสโคปนี้:** Group E (วิสัยทัศน์ระยะยาว)

---

## ภาพรวมกลไก

ฝ่ายค้านยื่นคำร้องต่อศาลรัฐธรรมนูญให้วินิจฉัยกรณีทุจริตของ PM หรือรัฐมนตรีคนใดคนหนึ่ง — **ศาลเป็นระบบภายนอก ไม่ใช่ผู้เล่นคนไหนตัดสิน** ผลวินิจฉัย (ผิด/ไม่ผิด) สุ่มถ่วงน้ำหนักตามจำนวนครั้งที่เป้าหมายเคย "ปกป้อง ไม่ตั้งกรรมการ" ในอีเวนต์ทุจริตที่เคยเกิดกับตัวเอง

ถ้าวินิจฉัยว่าผิด: รัฐมนตรีเสียตำแหน่งกระทรวง (ว่างตลอดสมัย) ส่วน PM ต้องหา PM ใหม่กลางสมัย (ใช้ระบบ coalition/cabinet ที่มีอยู่แล้วทั้งหมด โดยคงไตรมาสที่เล่นค้างไว้ ไม่รีสตาร์ทสมัย)

เพิ่มเติม: PM มีอำนาจ **ยุบสภาได้ทุกเมื่อไม่มีเงื่อนไข** จบสมัยทันทีเข้าสู่การเลือกตั้งใหม่

---

## 1. Data model ใหม่

```
corruptionIncidents: []            // {targetId, term, turn, choiceIndex} — ไม่รีเซ็ตข้ามรัฐบาล/สมัย ติดตัวพรรคไปตลอดเกม (คนละความหมายกับ decisionHistory)
pendingPmDisqualification: null    // {targetId} — ตั้งชั่วคราวรอกด "ดำเนินการต่อ" หลังศาลตัดสินว่า PM ผิด
midTermReshuffle: false            // บอก confirmCabinet()/AI-cabinet-automation ว่านี่คือ reshuffle กลางสมัย ไม่ใช่รัฐบาลใหม่ต้นสมัย
dissolvedByPm: false               // ใช้แสดงข้อความเฉพาะในหน้า term_summary
```

`oppositionActions[partyId]` เดิม (มี `noConfidenceUsedThisTerm`) ขยายเพิ่ม `courtPetitionUsedThisTerm:true` — **ต้อง merge ไม่ overwrite** (ดูข้อ 6 เรื่องบั๊กที่ต้องแก้ไปพร้อมกัน)

### จุดที่ต้องตั้งค่าเริ่มต้น/รีเซ็ต

| ฟังก์ชัน | `corruptionIncidents` | `pendingPmDisqualification` | `midTermReshuffle` | `dissolvedByPm` |
|---|---|---|---|---|
| `handleCreateRoom()` | `[]` | `null` | `false` | `false` |
| `resetRoomToLobby()` | `[]` | `null` | `false` | `false` |
| `startNewTerm()` | ไม่แตะ (ติดตัวพรรคข้ามสมัย) | ไม่แตะ | ไม่แตะ | `false` |
| `confirmCabinet()` / AI-cabinet automation | ไม่แตะ | ไม่แตะ | `false` (เคลียร์หลังใช้) | ไม่แตะ |
| `nextTurn()` (ตอนเข้า reshuffle) | ไม่แตะ | `null` (เคลียร์ทันทีที่ประมวลผล) | `true` (ตั้งก่อนเข้า coalition/cabinet) | ไม่แตะ |

---

## 2. ขยายอีเวนต์ทุจริตให้เกิดกับ PM ได้ด้วย

`drawEventForRoom()`: เปลี่ยนจากสุ่มเฉพาะกระทรวงของพรรคร่วม (`partnerMinistries`) เป็นสุ่มจาก**ทุกกระทรวงที่มีเจ้ากระทรวงแล้ว รวมของ PM เอง** — ผู้ตัดสินใจ (`ownerSlot`) ยังเป็น PM เสมอเหมือนเดิม (ไม่เปลี่ยน)

เพิ่ม field `_corruptionTarget: ownerId` เข้าไปในอ็อบเจกต์อีเวนต์ corruption ที่สร้างแบบ dynamic — **จำเป็นต้องมี** เพราะตอนนี้ `chooseEvent()` ไม่มีทางรู้ว่าใครถูกพาดพิง (มีแต่ข้อความใน title/desc) ต้องมี field ให้ query ได้เพื่อบันทึกลง `corruptionIncidents`

ทุกครั้งที่อีเวนต์ `corruption` ถูกแก้ไข (`chooseEvent`) → เพิ่มรายการเข้า `corruptionIncidents`:
```javascript
if(ev.id==='corruption'){
  update.corruptionIncidents=[...(r.corruptionIncidents||[]),{targetId:ev._corruptionTarget,term:r.term,turn:r.turn,choiceIndex:choiceIdx}];
}
```

---

## 3. เกณฑ์ยื่นคำร้อง + เลือกเป้าหมาย

```javascript
function countGuiltyIncidents(corruptionIncidents,targetId){
  return (corruptionIncidents||[]).filter(d=>d.targetId===targetId && d.choiceIndex===1).length;
}
```
นับเฉพาะครั้งที่เป้าหมายเลือก "ปกป้องพรรคร่วมรัฐบาล ไม่ตั้งกรรมการ" (`choiceIndex===1`) — เลือก "ตั้งกรรมการสอบสวนเอง" (`choiceIndex===0`) ไม่นับเป็นมูล

ยื่นได้เมื่อ `countGuiltyIncidents(...)>=1` — ฝ่ายค้านเลือกเป้าหมายจากรายชื่อผู้มีมูลได้ (ถ้าไม่มีใครมีมูลเลย แสดงข้อความอธิบายแทนปุ่ม) จำกัด **1 ครั้งต่อสมัยต่อพรรคฝ่ายค้าน** ผ่าน `oppositionActions[partyId].courtPetitionUsedThisTerm`

`r.queuedEvent` ใช้ร่วมกับญัตติไม่ไว้วางใจ โดยเพิ่ม field `type`:
```javascript
{filedBy, type:'noconf'}                    // ญัตติไม่ไว้วางใจ (เพิ่ม type ให้ fileNoConfidence() ด้วยเพื่อความชัดเจน แม้ค่า default เดิมจะทำงานถูกอยู่แล้ว)
{filedBy, targetId, type:'courtcase'}       // คำร้องศาลรัฐธรรมนูญ
```

---

## 4. อีเวนต์ "courtcase" — ไม่มีตัวเลือกจริง มีแค่ปุ่มรับทราบ

`drawEventForRoom()` เพิ่ม branch สำหรับ `r.queuedEvent.type==='courtcase'`:
```javascript
{
  id:'courtcase',
  title:'ศาลรัฐธรรมนูญวินิจฉัยคดี'+(targetSlot?targetSlot.name:''),
  desc:`${filerSlot?filerSlot.name:'ฝ่ายค้าน'}ยื่นคำร้องต่อศาลรัฐธรรมนูญให้วินิจฉัยกรณีทุจริตของ${targetSlot?targetSlot.name:'พรรคที่ถูกกล่าวหา'} ศาลรับคำร้องไว้พิจารณาแล้ว`,
  choices:[{label:'รับทราบคำวินิจฉัยของศาล',desc:'ศาลได้พิจารณาและมีคำวินิจฉัยแล้ว',effect:{}}],
  _filedBy:r.queuedEvent.filedBy,           // ใช้ร่วมกับ motionPending check เดิมที่เช็ค ev._filedBy อยู่แล้ว ไม่ต้องแก้ตรงนั้น
  _courtPetition:{targetId:r.queuedEvent.targetId},
}
```
ownerSlot = `r.pm` เสมอ (เหมือน corruption event — PM เป็นผู้ "รับทราบ" คำวินิจฉัยแทนรัฐบาล แม้เป้าหมายจะเป็น PM เองก็ตาม เพื่อไม่ต้องสร้าง logic เปลี่ยนเจ้าของ turn ใหม่)

**`chooseEvent()`** เพิ่ม early-return delegation ที่ต้นฟังก์ชัน:
```javascript
function chooseEvent(choiceIdx){
  const r=ui.room;
  const ev=r.currentEvent;
  if(ev.id==='courtcase'){ resolveCourtCase(); return; }
  ...โค้ดเดิมทั้งหมด ไม่เปลี่ยน...
```
เหตุผลที่แยกไปฟังก์ชันใหม่ทั้งหมดแทนที่จะ inject เข้า pipeline เดิมแบบ noconf: courtcase ไม่มี `choice.effect` จริงให้คำนวณ (`effect:{}` ว่างเปล่า) ผลลัพธ์ทั้งหมดมาจากการสุ่มวินิจฉัย ไม่ใช่ตัวเลือกที่ผู้เล่นกด จึงไม่ต้องพึ่ง logic คำนวณ treasury/national/tags/satisfactionAll ทั่วไปของ `chooseEvent()` เลย

### `resolveCourtCase()`
```javascript
function computeCourtVerdictProbability(corruptionIncidents,targetId){
  return clamp(0.4+0.15*countGuiltyIncidents(corruptionIncidents,targetId),0,0.95);
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
**หมายเหตุการออกแบบ:** `resolveCourtCase()` ไม่แตะ `decisionHistory` เลย (courtcase ไม่ใช่ "การตัดสินใจของรัฐบาล" แต่เป็นคำวินิจฉัยจากภายนอก) และไม่เพิ่ม `'courtcase'` เข้า `usedEventIds` ก็ได้เพราะ courtcase ไม่ได้อยู่ใน `EVENTS` pool ที่ต้องกันซ้ำ — ตัวจำกัดความถี่จริงคือ `oppositionActions[partyId].courtPetitionUsedThisTerm` เท่านั้น

---

## 5. Mid-term PM reshuffle (แทนที่การจบสมัยทันที)

**`nextTurn()`** เพิ่มเงื่อนไขใหม่เป็นอันดับแรก (ก่อน `pendingCollapse`/`turn>=turnsPerTerm` เดิม):
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
    ...โค้ดเดิม ไม่เปลี่ยน...
```
ใช้ `nextPmCandidate()` และ pattern การเช็ค `seats>=MAJORITY` เดียวกับ `proceedAfterElection()` ทุกประการ (reuse ไม่เขียนใหม่) — ถ้าไม่มีผู้สมัคร PM เหลือเลย (edge case สุดขั้ว) fallback กลับไปจบสมัยแบบ `collapsed`

**`confirmCabinet()`** เช็ค flag `midTermReshuffle` เพื่อตัดสินว่าจะรีเซ็ต `turn`/`usedEventIds` หรือไม่:
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
`decisionHistory` **รีเซ็ตเสมอ** ไม่ว่าจะ reshuffle หรือไม่ (รัฐบาลชุดใหม่เริ่มนับสถิติใหม่ — ตรงนี้ไม่เปลี่ยนจากพฤติกรรมเดิม) ส่วน `turn`/`usedEventIds` เท่านั้นที่แยกพฤติกรรมตาม `isReshuffle`

**AI-cabinet-automation branch** ใน `runHostAutomation()` (กรณี PM คนใหม่หลัง reshuffle เป็น AI) ต้องแก้ให้สอดคล้องกันทุกประการ — เช็ค `isReshuffle` แบบเดียวกัน

---

## 6. แก้บั๊กที่ต้องทำไปพร้อมกัน: `oppositionActions` merge ผิด

`fileNoConfidence()` ปัจจุบัน:
```javascript
const oppositionActions={...(r.oppositionActions||{}),[mine.id]:{noConfidenceUsedThisTerm:true}};
```
**overwrite** อ็อบเจกต์ทั้งก้อนของพรรคนั้น แทนที่จะ merge — ถ้าพรรคนี้เคยตั้ง `courtPetitionUsedThisTerm:true` ไว้ก่อนหน้า (ใหม่จาก spec นี้) การยื่นญัตติไม่ไว้วางใจทีหลังจะ**ลบล้าง**สถานะนั้นทิ้งโดยไม่ตั้งใจ ต้องแก้เป็น:
```javascript
const oppositionActions={...(r.oppositionActions||{}),[mine.id]:{...(r.oppositionActions?.[mine.id]||{}),noConfidenceUsedThisTerm:true}};
```
และ `fileCourtPetition()` ที่เพิ่มใหม่ต้อง merge แบบเดียวกันตั้งแต่ต้น (ดูข้อ 7)

---

## 7. Action function ใหม่: `fileCourtPetition(targetId)`

```javascript
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
โครงสร้าง guard เหมือน `fileNoConfidence()` ทุกประการ (คนละ flag เท่านั้น) — `motionPending` check เดิมในหน้า UI (`!!r.queuedEvent || (ev && ev._filedBy)`) ครอบคลุมทั้งสองประเภทอัตโนมัติเพราะ courtcase event ก็ตั้ง `_filedBy` เหมือนกัน ไม่ต้องแก้ตรงนั้น

---

## 8. ฟีเจอร์ย่อย: อำนาจยุบสภาของ PM

```javascript
function dissolveParliament(){
  const r=ui.room; const mine=mySlot();
  if(!mine || mine.id!==r.pm) return;
  if(!confirm('ยืนยันยุบสภา? สมัยนี้จะสิ้นสุดทันทีและเข้าสู่การเลือกตั้งใหม่')) return;
  stampAndThen('ยุบสภาแล้ว',()=>{
    roomRef().update({phase:'term_summary',collapsed:false,dissolvedByPm:true});
  });
}
```
ไม่มีเงื่อนไข/ต้นทุนใดๆ ตามที่ตกลง — PM กดได้ทุกเมื่อระหว่าง `governing` phase ไม่ว่าจะอยู่ไตรมาสไหน

**`PHASE_TEMPLATES.term_summary`** ต้องแยกข้อความ 3 แบบแทนที่ตรรกะ 2 แบบเดิม (`collapsed` เฉยๆ):
```javascript
<div class="term-badge">${r.dissolvedByPm?'PM ประกาศยุบสภา':(collapsed?'รัฐบาลสิ้นสุดก่อนกำหนด':'ครบวาระการดำรงตำแหน่ง')}</div>
<h1 class="serif" style="font-size:24px;margin:10px 0;">${r.dissolvedByPm?'ยุบสภา — เข้าสู่การเลือกตั้งใหม่':(collapsed?'รัฐบาลล่ม — พรรคร่วมถอนตัว':'สิ้นสุดสมัยที่ '+r.term)}</h1>
```
และข้อความอธิบายด้านล่าง (`<p style="font-size:13px...">`) เพิ่ม branch แรกสำหรับ `dissolvedByPm` เช่นกัน (ก่อน `collapsed`)

**`startNewTerm()`** เพิ่ม `dissolvedByPm:false` เข้า update object เพื่อเคลียร์ก่อนสมัยใหม่เริ่ม (ไม่งั้นข้อความจะค้างโชว์ผิดในสมัยถัดไปถ้าเกิด `collapsed` ปกติ)

---

## 9. UI: หน้า governing

### พาเนล "พรรคร่วมรัฐบาล" — เพิ่มปุ่มยุบสภา (เฉพาะ PM)
```javascript
${mine && mine.id===r.pm?`<button class="btn btn-ghost" style="width:100%;margin-top:10px;" onclick="dissolveParliament()">ยุบสภา</button>`:''}
```

### พาเนล "บทบาทฝ่ายค้าน" — เพิ่มส่วนยื่นคำร้องศาล
```javascript
<div style="margin-top:14px;border-top:1px solid var(--navy-line);padding-top:12px;">
  <div style="font-size:12.5px;color:var(--muted-on-navy);margin-bottom:8px;">ยื่นคำร้องต่อศาลรัฐธรรมนูญวินิจฉัยกรณีทุจริตที่มีมูล</div>
  ${eligibleCourtTargets.length?eligibleCourtTargets.map(t=>`<button class="btn btn-ghost" style="width:100%;margin-bottom:6px;" ${canFileCourtPetition?'':'disabled'} onclick="fileCourtPetition('${t.id}')">ยื่นคำร้องต่อ ${t.name}</button>`).join(''):'<div style="font-size:11px;color:var(--muted-on-navy);">ยังไม่มีพรรคใดมีมูลทุจริตให้ยื่นคำร้องได้</div>'}
  ${eligibleCourtTargets.length?`<div style="font-size:11px;color:var(--muted-on-navy);margin-top:4px;">${usedCourtPetition?'ใช้สิทธิ์ในสมัยนี้ไปแล้ว':(motionPending?'มีคำร้อง/ญัตติค้างอยู่ในคิว':'ใช้ได้ 1 ครั้งต่อสมัย')}</div>`:''}
</div>
```
ตัวแปรที่ต้องคำนวณเพิ่มที่ต้นฟังก์ชัน `PHASE_TEMPLATES.governing`:
```javascript
const eligibleCourtTargets=r.slots.filter(s=>countGuiltyIncidents(r.corruptionIncidents,s.id)>=1);
const usedCourtPetition=mine&&r.oppositionActions?.[mine.id]?.courtPetitionUsedThisTerm;
const canFileCourtPetition=isOpposition&&!usedCourtPetition&&!motionPending;
```

### Badge บอกมูลระหว่างดู courtcase event (เสริม)
ไม่จำเป็นเพิ่มเพราะ `desc` ของ courtcase event บอกชัดอยู่แล้วว่าใครถูกกล่าวหา — ไม่ต้องมี badge แยกเหมือน noconf

---

## Edge cases

- ถ้า `nextPmCandidate()` ไม่พบผู้สมัคร PM คนใหม่เลยหลัง PM ถูกตัดสิทธิ์ (ทุกพรรคมีที่นั่ง 0 ยกเว้นพรรคที่ถูกตัดสิทธิ์ไปแล้ว) → fallback จบสมัยแบบ `collapsed:true` (ข้อ 5)
- รัฐมนตรีที่เสียตำแหน่งกลางสมัย กระทรวงว่างตลอดสมัยนั้น ไม่มี UI จัดสรรใหม่กลางเทอม (เหมือนที่ตัดสินใจไว้รอบก่อน) — `drawEventForRoom()`'s `MINISTRY_EVENT_MAP` lookup จัดการ `holder` เป็น `undefined` ได้อยู่แล้วโดยไม่ error (fallback ownerSlot=PM)
- PM กด "ยุบสภา" ระหว่างที่มี courtcase/noconf ค้างอยู่ยังไม่ resolve — ปล่อยให้ทำได้ (ไม่ block) เพราะเป็นอำนาจฉุกเฉินไม่มีเงื่อนไขตามที่ตกลง ค่า `pendingPmDisqualification`/`queuedEvent` ที่ค้างจะกลายเป็นข้อมูลเก่าไม่มีผลหลังเข้า `term_summary` แล้ว (ถูกเคลียร์ตอน `startNewTerm()`)
- `corruptionIncidents` ไม่รีเซ็ตข้ามรัฐบาล/สมัย (คนละความหมายกับ `decisionHistory`) — พรรคที่เคยมีประวัติปกปิดทุจริตในรัฐบาลชุดก่อน ยังถูกยื่นคำร้องได้แม้เปลี่ยนรัฐบาลไปแล้ว เป็นความตั้งใจ

## Testing Plan (manual, ไม่มี automated test harness ในโปรเจกต์นี้)

1. เล่นจนเกิดอีเวนต์ทุจริตกับ PM เอง (ไม่ใช่แค่พรรคร่วม) อย่างน้อย 1 ครั้ง — ยืนยันว่าเกิดได้จริง
2. เลือก "ปกป้อง ไม่ตั้งกรรมการ" ในอีเวนต์ทุจริต แล้วเช็คว่าฝ่ายค้านเห็นปุ่ม "ยื่นคำร้องต่อ [พรรคนั้น]" ปรากฏขึ้น (ก่อนหน้านั้นต้องไม่เห็น)
3. ยื่นคำร้องต่อรัฐมนตรี → วินิจฉัยผิด → เช็คว่ากระทรวงว่างและสัดส่วนความพอใจพรรคร่วมคำนวณใหม่ถูกต้อง
4. ยื่นคำร้องต่อ PM → วินิจฉัยผิด → กด "ดำเนินการต่อ" → เช็คว่าเข้าสู่ phase coalition/cabinet ใหม่ (ไม่ใช่ term_summary) โดย `turn`/`usedEventIds` คงค่าเดิมไว้ และ `decisionHistory` รีเซ็ตเป็น `[]`
5. ยื่นคำร้องแล้วศาลยกคำร้อง (ทดสอบซ้ำหลายรอบเพราะเป็นความน่าจะเป็น) → เช็คว่าความนิยมฝ่ายค้านผู้ยื่นลดลงทุกภาค
6. กดปุ่ม "ยุบสภา" กลางไตรมาสใดก็ได้ → เช็คว่าเข้า term_summary ทันทีพร้อมข้อความ "PM ประกาศยุบสภา" ไม่ปนกับข้อความอื่น แล้วเริ่มสมัยใหม่ได้ตามปกติ
7. ยื่นทั้งญัตติไม่ไว้วางใจและคำร้องศาลในสมัยเดียวกันคนละครั้ง (พรรคเดียวกัน) → เช็คว่าใช้สิทธิ์ทั้งสองอย่างได้อิสระจากกัน ไม่ทับกัน (แก้บั๊ก merge ในข้อ 6 แล้ว)

## Spec self-review checklist
- [x] ไม่มี placeholder/TBD ค้างอยู่
- [x] ไม่มีข้อขัดแย้งกันระหว่างหัวข้อ (การรีเซ็ต field ใหม่ทั้ง 4 ตัวสอดคล้องกันในตารางข้อ 1 และโค้ดทุกจุด)
- [x] ขอบเขตเหมาะสมสำหรับ implementation plan เดียว (ไม่ผูกกับ Group E)
- [x] ไม่มี requirement ที่ตีความได้สองแบบ
- [x] รวมบั๊กที่ต้องแก้คู่กัน (ข้อ 6) ไว้ในสโคปนี้แล้ว ไม่ปล่อยให้เป็นบั๊กแฝงหลังฟีเจอร์นี้ deploy
