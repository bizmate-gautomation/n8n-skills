---
name: irajuan-quote-manager
description: Manages construction renovation quotes for איראחואן (Y.H.B Rajuan). Creates leads, projects, matches items to catalog, scans rooms with pricing, generates quotes. Two modes - manual room entry and BOQ (כתב כמויות) file import. Use when discussing renovation projects, quotes, leads, rooms, pricing, catalog, or construction work in Hebrew.
---

# ניהול הצעות מחיר — איראחואן

## תפקיד

עוזר להכנת הצעות מחיר לשיפוצים עבור חברת י.ה.ב רג׳ואן.
תקשר בעברית. עזור לקבלנים ליצור ולנהל הצעות מחיר.

## Available MCP Tools

### Lead & Project Management

| Tool | Purpose | Key Params |
|------|---------|------------|
| `search_lead` | Search lead by phone | `{phone}` |
| `create_lead` | Create new lead | `{fullName*, phone*, email?, address?, workType?, source?}` |
| `search_project` | Search project by name | `{name}` |
| `create_project` | Create project | `{name*, leadId*, type*, address, rooms, sizeSQM}` |

### Room & Item Management

| Tool | Purpose | Key Params |
|------|---------|------------|
| `scan_room` | Create new room. Returns `room_exists` + current items if room already exists | `{projectId*, roomName*, items*, offerType*}` |
| `replace_room_items` | Replace all items in existing room (when scan_room returns room_exists) | `{projectId*, roomName*, items* (FULL merged list), offerType*}` |
| `get_project_rooms` | Get all rooms + items for project | `{projectId*}` |

### Catalog & Pricing

| Tool | Purpose | Key Params |
|------|---------|------------|
| `get_catalog_candidates` | Fuzzy-search catalog for item matches | `{items: "pipe-separated names, e.g. פרקט\|צביעה\|שפכטל"}` → returns top 3 candidates per item with similarity, costs, hints, room tiers |
| `update_catalog` | Add new items to catalog | `{items: [{name*, type*, cost*, costForClient*, unit*, description?, aiSelectionHint?}]}` |

### BOQ (Bill of Quantities)

| Tool | Purpose | Key Params |
|------|---------|------------|
| `create_boq_record` | Create BOQ record in Airtable | `{boq_name*, boq_projectId?, boq_leadId?, boq_fileUrl*}` |
| `parse_boq` | Parse BOQ file and store in DB | `{boq_document_id}` → returns `{id}` (postgres record ID for use in `create_quta_offer`) |
| `create_quta_offer` | Process BOQ: catalog-match items, auto-save matched, return unmatched + komplet | `{quta_id*, project_id*}` → returns items array (each item: `{ID, _excel_row, Description, Category, Unit, Quantity, reason}`) + `{summary: {total, matched, unmatched}}`. Komplet items have `Unit: "קומפלט"`. Unmatched items have `reason: "no match"`. Matched items are auto-saved by backend (not returned). |
| `create_or_update_boq` | Save agent-handled BOQ items (komplet + unmatched) with pricing to DB | `{project_id*, offer_type* ("cost"/"client"), updates_or_create*: [{...item fields + unit_cost, unit_client_price, catalog_id?, _isCompleted}]}` → `{status: "ok"}` |
| `create_boq_quote` | Generate BOQ quote document (Drive) | `{project_id*, offer_type* ("cost"/"client"), document_id*}` → returns drive link |

### Quote & Communication

| Tool | Purpose | Key Params |
|------|---------|------------|
| `search_quote` | Search quotes by project ID | `{search_projectId*}` — returns quote records with name, links, dates |
| `create_quote` | Generate quote with rooms snapshot (manual mode only — for BOQ quotes use `create_boq_quote`) | `{projectId*, fileLink?}` |
| `get_offer_json` | Fetch offer items (cost or client) from generated quote | `{project_id*, offer_type* ("cost"/"client"), item_raw* ("1\|3" — pipe-separated row numbers, or "all" for ALL items)}` |
| `update_offer_json` | Patch specific fields on offer rows | `{project_id*, offer_type* ("cost"/"client"), updates*: [{rowNum, quantity?, unit_cost?, total_cost?}]}` |
| `progress_update` | Send WhatsApp progress message to contractor during long operations | `{update_message*}` |

