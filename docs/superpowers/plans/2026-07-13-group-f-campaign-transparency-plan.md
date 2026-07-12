# Group F: Campaign Transparency + Numeric Input Implementation Plan

> **For agentic workers:** Executed via **inline execution** in the same session (superpowers:executing-plans), not subagent-driven-development. No automated test suite exists in this project — verification per task is a Node.js syntax-parse check plus manual trace, matching the project's existing convention.

**Goal:** Reveal every party's campaign platform once all humans submit, show the previous term's final stats/seats from term 2 onward, and pair every policy/budget input with a linked numeric field.

**Architecture:** All changes land in `index (3).html`'s `PHASE_TEMPLATES.campaign` function and its immediate helpers — no new room-state fields, no changes to any other phase.

**Tech Stack:** Vanilla JS, no build step.

## Global Constraints

- Reveal trigger is exactly `claimedSlots().every(s=>s.submitted)` — the same condition `maybeRunElection()` already uses.
- Prior-term card renders only when `r.term>1`.
- Policy axis clamp: `-100..100`. Budget clamp: `0..(100-usedElsewhere)`, same rule `adjustDraftBudget()` already enforces.

---

### Task 1: Reveal table + prior-term comparison card

**Files:**
- Modify: `index (3).html` — `PHASE_TEMPLATES.campaign` (~line 625).

**Interfaces:**
- Produces: `campaignRevealTable(r)`, `priorTermCard(r)` — plain functions returning HTML strings (or `''`), called from all three `PHASE_TEMPLATES.campaign` return branches.

