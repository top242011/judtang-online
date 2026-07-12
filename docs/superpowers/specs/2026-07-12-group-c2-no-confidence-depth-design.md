# Group C2: ญัตติไม่ไว้วางใจแบบพรรคร่วมโหวต + อิงประวัติ — Design Spec

**วันที่:** 2026-07-12
**ขอบเขต:** แก้ 2 ฟีเจอร์ในกลไก "ญัตติอภิปรายไม่ไว้วางใจ" ของเกม "จัดตั้ง ออนไลน์" ([index (3).html](../../../index%20(3).html)) — ต่อยอดจาก Group A/B/C1 ที่ merge เข้า `main` แล้ว (commit `0ed5391`)

- **#3**: พรรคร่วมรัฐบาลไม่มีสิทธิ์ตัดสินใจตอนนี้ (มีแค่ PM คนเดียวเลือก) → เปลี่ยนเป็นให้พรรคร่วมทุกพรรคโหวต ถ่วงน้ำหนักตามที่นั่ง
- **#11**: การตอบญัตติไม่มีมิติเรื่องประวัติ → เพิ่มผลกระทบตัวเลขจริงจากประวัติการตัดสินใจของรัฐบาลชุดปัจจุบัน

**ไม่รวมในสโคปนี้:** Group C3 (แย่งจัดตั้งรัฐบาลพร้อมกันหลายพรรค), Group D (ศาลรัฐธรรมนูญ), Group E (วิสัยทัศน์ระยะยาว) — และกลไกนี้จำกัดเฉพาะอีเวนต์ `noconf` เท่านั้น อีเวนต์อื่นทั้งหมดยังให้เจ้าของกระทรวง/PM ตัดสินใจคนเดียวเหมือนเดิม ไม่เปลี่ยน

---

## 1. Data model ใหม่ (2 field ระดับห้อง)

```
decisionHistory: []   // ประวัติการตัดสินใจของรัฐบาลชุดปัจจุบัน แต่ละรายการ:
                       // {eventId, choiceIndex, trustDelta, turn, term}
noConfidenceVotes: {} // map partyId -> choiceIndex (0 หรือ 1) ของญัตติปัจจุบัน
```

**จุดที่ต้องตั้งค่าเริ่มต้น/รีเซ็ตทั้ง 2 field (ตรวจสอบครบทุกจุดที่มีอยู่แล้วในโค้ด):**

| ฟังก์ชัน | เหตุผล | `decisionHistory` | `noConfidenceVotes` |
|---|---|---|---|
| `handleCreateRoom()` | ห้องใหม่ | `[]` | `{}` |
| `resetRoomToLobby()` | หัวห้องรีเซ็ตกลาง game | `[]` | `{}` |
| `confirmCabinet()` | รัฐบาลชุดใหม่เริ่มนับประวัติใหม่ | `[]` | `{}` |
| AI-cabinet automation (ใน `runHostAutomation()`, ทำงานเดียวกับ `confirmCabinet()` แต่กรณี PM เป็น AI) | เดียวกับข้างบน | `[]` | `{}` |
| `nextTurn()` | อีเวนต์ noconf ใหม่ (สุ่มหรือถูกยื่น) ต้องเริ่มโหวตใหม่เสมอ | ไม่แตะ (สะสมต่อเนื่องทั้งสมัย) | `{}` |
| `chooseEvent()` | ญัตติที่เพิ่งแก้ไขจบแล้ว ต้องเคลียร์คะแนนโหวตก่อนอีเวนต์ถัดไป | เพิ่มรายการใหม่เข้าไป (ไม่ล้าง) | `{}` |

`decisionHistory` สะสมตลอดทั้งสมัยของรัฐบาลชุดนั้น (ไม่ล้างทุกไตรมาส) — ล้างเฉพาะตอนรัฐบาลชุดใหม่ก่อตั้ง (`confirmCabinet`) เพราะเป็น "ผลงาน/ชื่อเสียง" ของรัฐบาลชุดนั้นโดยเฉพาะ

---

## 2. #11: ตอบญัตติอิงประวัติ (กระทบตัวเลขจริง)

### Helper functions ใหม่ (วางใกล้ `computeDeficitCrisis` ในกลุ่ม pure helpers)

