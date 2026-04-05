# Catalog Matching Rules

## Matching Algorithm

Claude performs catalog matching using `get_catalog_candidates` results. For each item, Claude receives the top 3 candidates with similarity scores, costs, and hints.

### Step 1: Get Candidates
- Call `get_catalog_candidates(items="פריט1|פריט2|...")` with pipe-separated item names
- Returns top 3 catalog candidates per item, ranked by `GREATEST(similarity(), word_similarity())` using `pg_trgm`

### Step 2: Claude Selects Best Match
For each item, apply these rules in order:

1. **High similarity auto-pick**: Top score ≥ 0.8 AND gap to second candidate ≥ 0.15 → select top candidate
2. **Paint items** → select by project room count tier (see Paint Item Logic below)
3. **Tiered items** (have `minRooms`/`maxRooms` or `hint`) → use hint + project context to select correct tier
4. **Ambiguous** (multiple close candidates) → ask contractor to choose
5. **No match** (top score < 0.4 or irrelevant candidates) → ask contractor:
   ```
   "הפריט '[item name]' לא נמצא בקטלוג. מה תעדיף?
   1. לחפש מחירים בגוגל
   2. להזין מחיר ידנית"
   ```
   - **Option 1 (Google search)**: Use `WebSearch` to search for the item (e.g. "מחיר [item name] שיפוצים"). Show top results with links to the contractor. Contractor reviews and provides final pricing based on findings.
   - **Option 2 (Manual)**: Ask contractor for cost (עלות) and client price (מחיר ללקוח).
   - After getting prices either way → `update_catalog` → use returned catalog_id

---

## Paint Item Logic (צביעה/צבע)

Paint items require special handling:

1. **Never auto-matched** — always check room tier even if similarity is high
2. **Priced as קומפלט** (complete package) based on total rooms count
3. **Tiered catalog entries**: "צביעת קירות — דירה 4 חדרים", "צביעת קירות — דירה 5 חדרים"
4. **Selection uses** `minRooms`/`maxRooms` from candidates — pick the tier matching project room count
5. **Deduplication**: if global room ("כללי") has a paint item → room-level paint items are suppressed

---

## SQM-Based Items (פריטים לפי מ״ר)

- Items with unit **"מ״ר"** use the room's sqm as quantity
- If room sqm is unknown → ask contractor for room size before creating room
- Claude sets quantity = sqm when building the items array for `scan_room`

---

## Tiered Pricing

Catalog supports conditional selection via these columns:

| Column | Purpose |
|--------|---------|
| `ai_selection_hint` | Free-text instruction for when to select this item |
| `min_rooms` | Minimum room count for this tier |
| `max_rooms` | Maximum room count for this tier |
| `min_sqm` | Minimum project sqm for this tier |
| `max_sqm` | Maximum project sqm for this tier |

Claude uses project context (room count, sqm) to pick the right tier from candidates.

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

1. **Skip catalog matching** — do not call `get_catalog_candidates` for this item
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
1. `get_catalog_candidates(items="[description]")` — same as not_komplet items
2. Apply Step 2 selection rules: high similarity auto-pick, paint tiers, ambiguous → ask contractor, no match → ask contractor
3. Enrich item with: `catalog_id`, `unit_cost`, `unit_client_price`, actual `Quantity`, actual `Unit` (from catalog, not "קומפלט")

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