- [ ] **Step 1:** Add the two helper functions immediately before `PHASE_TEMPLATES.campaign=function(){`:
```js
function priorTermCard(r){
  if(r.term<=1) return '';
  const seatRows=(r.lastElection?.results||[]).map(x=>`<tr><td>${x.name}</td><td class="mono">${x.seats}</td></tr>`).join('');
  return `<div class="doc">
    <h3>ผลงานสมัยที่แล้ว (สมัยที่ ${r.term-1})</h3>
    <div class="doc-grid">
      <div>
        <p><strong>เศรษฐกิจ:</strong> ${fmt(r.national.economy)}/100</p>
        <p><strong>ความเชื่อมั่นประชาชน:</strong> ${fmt(r.national.trust)}/100</p>
        <p><strong>ความไม่สงบ:</strong> ${fmt(r.national.unrest)}/100</p>
        <p><strong>งบประมาณคงเหลือ:</strong> ฿${fmt(r.treasury)} ล้านบาท</p>
      </div>
      <div>
        <table class="ledger"><thead><tr><th>พรรค</th><th>ที่นั่งสมัยก่อน</th></tr></thead><tbody>${seatRows}</tbody></table>
      </div>
    </div>
  </div>`;
}
function campaignRevealTable(r){
  if(!claimedSlots().every(s=>s.submitted) || claimedSlots().length===0) return '';
  const rows=r.slots.map(s=>`<tr>
    <td>${s.name}</td>
    <td class="mono">${s.policy.economy}</td>
    <td class="mono">${s.policy.reform}</td>
    <td class="mono">${s.policy.security}</td>
    <td class="mono">${REGIONS.map(reg=>`${reg.name.slice(0,2)}:${s.budget[reg.id]||0}`).join(' ')}</td>
  </tr>`).join('');
  return `<div class="doc">
    <h3>นโยบายหาเสียงของทุกพรรค</h3>
    <table class="ledger"><thead><tr><th>พรรค</th><th>ศก.</th><th>ปฏิรูป</th><th>มั่นคง</th><th>งบหาเสียง</th></tr></thead><tbody>${rows}</tbody></table>
  </div>`;
}
```
- [ ] **Step 2:** Insert `${priorTermCard(r)}${campaignRevealTable(r)}` into all three return paths of `PHASE_TEMPLATES.campaign`:
  - No-`mine` (spectator) branch: `return `${topbar()}${breadcrumb('campaign')}<div class="doc">...</div>${priorTermCard(r)}${campaignRevealTable(r)}`;` (append after the existing waiting-box `</div>`, still inside the same template literal, before the closing backtick).
  - `mine.submitted` branch: same pattern — append after its existing waiting-box `</div>`.
  - Do **not** add it to the draft-editing branch (`mine` exists and hasn't submitted) — `campaignRevealTable`'s own guard already returns `''` there since the condition can't be true while `mine` hasn't submitted, and showing `priorTermCard` there is deferred to Task 1 Step 3 below for a clean single insertion point.
- [ ] **Step 3:** In the draft-editing branch's return template (the one starting `${topbar()}${breadcrumb('campaign')}\n  <div class="doc">\n    <h2>แถลงนโยบายพรรค...`), insert `${priorTermCard(r)}` immediately after `${breadcrumb('campaign')}` so a player sees last term's results while still deciding their platform.
- [ ] **Step 4 (verify):**
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
- [ ] **Step 5: Commit**
```bash
git add "index (3).html"
git commit -m "Reveal campaign platforms after all submit and show prior-term stats"
```

---

### Task 2: Numeric input paired with every policy/budget control

**Files:**
- Modify: `index (3).html` — the 3 policy-axis sliders and the region-budget stepper table inside `PHASE_TEMPLATES.campaign`'s draft-editing branch; add one new function `setDraftBudget`.

**Interfaces:**
- Consumes: `ui.draft.policy`, `ui.draft.budget` (existing draft state).
- Produces: `setDraftBudget(regionId, rawValue)`.

- [ ] **Step 1:** Replace the 3 policy-axis inputs with slider+number pairs, wired to stay in sync without a full re-render (matching the existing sliders' no-`render()` pattern):
```html
    <div class="axis"><div class="axis-labels"><span>ประชานิยม / รัฐสวัสดิการ</span><span>เสรีตลาด / ทุนนิยม</span></div>
      <div class="axis-input-row">
        <input type="range" min="-100" max="100" value="${draft.policy.economy}" oninput="ui.draft.policy.economy=Number(this.value); this.nextElementSibling.value=this.value">
        <input type="number" min="-100" max="100" value="${draft.policy.economy}" style="width:64px;" oninput="const v=Math.max(-100,Math.min(100,Number(this.value)||0)); ui.draft.policy.economy=v; this.previousElementSibling.value=v; this.value=v">
      </div>
    </div>
    <div class="axis"><div class="axis-labels"><span>อนุรักษ์นิยม / รักษาสถาบันเดิม</span><span>ปฏิรูปโครงสร้าง</span></div>
      <div class="axis-input-row">
        <input type="range" min="-100" max="100" value="${draft.policy.reform}" oninput="ui.draft.policy.reform=Number(this.value); this.nextElementSibling.value=this.value">
        <input type="number" min="-100" max="100" value="${draft.policy.reform}" style="width:64px;" oninput="const v=Math.max(-100,Math.min(100,Number(this.value)||0)); ui.draft.policy.reform=v; this.previousElementSibling.value=v; this.value=v">
      </div>
    </div>
    <div class="axis"><div class="axis-labels"><span>สายพิราบ / เจรจา</span><span>สายเหยี่ยว / ความมั่นคง</span></div>
      <div class="axis-input-row">
        <input type="range" min="-100" max="100" value="${draft.policy.security}" oninput="ui.draft.policy.security=Number(this.value); this.nextElementSibling.value=this.value">
        <input type="number" min="-100" max="100" value="${draft.policy.security}" style="width:64px;" oninput="const v=Math.max(-100,Math.min(100,Number(this.value)||0)); ui.draft.policy.security=v; this.previousElementSibling.value=v; this.value=v">
      </div>
    </div>
```
- [ ] **Step 2:** Add `setDraftBudget` right before `function submitCampaign(){` (immediately after `adjustDraftBudget`):
```js
function setDraftBudget(regionId,rawValue){
  const used=Object.entries(ui.draft.budget).reduce((s,[k,v])=>s+(k===regionId?0:v),0);
  const clamped=Math.max(0,Math.min(100-used,Math.round(Number(rawValue)||0)));
  ui.draft.budget[regionId]=clamped;
  render();
}
```
- [ ] **Step 3:** Add a number input next to each region's stepper (the `<div class="stepper">` block in the budget table):
```html
        <td><div class="stepper">
          <button onclick="adjustDraftBudget('${reg.id}',-5)">−</button>
          <span class="amt">${draft.budget[reg.id]}</span>
          <button onclick="adjustDraftBudget('${reg.id}',5)">+</button>
          <input type="number" min="0" max="100" value="${draft.budget[reg.id]}" style="width:56px;margin-left:6px;" onchange="setDraftBudget('${reg.id}',this.value)">
        </div></td>
```
- [ ] **Step 4 (verify):** run the Task 1 Step 4 syntax check. Expected: `syntax OK`
- [ ] **Step 5: Commit**
```bash
git add "index (3).html"
git commit -m "Pair policy sliders and budget steppers with numeric inputs"
```

## Self-Review

- **Spec coverage:** §1 reveal → Task 1; §2 prior-term card → Task 1; §3 numeric inputs → Task 2.
- **Placeholder scan:** none found — every step has complete code.
- **Type consistency:** `priorTermCard(r)`/`campaignRevealTable(r)` take the room object exactly like every other `PHASE_TEMPLATES.*` helper in this file; `setDraftBudget` mirrors `adjustDraftBudget`'s existing clamp formula exactly, so both stay consistent if either changes later.