```javascript
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

- ตัวเลือก 0 ("ชี้แจงข้อกล่าวหาอย่างละเอียดในสภา", base `trust:5`): ยิ่งมีประวัติตัดสินใจที่กระทบความเชื่อมั่นเชิงลบมาก โบนัสความเชื่อมั่นยิ่งลดลง (ชี้แจงได้ไม่น่าเชื่อถือเท่าที่ควร) — ไม่ติดลบ (มี `Math.max(0,...)`)
- ตัวเลือก 1 ("ปัดตกข้อกล่าวหาโดยไม่ให้น้ำหนัก", base `trust:-10`): ยิ่งประวัติแย่ ยิ่งโดนหนักเป็น 2 เท่าต่อรายการ (ปัดตกซ้ำเข้าไปอีกดูหยิ่งยโส)

### แก้ `chooseEvent()`

**เดิม** (บรรทัด 1161-1162):
```javascript
  const choice=ev.choices[choiceIdx];
  const eff=choice.effect;
```

**ใหม่:**
```javascript
  const choice=ev.choices[choiceIdx];
  let eff=choice.effect;
  if(ev.id==='noconf'){
    eff=computeNoConfidenceEffect(eff,choiceIdx,r.decisionHistory);
  }
```

ทุกอย่างที่เหลือใน `chooseEvent()` (การคำนวณ `update.national`, `tags`, filer's gambit ฯลฯ) อ้างอิงตัวแปร `eff` อยู่แล้ว จึงทำงานถูกต้องอัตโนมัติโดยไม่ต้องแก้จุดอื่น

**เพิ่ม** ที่ท้ายฟังก์ชัน (ก่อน `roomRef().update(update);`):
```javascript
  update.decisionHistory=[...(r.decisionHistory||[]),{eventId:ev.id,choiceIndex:choiceIdx,trustDelta:eff.trust||0,turn:r.turn,term:r.term}];
  update.noConfidenceVotes={};
```

### UI: badge แสดงประวัติก่อนโหวต/ตัดสินใจ

ใน `PHASE_TEMPLATES.governing` เพิ่ม badge ใต้คำอธิบายอีเวนต์ เฉพาะตอนเป็นอีเวนต์ `noconf` และยังไม่มีผลลัพธ์ (`!result`):
```javascript
${ev.id==='noconf'&&!result?`<div class="tag far" style="display:inline-block;margin-bottom:10px;">ประวัติการตัดสินใจที่กระทบความเชื่อมั่นเชิงลบ: ${countBadDecisions(r.decisionHistory)} ครั้ง</div>`:''}
```
ผู้เล่นเห็นตัวเลขนี้ก่อนโหวต ไม่ใช่ถูกเซอร์ไพรส์ตอนเห็นผลลัพธ์

---

## 3. #3: พรรคร่วมโหวตตอบญัตติ (ถ่วงน้ำหนักตามที่นั่ง, PM ชี้ขาดเมื่อเสมอ)

### Logic functions ใหม่ (วางใกล้ `fileNoConfidence()`)

```javascript
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

- `castNoConfidenceVote(choiceIdx, slotId)`: `slotId` เป็น optional — ผู้เล่นมนุษย์เรียกโดยไม่ใส่ (ใช้ `mySlot()` ของตัวเอง); AI automation เรียกโดยระบุ `slotId` ของ AI party ที่กำลังโหวตแทน
- `tallyNoConfidenceWinner(r)`: รวมที่นั่งของแต่ละฝั่งตามคะแนนโหวต, เสมอกันพอดี → คืนค่าตามที่ `r.pm` โหวตไว้ (PM ชี้ขาด)
- `resolveNoConfidenceVote()`: เรียกใช้ `chooseEvent()` เดิมโดยตรงด้วยผลโหวต — ไม่ต้องเขียนโค้ดคำนวณผลกระทบซ้ำ

### UI: `noConfidenceVotingPanel(r, mine, ev)` (helper ใหม่ วางใกล้ `gauge()`)

