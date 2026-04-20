# יצירת הצעת מחיר — תהליך מלא

## Step 1: Lead (ליד)

Ask for **שם מלא** (full name) and **טלפון** (phone) in one prompt.

```
"שלום! בוא נתחיל ליצור הצעת מחיר.
מה השם המלא של הלקוח ומספר הטלפון?"
```

1. `search_lead(phone)` — search by phone
2. **Found** → show lead details, confirm with contractor, use existing `leadId`
3. **Not found** → `create_lead(fullName, phone)` immediately
4. Also collect if offered: email, address, workType, source (ערוץ הגעה)
   - `workType` allowed values: שיפוץ דירה, צביעת דירה, התקנת מזגן, שיפוץ אמבטיה, החלפת דלתות, שיפוץ מטבח, התקנת פרקט, שיפוץ כללי, התקנת חלונות
   - `source` allowed values: פייסבוק, המלצה, גוגל, אינסטגרם

## Step 2: Project (פרויקט)

Ask in **one prompt**:

```
"מעולה! עכשיו לגבי הפרויקט:
• סוג פרויקט? (דירה / וילה / בית כנסת / חנות / משרדים / אחר)
• כתובת?
• מספר חדרים?
• גודל במ״ר?"
```

1. Auto-generate name: `"[fullName] — [address]"`
2. `search_project(name)` — check for existing
3. **Found** → confirm with contractor, use existing `projectId`
4. **Not found** → `create_project(name, leadId, type, address, rooms, sizeSQM)`

