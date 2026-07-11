# Group A: บั๊กด่วนที่ทำให้เกมค้าง/เพี้ยน — Design Spec

**วันที่:** 2026-07-12
**ขอบเขต:** แก้ 4 บั๊ก/ฟีเจอร์เร่งด่วนในเกม "จัดตั้ง ออนไลน์" ([index (3).html](../../../index%20(3).html)) โดยไม่แตะสถาปัตยกรรมหลัก (ยังเป็น single-file HTML + Firestore เหมือนเดิม)

**ไม่รวมในสโคปนี้ (แยกไปทำทีหลัง):**
- กลุ่ม B (ปรับปรุง Lobby: เพิ่ม/ลดผู้เล่น, ปุ่ม Ready)
- กลุ่ม C (ความลึกของการเจรจา/บริหาร: แสดงชื่อผู้เล่นในหน้าเจรจา, ระบบเสนอแย่งจัดตั้งรัฐบาลพร้อมกันหลายพรรคแบบเต็มรูปแบบ, ญัตติไม่ไว้วางใจให้พรรคร่วมโหวต, การตอบญัตติแบบอิงประวัติ)
- กลุ่ม D (ศาลรัฐธรรมนูญ)
- กลุ่ม E (วิสัยทัศน์ระยะยาว: Democracy4/Lawgiver2/Tropico)

---

## 1. เปลี่ยนจำนวนไตรมาสต่อสมัยเป็น 16 (แก้ #9)

**การเปลี่ยนแปลง:** ในฟังก์ชัน `handleCreateRoom()` เปลี่ยนค่าเริ่มต้นของ `turnsPerTerm` จาก `8` เป็น `16`

**เหตุผลที่ไม่ต้องมี migration:** `turnsPerTerm` เก็บเป็น field ต่อห้อง (`roomData.turnsPerTerm`) ไม่ใช่ global constant ที่โค้ดอ้างอิงตรงๆ ห้องเก่าที่เล่นค้างอยู่แล้วจะยังใช้ค่าที่ถูกบันทึกไว้ตอนสร้างห้อง (8) ห้องใหม่ที่สร้างหลังแก้โค้ดจะได้ 16 ทันที ไม่มีผลกระทบข้ามห้อง

**Risk:** ไม่มี — เป็นการเปลี่ยนค่าคงที่เพียงจุดเดียว

---

## 2. วิกฤตการคลังสะสม (แก้ #12)

### Data model
เพิ่ม field ใหม่ในเอกสารห้อง:
```
deficitStreak: number  // จำนวนไตรมาสติดต่อกันที่ treasury < 0, เริ่มที่ 0
```

### Logic
ใน `nextTurn()` ก่อนคำนวณ/วาดอีเวนต์ของไตรมาสถัดไป ให้เช็ค `r.treasury`:

- **ถ้า `treasury < 0`:**
  - `newDeficitStreak = deficitStreak + 1`
  - `trustPenalty = clamp(3 + newDeficitStreak * 2, 0, 20)` (ลบออกจาก trust)
  - `unrestPenalty = clamp(2 + newDeficitStreak * 1.5, 0, 15)` (บวกเข้า unrest)
  - อัปเดต `national.trust` และ `national.unrest` ด้วยค่าที่ clamp ไว้ในช่วง 0-100 ตามเดิม (ใช้ฟังก์ชัน `clamp()` ที่มีอยู่แล้ว)
- **ถ้า `treasury >= 0`:** `deficitStreak = 0`

**เหตุใดไม่ต้องเพิ่มเงื่อนไข "แพ้"/"จบเกม" ใหม่:** กลไกที่มีอยู่แล้วในเกม (พรรคร่วมรัฐบาลถอนตัวเมื่อ `satisfaction < 10-25%`, รัฐบาลล่มเมื่อ `trust < 25` หรือ `unrest > 70` ใน `chooseEvent()`) จะทำงานเองตามธรรมชาติเมื่อค่าพวกนี้ถูกกระทบจากวิกฤตการคลังสะสม ไม่ต้องเขียนเงื่อนไขจบเกมซ้ำซ้อน