```javascript
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

ใช้ CSS class ที่มีอยู่แล้วทั้งหมด (`tag pending/accepted/declined`, `party-row`, `majority-line`, `event-choice`) — ไม่ต้องเพิ่ม CSS ใหม่ การ map ตัวเลือก 0→`accepted`(เขียว)/1→`declined`(แดง) ใช้สีเชิงสัญลักษณ์เพื่อแยกแยะ 2 ฝั่งเท่านั้น ไม่ได้หมายถึง "ยอมรับ/ปฏิเสธ" ตามความหมายเดิมของ tag นั้น

### แก้ `PHASE_TEMPLATES.governing`

**เดิม** (บรรทัด 778-788):
```javascript
      <div class="doc event-card">
        <h2>${ev.title} ${isMyTurn?'<span class="turn-badge">ตาของคุณ</span>':''}</h2>
        <div class="doc-sub">ไตรมาสที่ ${r.turn} / ${r.turnsPerTerm} &middot; สมัยที่ ${r.term} &middot; ผู้ตัดสินใจ: ${ownerSlot.name}${ownerSlot.claimedBy?'':' (AI)'}${ev._filedBy?' &middot; ยื่นโดย '+(r.slots.find(s=>s.id===ev._filedBy)?.name||''):''}</div>
        <p style="font-size:14px;line-height:1.7;">${ev.desc}</p>
        ${result?`
          <div class="result-box">${result.text}<div class="effect-tags">${result.tags.map(t=>`<span class="efftag ${t.sign}">${t.text}</span>`).join('')}</div></div>
          ${isMyTurn?`<div class="btn-row"><button class="btn btn-stamp" onclick="nextTurn()">ดำเนินการต่อ</button></div>`:`<div class="waiting-box">รอ ${ownerSlot.name} ดำเนินการต่อ...</div>`}
        `:(isMyTurn?`
          ${ev.choices.map((c,i)=>`<button class="event-choice" onclick="chooseEvent(${i})"><span class="clabel">${c.label}</span><span class="cdesc">${c.desc}</span></button>`).join('')}
        `:`<div class="waiting-box">รอ ${ownerSlot.name}${ownerSlot.claimedBy?'':' (AI)'} ตัดสินใจ...</div>`)}
      </div>
```

**ใหม่:**
```javascript
      <div class="doc event-card">
        <h2>${ev.title} ${isMyTurn?'<span class="turn-badge">ตาของคุณ</span>':''}</h2>
        <div class="doc-sub">ไตรมาสที่ ${r.turn} / ${r.turnsPerTerm} &middot; สมัยที่ ${r.term} &middot; ผู้ตัดสินใจ: ${ownerSlot.name}${ownerSlot.claimedBy?'':' (AI)'}${ev._filedBy?' &middot; ยื่นโดย '+(r.slots.find(s=>s.id===ev._filedBy)?.name||''):''}</div>
        <p style="font-size:14px;line-height:1.7;">${ev.desc}</p>
        ${ev.id==='noconf'&&!result?`<div class="tag far" style="display:inline-block;margin-bottom:10px;">ประวัติการตัดสินใจที่กระทบความเชื่อมั่นเชิงลบ: ${countBadDecisions(r.decisionHistory)} ครั้ง</div>`:''}
        ${result?`
          <div class="result-box">${result.text}<div class="effect-tags">${result.tags.map(t=>`<span class="efftag ${t.sign}">${t.text}</span>`).join('')}</div></div>
          ${isMyTurn?`<div class="btn-row"><button class="btn btn-stamp" onclick="nextTurn()">ดำเนินการต่อ</button></div>`:`<div class="waiting-box">รอ ${ownerSlot.name} ดำเนินการต่อ...</div>`}
        `:(ev.id==='noconf'?noConfidenceVotingPanel(r,mine,ev):(isMyTurn?`
          ${ev.choices.map((c,i)=>`<button class="event-choice" onclick="chooseEvent(${i})"><span class="clabel">${c.label}</span><span class="cdesc">${c.desc}</span></button>`).join('')}
        `:`<div class="waiting-box">รอ ${ownerSlot.name}${ownerSlot.claimedBy?'':' (AI)'} ตัดสินใจ...</div>`))}
      </div>
```

หมายเหตุ: `ownerSlot` ยังคงเป็น PM เหมือนเดิมสำหรับอีเวนต์ noconf (ไม่เปลี่ยน `drawEventForRoom()`) — ใช้เป็น "ผู้กดดำเนินการต่อหลังเห็นผลลัพธ์" เท่านั้น ส่วน "ใครเป็นคนกำหนดตัวเลือก" เปลี่ยนจาก PM คนเดียวเป็นผลโหวตของพรรคร่วมทั้งหมด — แยกความรับผิดชอบ 2 อย่างออกจากกันชัดเจน

---

## 4. AI automation (พรรคร่วมที่เป็น AI ต้องโหวตเองอัตโนมัติ)

แก้ branch `if(r.phase==='governing')` ใน `runHostAutomation()` (บรรทัด 1378-1397) เพิ่ม logic ใหม่ก่อนโค้ดเดิม:

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
      /* ...โค้ดเดิมทั้งหมด ไม่เปลี่ยน... */
```