**At this point, ask the contractor:**
```
"יש לך כתב כמויות (Excel/PDF) או שנזין את הפריטים ידנית?"
```
- If BOQ → jump to [Step 3-BOQ](#step-3-boq-כתב-כמויות)
- If manual → continue to Step 3a

---

## Step 3a: Manual Mode — Global Items (פריטים כלליים)

```
"מה הפריטים הכלליים שחלים על כל הדירה?
לדוגמה: צביעה, לוח חשמל, אינסטלציה כללית..."
```

1. Parse the contractor's description into item names
2. **"יתומחר בהמשך" items** → skip catalog matching, add with `unit_cost: 0`, `unit_client_price: 0`, `unit: "קומפלט"`. Do NOT call `update_catalog` (see CATALOG_RULES.md)
3. If 5+ non-"יתומחר בהמשך" items → `progress_update("⏳ מחפש פריטים בקטלוג...")` before catalog search
4. For all other items: `get_catalog_candidates(items="צביעה|לוח חשמל|...")` — get candidates per item
5. Claude matches each item (see [CATALOG_RULES.md](CATALOG_RULES.md)):
   - High similarity (≥0.8) with clear gap → auto-pick
   - Paint items → select by project room count tier (use `minRooms`/`maxRooms`)
   - SQM items → use room sqm as quantity
   - Ambiguous → ask contractor to choose from candidates
   - No match → ask contractor: "לחפש בגוגל או להזין מחיר ידנית?" → Google: `WebSearch` for item pricing, show links → contractor provides final price / Manual: contractor gives cost + client price → `update_catalog` → get catalog_id
6. Call `scan_room(projectId, roomName="כללי", items=[{name, qty, unit, catalog_id, unit_cost, unit_client_price}], offerType="withoutBOQ")`
7. Show created items for confirmation (use Room Parsed template from TEMPLATES.md)
8. If contractor says none → skip to Step 3b

## Step 3b: Manual Mode — Special Jobs (עבודות מיוחדות)

```
"האם יש עבודות מיוחדות?
ניתן לתמחר לפי ימי עבודה או מחיר קבוע."
```

- **Work-day based**: see Work-Day Pricing in [CATALOG_RULES.md](CATALOG_RULES.md)
- **Fixed price**: contractor provides total price directly

1. Parse the contractor's description into item names
2. **"יתומחר בהמשך" items** → skip catalog, add with 0 costs and unit "קומפלט" (same as Step 3a)
3. For all other items: `get_catalog_candidates(items="...")` → match to catalog
4. Claude picks best match per item (same rules as Step 3a)
5. Call `scan_room(projectId, roomName="עבודות מיוחדות", items=[...], offerType="withoutBOQ")`
6. Show created items for confirmation
7. If none → skip to Step 3c

## Step 3c: Manual Mode — Room-by-Room (חדר אחרי חדר)

For each room:

```
"בוא נעבור חדר חדר. ספר לי על החדר הראשון:
• שם החדר?
• מה הפריטים? (עם כמויות אם ידוע)"
```

1. Parse room description into item names
2. **"יתומחר בהמשך" items** → skip catalog, add with 0 costs and unit "קומפלט" (same as Step 3a)
3. For all other items: `get_catalog_candidates(items="פרקט|שפכטל|...")` → get candidates
4. Claude matches each item to catalog (same rules as Step 3a)
5. `scan_room(projectId, roomName, items=[{name, qty, unit, catalog_id, unit_cost, unit_client_price}], offerType="withoutBOQ")` — creates room with priced items
6. Show created result using Room Parsed template
7. Let contractor confirm or correct
8. Ask: "יש חדר נוסף?"
9. Continue until contractor says "סיימתי" / "זהו" / "אין עוד" / "done"

---

## Step 3-BOQ: כתב כמויות

When contractor provides a BOQ file (Excel/PDF):

1. Agent receives: drive file ID, file name, file URL
2. `create_boq_record(boq_name=file_name, boq_projectId=projectId, boq_leadId=leadId, boq_fileUrl=fileUrl)` — store BOQ record in Airtable
3. `progress_update("⏳ מעבד את כתב הכמויות...")` — notify contractor before processing
4. `parse_boq(boq_document_id=drive_file_id)` — parses the BOQ file and stores in DB → returns `{id}` (postgres record ID)
5. `create_quta_offer(quta_id=id, project_id=projectId)` — backend does catalog matching, auto-saves matched items, returns only items needing agent handling:
   - Items with `Unit: "קומפלט"` → komplet items needing classification
   - Items with `reason: "no match"` (and Unit ≠ "קומפלט") → unmatched non-komplet items needing pricing
   - Each item has: `{ID, _excel_row, Description, Category, Unit, Quantity, reason}`
   - `summary: {total, matched, unmatched}`
   - **Note**: `_excel_row` is the unique identifier. `ID` values can be duplicated across items. Matched items are already saved by the backend and are NOT returned.

6. **Process komplet items** — filter items where `Unit: "קומפלט"`, sub-classify into 3 groups, then present in one message:

   **Step 6a: Classify each komplet item**

   For each komplet item, analyze `Notes`, `Description`, and `Category` to assign to a group. Apply in order — first match wins. **When uncertain, default to Group C** (show to contractor).

   **Sanity check**: Group A count + Group B count + Group C count must equal total komplet items. If it doesn't, re-classify before presenting.

   **Group A — Auto "יתומחר בהמשך"** (can't be priced now):
   Identify items whose Notes or Description indicate pricing is not possible yet. Use judgment — look for signals such as:
   - Explicit deferral: "יתומחר בנפרד", "יתומחר בהמשך", "יתומחר לאחר..."
   - Can't price: "לא ניתן לתמחר", "לא יכולים לתמחר"
   - Missing info: "עוד לא הוחלט", "אין כמות", "אין עוד כמות", "לא ידוע"
   - Dependencies: "אחרי סיום הריסות", "אחרי בחירת...", "תלוי ב..."
   - Explicit exclusion: "לא קומפלט", "לא כלול"
   - Any other context that means "we don't have enough information to price this"

   Do NOT pattern-match blindly — use contextual understanding. E.g., "לא ניתן לתמחר בצורה חלקית, תמחור מלא יהיה אחרי סיום הריסות" → Group A (overall meaning: can't price now).

   Treatment: `Quantity: 1`, `unit_cost: 0`, `unit_client_price: 0`, `_isCompleted: false`, `Status: "Pending Quote"`, unit stays "קומפלט". No catalog lookup, no contractor question.

   **Group B — Catalog-resolvable** (standard work, just needs quantity):
   Identify items whose Description describes **common, standard** construction work that a typical renovation contractor's catalog would contain — but is marked "קומפלט" only because the BOQ didn't specify an exact quantity. Signals:
   - Description mentions a recognizable **standard** catalog work type (חיפוי, ריצוף, פרקט, צביעה, גבס, שפכטל, דלתות פנים, ברזים, נקודות חשמל, etc.)
   - The work is inherently measurable (per מ"ר, מטר, or יחידה)
   - Notes do NOT indicate the item can't be priced (was not classified Group A)

   **NOT Group B** — these are specialty/custom work, send to Group C:
   - Custom glass (מחיצות זכוכית, דלתות זכוכית, חלונות מיוחדים)
   - Custom carpentry/millwork (נגרות אומן, דלפקים, ארונות מיוחדים)
   - Signage & branding (שילוטים, באנרים, גרפיטי)
   - Fire/safety systems (ספרינקלרים, גלאי עשן, הידרנט, כריזה)
   - HVAC/mechanical systems (מערכות מיזוג, בקרת כניסה)
   - Multi-trade composite items (combining several different trades in one item)

   Examples:
   - "חיפוי קירות במטבחון בקרמיקה" → tiling, priced per מ"ר → ask for מ"ר
   - "התקנת פרקט בסלון" → flooring, per מ"ר → ask for מ"ר
   - "החלפת 3 דלתות פנים" → doors, per יחידה → quantity=3 already in description
   - "הארכת נקודות חשמל" → electrical points, per יחידה → ask how many
   - "צביעת דלת מילוט קיימת+משקוף" → painting, per יחידה → quantity=1 (one door+frame)

   If quantity is embedded in Description (e.g., "3 דלתות"), extract it — no need to ask.

   Treatment: ask contractor for missing quantity → `get_catalog_candidates` → match using CATALOG_RULES.md rules → enrich with catalog pricing. On successful match: `_isCompleted: true`, `Status: "Priced"`. **Unit changes** from "קומפלט" to catalog unit (e.g., "מ"ר"), **Quantity changes** from 1 to actual. If catalog match fails → fall back to Group C treatment.

   **Group C — True manual pricing** (everything else):
   All remaining komplet items.
   Treatment: `Quantity: 1`, unit: "קומפלט". Ask contractor for `unit_cost` and `unit_client_price`. Priced → `_isCompleted: true`, `Status: "Priced"`. Contractor says "יתומחר בהמשך" → Group A treatment.

   **Step 6b: Present to contractor in one structured message**

   Use the **BOQ Komplet Items Pricing Request** template (TEMPLATES.md). One message with all groups:
   1. Group B items — asking for quantities/measurements
   2. Group C items — asking for unit_cost + unit_client_price
   3. Group A items — FYI, no action needed
   If a group has 0 items, omit its section entirely.
   If ALL items are Group A (no contractor input needed), show the Group A FYI section and proceed directly to step 7 — skip step 6c.

   **Step 6c: Process contractor's response**

   - Group B: take quantities → `get_catalog_candidates` (batch, pipe-separated). If 5+ items, `progress_update("⏳ מתאים פריטים לקטלוג...")` first. Match per CATALOG_RULES.md. Ambiguous matches → ask contractor to choose (may require follow-up questions). Failed matches → ask contractor for manual pricing (Group C fallback).
   - Group C: apply contractor-provided prices. Items marked "יתומחר בהמשך" by contractor → Group A treatment.
   - Group A: already handled, no processing needed.

7. **Process unmatched non-komplet items** — filter items where `reason: "no match"` and Unit ≠ "קומפלט":
   - These items already have Unit and Quantity from the BOQ
   - Ask contractor for pricing: "לחפש בגוגל או להזין מחיר ידנית?"
   - Google → `WebSearch` for item pricing, show links → contractor provides final price
   - Manual → contractor gives cost + client price
   - After getting prices → `update_catalog` → get catalog_id
   - Enrich each item with: `catalog_id`, `unit_cost`, `unit_client_price`
   - For each priced item: set `_isCompleted: true`, `Status: "Priced"`

8. Build the items array for `create_or_update_boq` — only items the agent handled (matched items already saved by backend):
   - Group A komplet items: `unit_cost: 0`, `unit_client_price: 0`, `Quantity: 1`, `_isCompleted: false`, `Status: "Pending Quote"`
   - Group B komplet items (catalog-matched): `catalog_id`, `unit_cost`, `unit_client_price`, actual `Quantity`, actual `Unit` (from catalog — replaces "קומפלט"), `_isCompleted: true`, `Status: "Priced"`
   - Group B komplet items (fallback to C): same as Group C
   - Group C komplet items: `unit_cost`, `unit_client_price`, `Quantity: 1`, `Unit: "קומפלט"`, `_isCompleted: true`, `Status: "Priced"`
   - Unmatched non-komplet items: `catalog_id`, `unit_cost`, `unit_client_price`, `Quantity` (from BOQ), `_isCompleted: true`, `Status: "Priced"`

9. Show complete summary of all agent-handled items with pricing to contractor for confirmation

→ Continue to Step 5-BOQ

---

## Step 4: Review (סקירה) — Manual Mode Only

> **Note:** Step 4 applies to manual mode only. BOQ mode skips directly to Step 5-BOQ.

1. `get_project_rooms(projectId)` — fetch all rooms with items
2. Display complete summary — all rooms, all items, quantities, units, prices
3. Let contractor make corrections:
   - **Add new room** → match items with `get_catalog_candidates` → `scan_room` with pricing
   - **Add items to existing room** → `scan_room` returns `room_exists` with current items → merge → `replace_room_items` with full list
   - **Corrections** → describe changes needed and handle accordingly

## Step 5: Generate Quote — Manual Mode (יצירת הצעת מחיר — ללא כתב כמויות)

1. `progress_update("⏳ מכין הצעת מחיר...")` — notify contractor before quote generation
2. `create_quote(projectId, quote_type="offer")` — generate cost quote
3. Show internal cost summary to contractor (use Internal Cost Summary template from TEMPLATES.md) — includes עלות, מחיר ללקוח, and רווח
4. Show the cost quote `driveLink` immediately (use **Cost quote created** template from TEMPLATES.md)
5. Ask contractor to review and approve the costs:
   ```
   "האם העלויות נראות תקינות? אפשר לתקן לפני שנמשיך להצעה ללקוח."
   ```
6. **If contractor approves costs** → skip to step 8
7. **If contractor wants corrections** → run Offer Correction sub-flow (see below), then return to step 1
8. **Only after explicit cost approval** → `create_quote(projectId, quote_type="client")` — generate client quote
9. Show client quote summary (use Quote Summary template from TEMPLATES.md) and show the client quote `driveLink` immediately (use **Client quote created** template from TEMPLATES.md)
10. **If contractor wants corrections to client quote** → run Offer Correction sub-flow with offer_type="client", then `create_quote(projectId, quote_type="client")` to regenerate, show updated client quote + `driveLink` again
11. **Only after explicit client approval** → done.

## Step 5-BOQ: Generate Quote — BOQ Mode (יצירת הצעת מחיר — כתב כמויות)

1. `create_or_update_boq(project_id, offer_type="cost", updates_or_create=[...agent-handled items with pricing...])` — save komplet + unmatched items for cost offer (matched items already saved by backend)
2. `progress_update("⏳ מכין הצעת מחיר...")`
3. `create_boq_quote(project_id, offer_type="cost", document_id=drive_file_id)` — generate cost quote → returns drive link
4. Show internal cost summary to contractor (Internal Cost Summary template)
5. Show the cost quote drive link (BOQ Cost quote created template)
6. Ask contractor to review and approve costs
7. **If contractor wants corrections** → Offer Correction sub-flow (use `create_boq_quote` to regenerate instead of `create_quote`) → return to step 1
8. **After explicit cost approval** → `create_or_update_boq(project_id, offer_type="client", updates_or_create=[...same items...])` — save items for client offer
9. `create_boq_quote(project_id, offer_type="client", document_id=drive_file_id)` — generate client quote → returns drive link
10. Show client quote + drive link (BOQ Client quote created template)
11. Same correction/approval cycle for client quote
12. Done after explicit client approval

### Offer Correction (תיקון הצעה)

When contractor identifies rows to correct after reviewing either offer:

1. `progress_update("⏳ מעדכן הצעת מחיר...")` — once before the correction cycle
2. Identify rows + what to change. Ask contractor if not clear
3. Determine offer_type per Rule 19 — quantity changes → auto-update both; price changes → ask or infer
4. For each offer_type:
   a. `get_offer_json(project_id, offer_type, item_raw="row1|row2")` — fetch current state
   b. Show current values using Offer Row Details template; confirm what will change
   c. `update_offer_json({project_id, offer_type, updates: [{rowNum, quantity?, unit_cost?, total_cost?}]})` — patch only changed fields. Always include computed `total_cost` when changing quantity or unit_cost
   d. `get_offer_json(project_id, offer_type, item_raw="row1|row2")` — verify changes applied; show updated values. If mismatch → report to contractor and retry
5. `progress_update("⏳ מכין הצעת מחיר...")` — before regeneration
6. **Regenerate** — depends on mode:
   - Manual mode: `create_quote(projectId, quote_type)` — regenerate the relevant document
   - BOQ mode: `create_boq_quote(project_id, offer_type, document_id)` — regenerate the relevant document
7. Show updated `driveLink`