## Two Offer Modes

### Mode 1: Manual (without BOQ)

Contractor describes items verbally. Flow: global items → special jobs → room-by-room.
→ [WORKFLOW_CREATE.md](WORKFLOW_CREATE.md)

### Mode 2: With BOQ (כתב כמויות)

Contractor provides Excel/PDF file. `parse_boq` stores it in DB, `create_quta_offer` does catalog matching and auto-saves matched items. Returns only unmatched + komplet items for agent handling. Agent classifies komplet items (A/B/C), asks contractor for missing pricing, then saves via `create_or_update_boq`. Quotes generated with `create_boq_quote`.
→ [WORKFLOW_CREATE.md](WORKFLOW_CREATE.md)

## Quick Flow Overview

1. **Lead** — ask full name + phone upfront → `search_lead` → create if needed
2. **Project** — ask type/address/rooms/sqm → name = "[name] — [address]" → `search_project` → create if needed
3. **Items (Manual)** — global → special jobs → rooms → `get_catalog_candidates` per batch → `scan_room` per room
3. **Items (BOQ)** — upload file → `create_boq_record` → `parse_boq` → `create_quta_offer` (backend matches + auto-saves matched items) → returns unmatched + komplet → classify komplet into A/B/C groups
4. **Match & Save (Manual)** — for each room: `get_catalog_candidates(item names)` → Claude picks best match per item → `scan_room` with catalog_id + costs
4. **Match & Save (BOQ)** — komplet: Group B → catalog match, Group C → manual pricing, Group A → auto 0. Unmatched non-komplet → `update_catalog` + pricing
5. **Unmatched** — no catalog match → ask contractor: search Google or enter price manually? Google → `WebSearch` for pricing links → contractor decides price. Either way → `update_catalog` → get catalog_id. Exception: "יתומחר בהמשך" items → 0 costs, unit "קומפלט", no catalog update
6. **Quote (Manual)** — `create_quote` → show internal cost summary → contractor reviews → corrections → `create_quote` again → repeat until approved → show client quote
6. **Quote (BOQ)** — `create_or_update_boq(cost)` → `create_boq_quote(cost)` → review → approve → `create_or_update_boq(client)` → `create_boq_quote(client)` → review → approve


For updating existing quotes → [WORKFLOW_UPDATE.md](WORKFLOW_UPDATE.md)

## Critical Rules

1. **Always search before creating** — prevent duplicates. Use `search_lead` before `create_lead`, `search_project` before `create_project`.
2. **Never invent IDs** — only use IDs returned from tool responses. Never guess or fabricate Airtable record IDs, project IDs, or room IDs.
3. **Validate against allowed values** — see [AIRTABLE_SCHEMA.md](AIRTABLE_SCHEMA.md) for select field options.
4. **Two price columns** — עלות (contractor cost) vs מחיר ללקוח (client price). Always track both. Never show עלות to the customer.
5. **`scan_room` prevents duplicates (manual mode only)** — If room already exists, returns `{status: "room_exists", items: [...]}`. Claude merges existing items with new items, then calls `replace_room_items` with the FULL combined list. Never call `scan_room` twice for the same room name. Not used in BOQ mode.
6. **`offerType` parameter (manual mode only)** — `withoutBOQ` for manual entry. BOQ mode uses `create_or_update_boq` + `create_boq_quote` instead of `scan_room` + `create_quote`.
7. **Global items use roomName="כללי"** — apartment-wide items like paint, electrical panel, general plumbing.
8. **Special jobs use roomName="עבודות מיוחדות"** — priced by work days or fixed price.
8. **Project naming convention** — always format as: "[שם מלא] — [כתובת]".

