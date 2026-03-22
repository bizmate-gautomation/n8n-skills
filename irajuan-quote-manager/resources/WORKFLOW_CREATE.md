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

1. `create_boq(name, projectId, leadId, fileUrl)` — store BOQ record in Airtable
2. `progress_update("⏳ מעבד את כתב הכמויות...")` — notify contractor before parsing
3. `parse_boq(document_id)` — parse file, returns flat list: `[{Category, Description, Quantity, Unit}, ...]`
4. Group items by `Category` — each unique Category becomes a room (roomName = Category)
5. `progress_update("⏳ מתאים פריטים לקטלוג...")` — notify contractor before batch matching
6. Collect all `Description` values across all rooms → `get_catalog_candidates(items="desc1|desc2|desc3|...")`
7. Claude matches each item to catalog (same rules as Step 3a)
8. If 3+ rooms: `progress_update("⏳ יוצר חדרים בפרויקט...")` — notify before room loop
9. For each room/Category: `scan_room(projectId, roomName=Category, items=[{name=Description, qty=Quantity, unit=Unit, catalog_id, unit_cost, unit_client_price}], offerType="BOQ")`
10. Show parsed rooms summary:

```
"כתב הכמויות נקלט. נמצאו [N] חדרים:
• [room1] — [X] פריטים
• [room2] — [Y] פריטים
...
סה"כ [total] פריטים. האם הכל נראה תקין?"
```

11. Let contractor review and correct if needed

---

## Step 4: Review (סקירה)

1. `get_project_rooms(projectId)` — fetch all rooms with items
2. Display complete summary — all rooms, all items, quantities, units, prices
3. Let contractor make corrections:
   - **Add new room** → match items with `get_catalog_candidates` → `scan_room` with pricing
   - **Add items to existing room** → `scan_room` returns `room_exists` with current items → merge → `replace_room_items` with full list
   - **Corrections** → describe changes needed and handle accordingly

## Step 5: Generate Quote (יצירת הצעת מחיר)

1. `progress_update("⏳ מכין הצעת מחיר...")` — notify contractor before quote generation
2. `create_quote(projectId)` — creates quote with rooms_snapshot
3. **First: show internal cost summary** to contractor (use Internal Cost Summary template from TEMPLATES.md) — includes עלות, מחיר ללקוח, and רווח
4. Ask contractor to review and approve the costs:
   ```
   "האם העלויות נראות תקינות? אפשר לתקן לפני שנמשיך להצעה ללקוח."
   ```
5. **If contractor approves costs** → skip to step 7
6. **If contractor wants corrections** → run Offer Correction sub-flow (see below), then return to step 1
7. **Only after explicit cost approval** → show full client quote summary (use Quote Summary template from TEMPLATES.md)
8. **If contractor wants corrections to client quote** → run Offer Correction sub-flow with offer_type="client", then `create_quote(projectId)` to regenerate, show updated client quote again
9. **Only after explicit client approval** → done. Confirm quote is ready.

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
6. `create_quote(projectId)` — regenerate the document

