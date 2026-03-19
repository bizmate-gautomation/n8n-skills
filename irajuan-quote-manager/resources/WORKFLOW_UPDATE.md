# עדכון הצעת מחיר — תהליך

> **Always start with phone number (Step 1).** Never skip to project search — you need the lead first.

## Step 1: Find Lead (מציאת ליד)

"מה מספר הטלפון של הלקוח?"

1. `search_lead(phone)` — find the lead
2. **Not found** → suggest creating new quote instead (→ WORKFLOW_CREATE.md)
3. Lead found → extract `פרוייקט` array (list of project IDs) from response

## Step 2: Find Quote (מציאת הצעה)

1. For each project_id in lead's `פרוייקט` → `search_quote(search_projectId=project_id)`
2. Collect all quotes across all projects
3. **No quotes found** → tell contractor no quotes exist for this lead, suggest creating new quote
4. **Exactly 1 quote** → use it directly, show quote name for confirmation, extract projectId from quote's `פרוייקט` field
5. **Multiple quotes** → show numbered list (use Quote Selection List template) → contractor picks by number → extract projectId from selected quote's `פרוייקט` field

## Step 3: Show Current State (מצב נוכחי)

1. `get_offer_json(project_id, "offer", item_raw="all")` — fetch ALL items from the cost offer
2. `get_offer_json(project_id, "client", item_raw="all")` — fetch ALL items from the client offer
3. Display both offers in rowNum order (use Current Quote State template)
4. Ask contractor what they want to change

## Step 4: What to Change (מה לשנות)

Ask contractor what changes are needed. Supported operations:

### Add new room
- Parse items into item names
- `get_catalog_candidates(items="פריט1|פריט2|...")` → get candidates
- Claude matches each item to catalog (see CATALOG_RULES.md)
- Unmatched → ask contractor for pricing → `update_catalog` → get catalog_id
- `scan_room(projectId, roomName, items=[{name, qty, unit, catalog_id, unit_cost, unit_client_price}], offerType)` — creates room with priced items
- Show created room items for confirmation

### Add items to existing room
1. `scan_room(projectId, roomName, newItems, offerType)` → returns `{status: "room_exists", items: [...]}`
2. Match new items to catalog via `get_catalog_candidates` if not already matched
3. Combine existing items (from response) with new matched items into one array
4. `replace_room_items(projectId, roomName, fullItemsList, offerType)` → replaces room with complete list
5. Show updated room for confirmation

### Replace from BOQ
- Contractor uploads revised BOQ file
- `progress_update("⏳ מעבד את כתב הכמויות...")` — notify contractor before parsing
- `parse_boq(document_id)` — returns flat item list
- `progress_update("⏳ מתאים פריטים לקטלוג...")` — notify contractor before batch matching
- Group by Category → rooms, match via `get_catalog_candidates`, then `scan_room` per room
- Show updated rooms summary

### Remove items (v2)
- Not yet supported. Flag to contractor:
```
"מחיקת פריטים בודדים עדיין לא נתמכת. ניתן ליצור הצעה חדשה מאפס או להעלות כתב כמויות מעודכן."
```

### Edit offer rows (עריכת שורות בהצעה)

When contractor wants to change prices or quantities on existing offer items:

1. `progress_update("⏳ מעדכן הצעת מחיר...")` — once before the correction cycle
2. Identify rows. Contractor may specify by name or row number
   - By name → `get_offer_json(project_id, offer_type)` with item_raw = all to find row numbers first (rare)
   - By row number → proceed directly (90% of cases)
3. Determine offer_type: cost or client (Rule 19)
4. For each offer_type:
   a. `get_offer_json(project_id, offer_type, item_raw="row1|row2")` — fetch current state
   b. Show current values (use Offer Row Details template); confirm changes with contractor
   c. `update_offer_json({project_id, offer_type, updates: [{rowNum, quantity?, unit_cost?, total_cost?}]})` — patch only changed fields. Always include computed `total_cost` when changing quantity or unit_cost
   d. `get_offer_json(project_id, offer_type, item_raw="row1|row2")` — verify changes applied; show updated values. If mismatch → report to contractor and retry
5. Continue to Step 5 (Generate Updated Quote)

## Step 5: Generate Updated Quote (הצעה מעודכנת)

1. `progress_update("⏳ מכין הצעת מחיר...")` — notify contractor before quote generation
2. `create_quote(projectId)` — creates a **new** quote record (not overwriting previous)
3. **First: show internal cost summary** to contractor (Internal Cost Summary template) — includes עלות, מחיר ללקוח, and רווח
4. Ask contractor to review and approve the costs:
   ```
   "האם העלויות נראות תקינות? אפשר לתקן לפני שנמשיך להצעה ללקוח."
   ```
5. **If contractor approves costs** → skip to step 7
6. **If contractor wants corrections** → use "Edit offer rows" flow from Step 4 → then return to step 1 to regenerate
7. **Only after explicit cost approval** → show client quote with changes highlighted
8. **If contractor wants corrections to client quote** → use "Edit offer rows" with offer_type="client" → regenerate → show updated client quote
9. **Only after explicit client approval** → proceed to Step 6 (Send)

## Step 6: Send (שליחה)

1. Format WhatsApp message (TEMPLATES.md)
2. Show to contractor for approval
3. **Only after confirmation** → `send_whatsapp(phone, message)`

---

## BOQ Update Variant

When contractor uploads a revised BOQ file:

1. Steps 1-3 same as above
2. In Step 4: `parse_boq(document_id)` → group by Category → match via `get_catalog_candidates` → `scan_room` per room
3. Continue from Step 5 (generate quote)

### BOQ re-import
If scan_room returns room_exists during BOQ import, ask:
"חדר זה כבר קיים בפרויקט. האם להחליף את הפריטים הקיימים?"
→ Yes: call replace_room_items with the new items from BOQ
→ No: skip this room
