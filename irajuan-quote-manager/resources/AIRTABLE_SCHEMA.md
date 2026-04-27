# Airtable Schema — י.ה.ב רג׳ואן

**Base ID**: `apprhc2tpgcXvzPS8`

---

## לידים (Leads)

**Table ID**: `tblse0TVEyKMRGg1n`

| Field | Type | Required | Allowed Values |
|-------|------|----------|----------------|
| שם מלא | Text | Yes | — |
| טלפון | Text | Yes | — |
| אימייל | Email | No | — |
| כתובת | Text | No | — |
| ערוץ הגעה | Select | No | פייסבוק, המלצה, גוגל, אינסטגרם |
| סוג עבודה מבוקש | Select | No | שיפוץ דירה, צביעת דירה, התקנת מזגן, שיפוץ אמבטיה, החלפת דלתות, שיפוץ מטבח, התקנת פרקט, שיפוץ כללי, התקנת חלונות |
| סטטוס ליד | Select | No | חדש, בטיפול, סגור |
| פרוייקט | Link | No | → פרויקטים |
| הצעות מחיר של לקוח | Link | No | → הצעות מחיר של לקוח |

---

## פרויקטים (Projects)

**Table ID**: `tblmD7bxVvDYnbgQN`

| Field | Type | Required | Allowed Values |
|-------|------|----------|----------------|
| שם פרוייקט | Text | Yes | — |
| לידים | Link | Yes | → לידים |
| סוג הפרוייקט | Select | No | בית כנסת, דירה, וילה, חנות, משרדים, אחר |
| סטטוס ביצוע | Select | No | בתהליך, הושלם, בתכנון, נוצר הצעת מחיר, פרוייקט חדש |
| כתובת הפרויקט | Text | No | — |
| שווי פרוייקט | Currency | No | — |

---

## הצעות מחיר של לקוח (Quotes)

**Table ID**: `tblW2mLMjKDPui79c`

| Field | Type | Required |
|-------|------|----------|
| שם הצעת מחיר | Text | Yes |
| פרוייקט | Link | Yes | → פרויקטים |
| ליד | Link | Yes | → לידים |
| לקוח | Lookup | Auto (from ליד) |
| prompt for offer | Long Text | No |
| לינק להצעת מחיר | URL | No |
| לינק להצעת מחיר עלות | URL | No |

---

## כתבי כמויות (BOQ)

**Table ID**: `tbl0uQEvpMYcJKxH6`

| Field | Type | Required |
|-------|------|----------|
| שם כתב כמויות | Text | Yes |
| לינק לכתב כמויות | URL | No |
| פרוייקט | Link | No | → פרויקטים |
| ליד | Link | No | → לידים |
| לקוח | Lookup | Auto (from ליד) |

---

## Relationships

```
לידים (1) ──→ (N) פרויקטים
לידים (1) ──→ (N) הצעות מחיר של לקוח
פרויקטים (1) ──→ (N) הצעות מחיר של לקוח
פרויקטים (1) ──→ (N) כתבי כמויות
לידים (1) ──→ (N) כתבי כמויות
```

## Notes

- Rooms and items are stored in **PostgreSQL**, not Airtable. Airtable holds leads, projects, quotes, and BOQs.
- Quote records contain `rooms_snapshot` (JSONB in Postgres) — a frozen copy of rooms/items at quote creation time.
- Link fields use Airtable record IDs (e.g., `recXXXXXXXXXXXXXX`). Only use IDs returned from tool responses.
