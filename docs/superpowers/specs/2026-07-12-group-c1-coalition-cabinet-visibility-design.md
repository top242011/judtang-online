# Group C1: แสดงชื่อผู้เล่น + เปิดหน้าจัดสรร ครม. ให้พรรคร่วมดู — Design Spec

**วันที่:** 2026-07-12
**ขอบเขต:** แก้ 2 ฟีเจอร์ UI-only ในเกม "จัดตั้ง ออนไลน์" ([index (3).html](../../../index%20(3).html)) — ต่อยอดจาก [Group A](2026-07-12-group-a-critical-bugfixes-design.md) และ [Group B](2026-07-12-group-b-lobby-improvements-design.md) ที่ merge เข้า `main` แล้ว (commit `3df77d4`)

**ไม่รวมในสโคปนี้ (แยกไปทำทีหลังเป็น Group C2/C3):**
- Group C2: พรรคร่วมมีสิทธิ์ตัดสินใจในญัตติไม่ไว้วางใจ (#3), การตอบญัตติละเอียดขึ้นตามประวัติการตัดสินใจ (#11)
- Group C3: ระบบเสนอแย่งจัดตั้งรัฐบาลพร้อมกันหลายพรรค (#5 ส่วนที่เหลือ — ส่วน "สละสิทธิ์" ทำไปแล้วใน Group A)

ไม่มีการเปลี่ยนแปลง data model ในสโคปนี้เลย — ทั้งสองข้อเป็นการอ่านข้อมูลที่มีอยู่แล้ว (`playerDisplayName`, `r.cabinet`) มาแสดงผลเพิ่มเติมเท่านั้น

---

## 1. แสดงชื่อผู้เล่นในหน้าเจรจาจัดตั้งรัฐบาลผสม (แก้ #8)

### แถวพรรคที่เชิญ/รอตอบรับ (`PHASE_TEMPLATES.coalition`, ส่วน `others.map(...)`)
เปลี่ยน tag ทั่วไป "คน" ให้เป็นชื่อผู้เล่นจริง:

**เดิม:**
```javascript
<div class="pname">${s.name} <span class="tag ${tag.cls}">${tag.label}</span> ${s.claimedBy?'<span class="tag human">คน</span>':'<span class="tag ai">AI</span>'}</div>
```

**ใหม่:**
```javascript
<div class="pname">${s.name} <span class="tag ${tag.cls}">${tag.label}</span> ${s.claimedBy?`<span class="tag human">${s.playerDisplayName||'ไม่ทราบชื่อ'}</span>`:'<span class="tag ai">AI</span>'}</div>
```

### เส้น doc-sub ของพรรคแกนนำ (PM)
**เดิม:**
```javascript
<div class="doc-sub">พรรคแกนนำ: <strong>${pmSlot.name}</strong>${pmSlot.claimedBy?'':' (AI ดำเนินการอัตโนมัติ)'} มี ${pmSeats} ที่นั่ง &middot; ต้องการรวม ${MAJORITY} ที่นั่ง</div>
```

**ใหม่:**
```javascript
<div class="doc-sub">พรรคแกนนำ: <strong>${pmSlot.name}</strong>${pmSlot.claimedBy?` (ผู้เล่น: ${pmSlot.playerDisplayName||'ไม่ทราบชื่อ'})`:' (AI ดำเนินการอัตโนมัติ)'} มี ${pmSeats} ที่นั่ง &middot; ต้องการรวม ${MAJORITY} ที่นั่ง</div>
```

ไม่มีการเปลี่ยน data model — `playerDisplayName` มีอยู่ในทุก slot ตั้งแต่โค้ดเดิม (ตั้งค่าตอน `claimSlot()`)

---

## 2. เปิดหน้าจัดสรร ครม. ให้พรรคร่วมทุกคนดูแบบ real-time (แก้ #2)

### สถานะปัจจุบัน
`PHASE_TEMPLATES.cabinet` (บรรทัด 688-691) — พรรคร่วมที่ไม่ใช่ PM เห็นแค่:
```javascript
if(!isPM){
    return `${topbar()}${breadcrumb('cabinet')}<div class="doc"><h2>รอการแต่งตั้งคณะรัฐมนตรี</h2>
      <div class="waiting-box">${pmSlot.claimedBy?('พรรค '+pmSlot.name+' กำลังจัดสรรตำแหน่งรัฐมนตรี...'):'AI กำลังจัดสรรตำแหน่งรัฐมนตรี...'}</div></div>`;
}
```
ไม่มีรายละเอียดใดๆ เลยระหว่างรอ

### การเปลี่ยนแปลง
สร้าง read-only view ของหน้าเดียวกับที่ PM เห็น โดยใช้ตัวแปรที่คำนวณไว้แล้วก่อนหน้า if (`parties`, `assignedWeight`, `allAssigned` — ไม่ต้องคำนวณซ้ำ)

**Helper ใหม่ 2 ตัว** (เพื่อไม่ให้ markup ซ้ำกันระหว่าง 2 มุมมอง — หลีกเลี่ยง "verbatim duplication of a logic block"):

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
```

**`PHASE_TEMPLATES.cabinet` ใหม่ทั้งฟังก์ชัน:**

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

**สิ่งที่ตั้งใจไม่เปลี่ยน:** พรรคร่วมยังคง **แก้ไขไม่ได้** — ไม่มี `<select>`, ไม่มีปุ่ม "จัดสรรอัตโนมัติ", ไม่มีปุ่ม "แต่งตั้งคณะรัฐมนตรี" ในมุมมองที่ไม่ใช่ PM สิทธิ์ตัดสินใจยังอยู่ที่ PM คนเดียวเหมือนเดิม — ฟีเจอร์นี้เพิ่มแค่ "การมองเห็น" (visibility) ไม่เพิ่ม "อำนาจ" (authority)

การอัปเดตแบบ real-time ระหว่าง PM กำลังกด `<select>` เปลี่ยนตำแหน่งรัฐมนตรี จะเห็นผลอัตโนมัติผ่าน Firestore `onSnapshot` ที่มีอยู่แล้ว ไม่ต้องเพิ่มโค้ดใดๆ เพื่อ sync

---

## Edge cases

- `pmSlot.playerDisplayName` อาจเป็น `null` ได้ในทางทฤษฎีถ้า claimedBy มีค่าแต่ playerDisplayName ไม่มี (ไม่ควรเกิดขึ้นจริงเพราะ `claimSlot()` ตั้งทั้งคู่พร้อมกันเสมอ) — ใช้ fallback `||'ไม่ทราบชื่อ'` เดียวกับที่ `slotCard()` ใช้อยู่แล้ว เพื่อความสอดคล้อง
- Ministry ที่ยังไม่ถูกจัดสรร (`r.cabinet[m.id]` เป็น `undefined`) ในมุมมอง read-only ต้องแสดง "ยังไม่ได้กำหนด" ไม่ใช่ค่าว่างเปล่าหรือ `undefined` ที่หลุดออกมาทาง UI

## Testing Plan (manual, ไม่มี automated test harness ในโปรเจกต์นี้)

1. สองผู้เล่นเข้าห้อง คนหนึ่งเป็น PM อีกคนเป็นพรรคร่วม (ต้องเจรจาจนได้เสียงข้างมากก่อน) — เช็คว่าหน้าเจรจาจัดตั้งรัฐบาลผสมแสดงชื่อผู้เล่นจริงทั้งที่แถวพรรคร่วมและที่ doc-sub ของพรรคแกนนำ (ทั้งฝั่ง PM และฝั่งพรรคร่วมเห็นชื่อตรงกัน)
2. เข้าสู่ phase cabinet — ฝั่งพรรคร่วม (ไม่ใช่ PM) ต้องเห็นรายการ 12 กระทรวงพร้อมดาว และเห็นชื่อพรรคที่ถูกจัดสรรอัปเดตแบบ real-time ทันทีที่ PM เลือกจาก dropdown ของตัวเอง (ไม่ต้อง refresh หน้า) และไม่มี dropdown/ปุ่มใดๆ ให้กดในฝั่งพรรคร่วม
3. เช็คว่าสรุปสัดส่วนน้ำหนักต่อพรรค (X/TOTAL_WEIGHT) ตรงกันทั้ง 2 ฝั่ง (PM และพรรคร่วม)

## Spec self-review checklist
- [x] ไม่มี placeholder/TBD ค้างอยู่
- [x] ไม่มีข้อขัดแย้งกันระหว่างหัวข้อ
- [x] ขอบเขตเหมาะสมสำหรับ implementation plan เดียว (ไม่ผูกกับ C2/C3)
- [x] ไม่มี requirement ที่ตีความได้สองแบบ
- [x] ไม่มีการเปลี่ยนแปลง data model — ลด risk การกระทบ field ที่ Group A/B เพิ่งเพิ่มเข้ามา