**สิ่งที่ตั้งใจไม่ทำในสโคปนี้:** การแก้ไขนี้เป็น host-only automation เหมือนกับ automation ของ coalition/cabinet ที่มีอยู่แล้ว (ไม่ transaction-guard เหมือนที่ทำกับ `maybeRunElection()` ใน Group A) — ถ้าหัวห้องปิด tab ระหว่างช่วงโหวต การโหวต/สรุปผลจะค้างจนกว่าหัวห้องกลับมา เป็นความเสี่ยงเดียวกับที่ automation ของ coalition/cabinet มีอยู่แล้วในโค้ดปัจจุบัน ไม่ใช่ปัญหาใหม่ที่ spec นี้ต้องแก้ (Group A แก้เฉพาะกรณี election calculation ที่ถูกรายงานเป็นบั๊กเจาะจง #7 เท่านั้น)

---

## Edge cases

- รัฐบาลเสียงข้างมากเดี่ยว (`r.coalitionSlots.length===1`, มีแค่ PM): การโหวตยุบเหลือ PM โหวตคนเดียว ผลลัพธ์เท่ากับ PM ตัดสินใจเองเหมือนพฤติกรรมเดิมก่อนแก้ (ไม่ต้องมี branch พิเศษ — `tallyNoConfidenceWinner` คำนวณถูกต้องโดยธรรมชาติเมื่อมีผู้โหวตแค่คนเดียว)
- ผู้เล่นมนุษย์ในพรรคร่วมไม่โหวต (เงียบหาย/ออกจากหน้าจอ): เกมจะรอค้างจนกว่าจะโหวต — ยอมรับความเสี่ยงเดิม เหมือน campaign phase ที่รอทุกคนส่งนโยบายอยู่แล้ว ไม่ใช่ปัญหาคลาสใหม่
- `r.decisionHistory` ว่างเปล่า (`[]`) ตอนต้นสมัย → `countBadDecisions` คืนค่า `0` → ไม่กระทบตัวเลขใดๆ พฤติกรรมเหมือนก่อนแก้ spec นี้ทุกประการ (backward-compatible กับห้องที่ตั้งค่าเริ่มต้นถูกต้อง)

## Testing Plan (manual, ไม่มี automated test harness ในโปรเจกต์นี้)

1. **#11**: เล่นจนมีประวัติ `decisionHistory` สะสมอย่างน้อย 2-3 รายการที่ `trustDelta<0` (เลือกตัวเลือกที่กระทบ trust ติดลบในอีเวนต์ทั่วไปหลายครั้ง) แล้วให้เกิดอีเวนต์ noconf — เช็คว่า badge แสดงจำนวนครั้งถูกต้อง และผลกระทบ trust ของตัวเลือกทั้ง 2 ถูกปรับตามสูตร (เทียบกับค่า base `5`/`-10` ตามจำนวน `badCount`)
2. **#3 (2 มนุษย์ในพรรคร่วม)**: ตั้งรัฐบาลผสม 2 พรรคมนุษย์ (PM + พรรคร่วม 1) ให้เกิดอีเวนต์ noconf — ทั้งคู่โหวตคนละตัวเลือก เช็คว่าเห็นคะแนนโหวตของกันและกันทันที และผลตัดสินออกมาตามที่นั่งที่มากกว่า
3. **#3 (เสมอกันพอดี)**: จัดที่นั่งพรรคร่วมให้เท่ากันพอดีระหว่าง 2 ฝั่ง (ปรับข้อมูลผ่าน console) — เช็คว่าผลตัดสินตามเสียงโหวตของ PM
4. **#3 (พรรคร่วมเป็น AI)**: ทดสอบด้วยพรรคร่วมที่ไม่มีคนเล่น (AI) — เช็คว่า AI โหวตอัตโนมัติโดยไม่ต้องรอมนุษย์ และเมื่อครบทุกพรรคแล้วผลลัพธ์คำนวณอัตโนมัติ (ผ่าน host automation)
5. เช็คว่าอีเวนต์อื่นที่ไม่ใช่ noconf ยังทำงานแบบ PM/เจ้าของกระทรวงตัดสินใจคนเดียวเหมือนเดิม ไม่ถูกกระทบจากการเปลี่ยนแปลงนี้

## Spec self-review checklist
- [x] ไม่มี placeholder/TBD ค้างอยู่
- [x] ไม่มีข้อขัดแย้งกันระหว่างหัวข้อ (การรีเซ็ต `noConfidenceVotes`/`decisionHistory` สอดคล้องกันทั้งตารางข้อ 1 และโค้ดในข้อ 2/3)
- [x] ขอบเขตเหมาะสมสำหรับ implementation plan เดียว (จำกัดเฉพาะอีเวนต์ noconf ไม่ลามไปอีเวนต์อื่น ไม่ผูกกับ C3/D/E)
- [x] ไม่มี requirement ที่ตีความได้สองแบบ
