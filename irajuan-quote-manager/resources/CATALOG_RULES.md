# Catalog Matching Rules

## Matching Algorithm

Claude performs catalog matching using `SearchStore` (Google Gemini File Search). The RAG store contains catalog data with pricing knowledge. Claude queries it with natural language and extracts structured pricing from the free-text response.

### Step 1: Query SearchStore
- Call `SearchStore(query="natural language item description")` — can include multiple items in one query
  - **Small batches (≤5 items)**: combine in one query for efficiency
  - **Large batches (>5 items)**: split into multiple calls to keep responses focused
- **Query formulation guidance:**
  - Use natural language, not pipe-separated syntax
  - Paint items: include room count → `"צביעת דירה 5 חדרים"`
  - SQM items: include area context → `"ריצוף 40 מ״ר"`
  - Tiered items: include project size → `"לוח חשמל דירה 4 חדרים"`
  - Multi-item: `"מצא מחירים עבור: צביעת דירה 5 חדרים, פרקט למינציה, שפכטל שתי ידיים"`

### Step 2: Extract Pricing from Response
For each item, extract from the response text:
1. **cost (עלות)** — contractor's actual cost per unit
2. **cost_for_client (מחיר ללקוח)** — client-facing price per unit
3. **unit** — pricing unit (יחידה / מטר / מ״ר / קומפלט)
4. **notes** — relevant conditions, tier info, considerations

Then apply:
- **Auto-pick** — extract and use the pricing directly
- **Ambiguous/multiple ranges** → ask contractor to choose
- **No relevant pricing in response** → ask contractor:
   ```
   "הפריט '[item name]' לא נמצא בקטלוג. מה תעדיף?
   1. לחפש מחירים בגוגל
   2. להזין מחיר ידנית"
   ```
   - **Option 1 (Google search)**: Use `WebSearch` to search for the item (e.g. "מחיר [item name] שיפוצים"). Show top results with links to the contractor. Contractor reviews and provides final pricing based on findings.
   - **Option 2 (Manual)**: Ask contractor for cost (עלות) and client price (מחיר ללקוח).
   - After getting prices either way → `update_catalog`

---

## Paint Item Logic (צביעה/צבע)

Paint items require special handling:

1. **Include room count in query** — e.g. `SearchStore(query="צביעת דירה 5 חדרים")` → extract correct tier pricing from response
2. **Priced as קומפלט** (complete package) based on total rooms count
3. **Tiered catalog entries**: "צביעת קירות — דירה 4 חדרים", "צביעת קירות — דירה 5 חדרים"
4. **Deduplication**: if global room ("כללי") has a paint item → room-level paint items are suppressed

---

## SQM-Based Items (פריטים לפי מ״ר)

- Items with unit **"מ״ר"** use the room's sqm as quantity
- If room sqm is unknown → ask contractor for room size before creating room
- Claude sets quantity = sqm when building the items array for `scan_room`

---

## Tiered Pricing

Some catalog items have tiered pricing based on project size (room count, sqm). The tier information is embedded in the RAG store text.

- **Include project context in query** — e.g. `SearchStore(query="לוח חשמל דירה 4 חדרים")` or `SearchStore(query="ריצוף 80 מ״ר")`
- Agent extracts the correct tier from the response text based on the project's room count and sqm

---

## Unit Types

| Hebrew | English | When Used |
|--------|---------|-----------|
| יחידה | Unit | Single items (door, toilet, A/C) |
| מטר | Linear meter | Linear work (skirting, cabling) |
| מ״ר | Square meter | Area work (flooring, tiling) |
| מ״א | Linear meter (alt.) | Alternative notation for מטר |
| קומפלט | Complete | Package pricing (full room paint) |

---

## Price Fields

Two prices tracked for every item:

| Field | Hebrew | Meaning |
|-------|--------|---------|
| `cost` | עלות | Contractor's actual cost |
| `cost_for_the_client` | מחיר ללקוח | Price charged to client (with markup) |

Per-item calculation:
- `unit_cost × quantity = total_cost`
- `unit_client_price × quantity = total_client_price`