### UI
- ใน `PHASE_TEMPLATES.governing`: แสดง badge เตือน เช่น `⚠️ วิกฤตการคลัง: สะสม N ไตรมาสติดต่อกัน` เมื่อ `r.deficitStreak > 0` ในพาเนล "สถานการณ์ประเทศ"
- ใน `topbar()`: เปลี่ยนสี class ของค่า `฿${fmt(r.treasury)}ล.` เป็นสีแดง (`--bad`) เมื่อ `r.treasury < 0`

### Edge cases
- `deficitStreak` ต้องรีเซ็ตกลับ 0 ตอน `startNewTerm()` และตอน reset ห้อง (ดูข้อ 3) เพื่อไม่ให้ค้างข้ามสมัย/ข้ามรอบเกม

---

## 3. ปุ่ม Reset ห้อง (หัวห้องเท่านั้น) + แก้ต้นตอบั๊กค้างที่ "6/6" (แก้ #7)

### 3.1 ปุ่ม Reset

**UI:** เพิ่มปุ่ม `รีเซ็ตเกม` ใน `topbar()` ถัดจากปุ่ม "ออกจากห้อง" **แสดงเฉพาะเมื่อ `isHost()` เป็นจริง**

**Flow:**
1. กดปุ่ม → เรียก `confirm('รีเซ็ตเกมกลับไปหน้าเลือกพรรค? ความคืบหน้าของรอบนี้จะหายทั้งหมด')`
2. ถ้ายืนยัน → เขียนทับเอกสารห้องด้วยค่าเริ่มต้นแบบเดียวกับตอนสร้างห้อง (`phase:'lobby'`, `term:1`, `treasury:200`, `national:{economy:50,trust:50,unrest:20}`, `deficitStreak:0`, `turn:1`, `turnsPerTerm:16`, `usedEventIds:[]`, `currentEvent:null`, `currentEventOwnerSlot:null`, `currentEffectResult:null`, `lastElection:null`, `pm:null`, `coalitionSlots:[]`, `invitations:{}`, `cabinet:{}`, `coalitionParties:[]`, `collapsed:false`, `electionLock:false`, `pendingCollapse:false`, `oppositionActions:{}`, `queuedEvent:null`, `concededPm:[]`)
3. **field `slots` ไม่ถูกล้างทั้งหมด** — คงค่า `claimedBy` / `playerDisplayName` ของผู้เล่นแต่ละคนไว้ (ไม่ต้องออกจากห้องแล้วขอรหัสใหม่) แต่รีเซ็ต `policy` กลับเป็นค่า template เดิม, `submitted:false`, `budget` กลับเป็น 0 ทุกภาค

**เหตุผลที่เลือก "กลับ Lobby เสมอ":** เป็น fallback เดียวที่ครอบคลุมทุกสถานการณ์ค้าง ไม่ต้องแยก logic ตาม phase ที่ค้าง ง่ายสุดสำหรับหัวห้องที่ไม่เก่งเทคนิค

### 3.2 แก้ต้นตอบั๊กค้างที่ 6/6

**ปัญหาปัจจุบัน:** `runHostAutomation()` มีเงื่อนไข `if(!isHost()||automationBusy||!ui.room) return;` ที่ด่านหน้าสุด ทำให้ฟังก์ชัน `maybeRunElection()` (ตัวคำนวณผลเลือกตั้งเมื่อทุกคนส่งนโยบายครบ) ถูกรันได้จาก **browser ของหัวห้องเท่านั้น** ถ้า tab ของหัวห้องมีปัญหาจังหวะ (ถูก browser throttle ตอนอยู่ background, หรือ listener ยังไม่ทันประมวลผล) ผู้เล่นทุกคนจะติดค้างที่หน้า "รอผู้เล่นคนอื่นยืนยันนโยบาย (N/N)" โดยไม่มีใคร trigger คำนวณผลแทนได้

