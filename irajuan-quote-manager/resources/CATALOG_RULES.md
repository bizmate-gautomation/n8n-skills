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

## Special Job Pricing (תעריף יום עבודה)

- Catalog contains a **"תעריף יום עבודה"** entry with a daily rate
- Special jobs: quantity = number of work days
- Alternative: contractor specifies a fixed total price (overrides catalog rate)