**Never show עלות to the customer.** Customer-facing quotes use מחיר ללקוח only.

---

## "יתומחר בהמשך" — To Be Priced Later

When an item is marked as **"יתומחר בהמשך"** (by the contractor, or auto-detected by the agent in BOQ mode):

1. **Skip catalog matching** — do not call `SearchStore` for this item
2. **Insert with zero costs** — `unit_cost: 0`, `unit_client_price: 0`, `_isCompleted: false`, `Status: "Pending Quote"`
3. **Unit = "קומפלט"** — always set unit to "קומפלט" for these items
4. **Never add to catalog** — do NOT call `update_catalog`. This item stays out of the catalog even when prices are provided later
5. **When contractor later provides prices**:
   - Manual mode: update the room item via `replace_room_items` with the actual prices, keeping unit as "קומפלט"
   - BOQ mode: update via `update_or_create_with_boq` with the actual prices
   - Still no `update_catalog` in either mode

---

## Catalog-Resolvable Komplet Items (BOQ Mode)

Some BOQ items arrive as "קומפלט" not because they are lump-sum work, but because the BOQ didn't specify quantities. These describe standard catalog work and can be resolved through catalog matching once a quantity is provided.

### Identification

The agent analyzes Description to determine if it describes **common, standard** construction work that exists in the catalog under a measurable unit. Signals:
- Description contains a recognizable **standard** work type: חיפוי (tiling), ריצוף (flooring), פרקט (parquet), צביעה (painting), גבס (drywall), שפכטל (plastering), דלתות פנים (interior doors), ברזים (faucets), נקודות חשמל (electrical points), etc.
- The work would naturally be priced per unit in the catalog, not as קומפלט
- The only reason it's קומפלט is missing quantity information

**Not catalog-resolvable** — specialty/custom work goes to Group C: custom glass (מחיצות זכוכית, דלתות זכוכית), custom carpentry (נגרות אומן), signage (שילוטים), fire/safety systems (ספרינקלרים, גלאי עשן), HVAC systems, multi-trade composites.

When uncertain whether an item is catalog-resolvable → default to manual pricing (Group C). Don't force catalog matching on items that don't clearly fit.

### Quantity Resolution

1. **Quantity in Description** — extract directly (e.g., "החלפת 3 דלתות" → 3). No need to ask.
2. **Quantity missing** — ask contractor with the expected unit:
   - Area work (חיפוי, ריצוף, פרקט) → ask for מ"ר
   - Linear work (צנרת, אדנים) → ask for מטר
   - Unit work (דלתות, ברזים, נקודות חשמל) → ask for count (יחידות)

### Catalog Matching

After obtaining quantity, resolve through the standard catalog flow:
1. `SearchStore(query="[description with context]")` — same as not_komplet items
2. Extract pricing from response: auto-pick, ambiguous → ask contractor, no match → ask contractor
3. Enrich item with: `unit_cost`, `unit_client_price`, actual `Quantity`, actual `Unit` (from catalog, not "קומפלט")

### Fallback

If catalog matching fails (no suitable candidate, contractor rejects all options) → treat as manual-pricing komplet item: revert to unit="קומפלט", Quantity=1, ask contractor for lump-sum pricing.

### Important

- Item's `Unit` changes from "קומפלט" to the catalog unit (e.g., "מ"ר") on successful match
- Item's `Quantity` changes from 1 to the actual quantity
- These items get `_isCompleted: true`, `Status: "Priced"` after successful matching

---

## Work-Day Pricing (תעריף יום עבודה)

- Catalog contains a **"תעריף יום עבודה"** entry with a daily rate
- Applies to **any room** (global, special jobs, or room-by-room) — not limited to special jobs
- **Quantity is always 1**. Multiply the daily rate × number of work days to compute `unit_cost` and `unit_client_price`
  - Example: "טיפול רטיבות 5 ימי עבודה" with daily rate cost=200, client_price=300 → `quantity: 1, unit_cost: 1000, unit_client_price: 1500`
- Alternative: contractor specifies a fixed total price → quantity=1, unit_cost=fixed price