**แก้ไข:** แยก `maybeRunElection()` ออกจาก gate ของ `isHost()` — ให้ทุก client ที่ subscribe อยู่ (ใน callback ของ `subscribeRoom`) ลองเรียก `maybeRunElection()` ได้เสมอเมื่อ phase เป็น `campaign` โดยไม่ต้องเช็คว่าตัวเองเป็นหัวห้องหรือไม่ ฟังก์ชันนี้ปลอดภัยอยู่แล้วเพราะใช้ Firestore `runTransaction` ที่เช็ค `electionLock` ซ้ำอีกครั้งข้างในก่อนเขียน (ถ้าหลาย client trigger พร้อมกัน จะมีแค่ transaction แรกที่ชนะ ที่เหลือ no-op) ส่วน automation อื่น (coalition/cabinet/governing ของ AI) ยังคงจำกัดเฉพาะหัวห้องเหมือนเดิม (ไม่อยู่ในสโคปนี้)

**การเปลี่ยนโค้ด:** ย้ายเงื่อนไขเรียก `maybeRunElection()` ออกมานอก `runHostAutomation()` ให้ไปเรียกตรงจาก callback ของ `subscribeRoom()` (ทุก client เรียกได้), ส่วน `runHostAutomation()` ที่เหลือ (coalition/cabinet/governing automation) ยังคง gate ด้วย `isHost()` เหมือนเดิม

---

## 4. สละสิทธิ์แกนนำจัดตั้งรัฐบาล — ฉบับขั้นต่ำ (แก้ #6)

### Data model
เพิ่ม field ใหม่ในเอกสารห้อง:
```
concededPm: string[]  // slot id ของพรรคที่เคยได้ลองเป็นแกนนำจัดตั้งรัฐบาลรอบนี้แล้วแต่สละสิทธิ์ไป
```

### Logic — ปุ่มสละสิทธิ์ (ผู้เล่นที่เป็น PM)
ใน `PHASE_TEMPLATES.coalition` เพิ่มปุ่ม `สละสิทธิ์เป็นแกนนำจัดตั้งรัฐบาล` ที่ `isPM()` เห็นได้ตลอดหน้านี้ (ก่อนกด "ลงนามจัดตั้งรัฐบาล") เรียกฟังก์ชันใหม่ `concedePmCandidacy()`:

1. เพิ่ม PM ปัจจุบันเข้า `concededPm`
2. หาผู้สมัคร PM คนถัดไป: เรียง `lastElection.results` จากที่นั่งมากไปน้อย เลือกพรรคแรกที่ `seats > 0` และ **ไม่อยู่ใน `concededPm`**
3. **ถ้าพบผู้สมัครใหม่:** อัปเดต `pm = <slotId ใหม่>`, เคลียร์ `invitations:{}`, `coalitionSlots:[]`, `cabinet:{}` เริ่มเจรจาใหม่ (ยัง phase `coalition` เหมือนเดิม)
4. **ถ้าไม่พบ** (ทุกพรรคที่มีที่นั่ง `>0` อยู่ใน `concededPm` หมดแล้ว): เรียกฟังก์ชัน fallback `restartElectionSameTerm()` — กลับไป phase `campaign` (ไม่เพิ่ม `term`), รีเซ็ต `slots[].submitted=false`, `budget` กลับ 0 ทุกภาค, `electionLock:false`, `concededPm:[]`, `lastElection` คงไว้จนกว่าจะมีผลใหม่