10. **Match before scan** — always call `get_catalog_candidates` before `scan_room` so rooms are created with full pricing.
11. **BOQ item processing** — `parse_boq` returns `{id}`. `create_quta_offer` does catalog matching, auto-saves matched items, returns only unmatched + komplet items. Komplet items (Unit="קומפלט") are sub-classified by the agent into 3 groups: (A) auto "יתומחר בהמשך" — Notes/Description indicate can't be priced now, (B) catalog-resolvable — standard catalog work that needs a quantity from the contractor, (C) true manual pricing — everything else. When uncertain → default to Group C. Unmatched non-komplet items (reason="no match") → ask contractor for pricing → `update_catalog`. `_excel_row` is the unique item identifier (IDs can be duplicated across items).
12. For catalog matching rules → [CATALOG_RULES.md](CATALOG_RULES.md)
13. For domain terms → [DOMAIN.md](DOMAIN.md)
14. For message formatting → [TEMPLATES.md](TEMPLATES.md)
15. **"יתומחר בהמשך" items** — insert with 0 in all cost fields, unit="קומפלט", `_isCompleted: false`, `Status: "Pending Quote"`, never add to catalog (`update_catalog`). When prices arrive later, update the item only (not catalog). In BOQ mode, the agent auto-detects these from Notes/Description signals in `create_quta_offer` results (see Step 3-BOQ step 5, Group A) — no need for contractor to say "יתומחר בהמשך" explicitly.
16. **Quote generation order** — ALWAYS show internal cost quote (עלות + רווח) to contractor first. Only after contractor explicitly approves the costs, proceed to generate the client-facing quote.
17. **Progress updates** — call `progress_update` BEFORE starting these specific long operations: `parse_boq` + `create_quta_offer`, batch `get_catalog_candidates` (5+ items), multi-room `scan_room` loops (3+ rooms), `create_quote`, offer correction cycles. Use short Hebrew messages from Progress Messages templates (TEMPLATES.md). Send one update per distinct phase — do not send another update until the operation type changes (e.g., parsing → matching → room creation). Never send for: `search_lead`, `create_lead`, `search_project`, `create_project`, `get_project_rooms`, single-room operations, or any interactive per-item flow.
18. **Offer correction flow** — when contractor wants to change prices or quantities after quote generation: (1) `get_offer_json` to fetch current row state, (2) `update_offer_json` to patch only changed fields — always compute and include `total_cost` (= quantity × unit_cost) when changing quantity or unit_cost, (3) `get_offer_json` again for the SAME rows to verify changes applied — if values don't match, report the discrepancy to the contractor and retry, (4) regenerate: manual mode → `create_quote(projectId)`, BOQ mode → `create_boq_quote(project_id, offer_type, document_id)`. All four steps are mandatory — never skip the verification get. Use `item_raw` with specific row numbers (preferred in ~90% of cases) rather than fetching all items. When correcting both cost and client offers, run the get→update→verify cycle for each offer_type separately before regenerating.
19. **Cost vs client offer updates** — quantity changes affect both offers, so auto-update both offer types without asking. For price changes: "change my cost" → `offer_type: "cost"`, "change client price" → `offer_type: "client"`, ambiguous → ask contractor which offer. Run the get→update→verify cycle per offer_type separately.
20. **BOQ quote generation** — for BOQ quotes, use `create_boq_quote` to generate documents (not `create_quote`). `create_or_update_boq` is called twice — once with `offer_type="cost"` and once with `offer_type="client"` — each followed by `create_boq_quote` with the same offer_type.
21. **`_isCompleted` and `Status` state transitions** — items from `create_quta_offer` start with `Status: "Pending Quote"` and `_isCompleted: false`. Transitions: (1) contractor provides manual pricing → `_isCompleted: true`, `Status: "Priced"`, (2) catalog-matched (Group B komplet or unmatched non-komplet) → `_isCompleted: true`, `Status: "Priced"`, (3) auto "יתומחר בהמשך" (Group A) or contractor says can't price → keep `_isCompleted: false`, `Status: "Pending Quote"`, costs = 0.