### Logic — AI ที่เป็น PM (ป้องกันไม่ให้ automation ค้างซ้ำ)
ใน host automation ของ phase `coalition` (ส่วนที่จัดการเมื่อ `pm` เป็น AI): หลังจากเชิญพรรคที่เป็นไปได้ทั้งหมดแล้ว (ตาม logic เดิมที่ไล่เชิญตามลำดับที่นั่งจนกว่าจะครบ/หมดพรรค) ถ้ารวมที่นั่งแล้ว **ยังไม่ถึง `MAJORITY`** ให้เรียก logic เดียวกับ `concedePmCandidacy()` ทันที (ไม่ใช้ปุ่ม แต่เรียกจาก automation) แทนที่จะจบ automation ไว้เฉยๆ อย่างที่เป็นอยู่ตอนนี้ — **นี่คือจุดที่เชื่อว่าเป็นสาเหตุจริงของบั๊ก "ดีลตั้งรัฐบาลล่มแล้วเกมค้าง"** เพราะ AI PM ไม่มีทางออกเมื่อเชิญครบแล้วยังไม่ถึงกึ่งหนึ่ง

### Edge cases
- `concededPm` ต้องรีเซ็ตเป็น `[]` ตอนเข้าสู่ phase `coalition` ใหม่ทุกครั้ง (ทั้งจากการเลือกตั้งใหม่ปกติ และจาก reset ห้องตามข้อ 3)
- ปุ่มสละสิทธิ์ต้องซ่อน/ปิดถ้า `coalitionSlots.length > 0` แล้ว (เผื่อกรณี race ที่กดพร้อมกับตอนกำลังจะ confirm — ป้องกัน edge case ไม่ใช่จุดโฟกัสหลักของสโคปนี้)
- ฟีเจอร์นี้เก็บ `concededPm` และรูปแบบ "เลื่อนสิทธิ์ PM ทีละคน" ไว้เป็นโครงสำหรับต่อยอดเป็นระบบ "เสนอแย่งจัดตั้งรัฐบาลพร้อมกันหลายพรรค" แบบเต็มรูปแบบในกลุ่ม C ภายหลัง โดยไม่ต้องรื้อ field ที่เพิ่มไว้ตรงนี้

---

## Testing Plan (manual, เพราะไม่มี automated test harness ในโปรเจกต์นี้)

1. **#9:** สร้างห้องใหม่ เล่นจนถึงหน้า governing เช็คว่า `ไตรมาสที่ X / 16` แสดงถูกต้อง
2. **#12:** ปั้นสถานการณ์ให้ treasury ติดลบ (เลือกอีเวนต์ที่หัก treasury ต่อเนื่องหลายไตรมาส) เช็คว่า trust/unrest ขยับตามสูตร และ badge เตือนขึ้นเมื่อ `deficitStreak > 0`
3. **#7:** จำลอง 6 ผู้เล่นส่งนโยบายครบ เช็คว่าเข้าสู่ election_result ได้แม้เปิดจาก client ที่ไม่ใช่หัวห้อง (ทดสอบโดยปิด/เปิด tab หัวห้องระหว่างส่งคนสุดท้าย); ทดสอบปุ่ม Reset จากทุก phase (lobby/campaign/coalition/cabinet/governing/term_summary) ว่ากลับ lobby ได้ปกติและผู้เล่นไม่หลุดห้อง
4. **#6:** จำลองผลเลือกตั้งที่ไม่มีพรรคไหนได้เสียงข้างมาก และ policy ของทุกพรรคห่างกันมาก (ให้ทุกคำเชิญโดนปฏิเสธ) เช็คว่า AI PM สละสิทธิ์ต่อไปเรื่อยๆ จนสุดท้าย fallback กลับไป campaign ได้โดยไม่ค้าง; ทดสอบกดปุ่มสละสิทธิ์เองในฐานะผู้เล่นด้วย

## Spec self-review checklist
- [x] ไม่มี placeholder/TBD ค้างอยู่
- [x] ไม่มีข้อขัดแย้งกันระหว่างหัวข้อ (การรีเซ็ต `deficitStreak`/`concededPm` ถูกอ้างถึงสอดคล้องกันทั้งในข้อ 2, 3, 4)
- [x] ขอบเขตเหมาะสมสำหรับ implementation plan เดียว (ไม่ผูกกับกลุ่ม B/C/D/E)
- [x] ไม่มี requirement ที่ตีความได้สองแบบ
