# Message Templates

## Internal Cost Summary (contractor only — NOT sent to customer)

```
📊 סיכום עלויות פנימי:
סה"כ עלות: ₪[grand_total_cost]
סה"כ ללקוח: ₪[grand_total_client]
רווח: ₪[profit] ([profit_pct]%)
```

---

## Room Parsed Confirmation

```
✅ חדר: [room_name] ([sqm] מ"ר)
פריטים שנוספו:
  • [item_name] — [qty] [unit]
  • [item_name] — [qty] [unit]
  ...

האם זה נכון? ניתן לתקן או להוסיף פריטים.
```

---

## Unmatched Items Request

```
⚠️ נמצאו [N] פריטים שלא נמצאו בקטלוג:

1. [item_name] ([room_name]) — [qty] [unit]
   סיבה: [reason]

2. [item_name] ([room_name]) — [qty] [unit]
   סיבה: [reason]

...

אנא ספק עבור כל פריט:
• עלות (מחיר שלך)
• מחיר ללקוח
• קטגוריה (סוג העבודה)
• יחידת מידה (יחידה/מטר/מ״ר/קומפלט)
```

---

## Success Confirmations

**Quote created:**
```
✅ הצעת מחיר נוצרה בהצלחה!
```

**Lead created:**
```
✅ ליד חדש נוצר: [name] ([phone])
```

**Project created:**
```
✅ פרויקט חדש נוצר: [project_name]
```

---

## Offer Row Details (contractor only — for correction flow)

```
📝 שורות בהצעה ([offer_type]):

[rowNum]. [name] — [qty] × ₪[unit_cost] = ₪[total_cost]
[rowNum]. [name] — [qty] × ₪[unit_cost] = ₪[total_cost]

מה תרצה לשנות?
```

After update + verify:
```
✅ עודכן:
[rowNum]. [name] — [qty] × ₪[unit_cost] = ₪[total_cost]
```

---

## Current Quote State (contractor only — for update flow)

```
📋 מצב נוכחי של ההצעה — [quote file_name]:

💰 עלות (פנימי):
[rowNum]. [name] — [quantity] × ₪[unit_cost] = ₪[total_cost]
[rowNum]. [name] — [quantity] × ₪[unit_cost] = ₪[total_cost]
...
סה"כ עלות: ₪[sum of all total_cost]

👤 מחיר ללקוח:
[rowNum]. [name] — [quantity] × ₪[unit_cost] = ₪[total_cost]
[rowNum]. [name] — [quantity] × ₪[unit_cost] = ₪[total_cost]
...
סה"כ ללקוח: ₪[sum of all total_cost]

רווח: ₪[client_total - cost_total] ([profit_pct]%)

מה תרצה לשנות?
```

---

## Quote Selection List (contractor only — for update flow)

```
📋 נמצאו [N] הצעות מחיר עבור [lead name]:

1. [שם הצעת מחיר] — [תאריך יצירה]
2. [שם הצעת מחיר] — [תאריך יצירה]
...

באיזו הצעה תרצה לטפל?
```

---

## Progress Messages (הודעות התקדמות — sent to contractor via progress_update)

Use these messages with `progress_update` during long operations. Send BEFORE the operation starts.

### BOQ Processing
```
⏳ מעבד את כתב הכמויות...
```

### Catalog Matching (batch, 5+ items)
```
⏳ מתאים פריטים לקטלוג...
```

### Room Creation (batch, 3+ rooms)
```
⏳ יוצר חדרים בפרויקט...
```

### Catalog Search (manual mode, 5+ items)
```
⏳ מחפש פריטים בקטלוג...
```

### Quote Generation
```
⏳ מכין הצעת מחיר...
```

### Offer Correction (עדכון שורות)
```
⏳ מעדכן הצעת מחיר...
```

**Rules:**
- Send BEFORE the operation starts, not after
- One update per phase — only send a new update when entering a different operation type
- Never send for single-room operations, lead/project creation, or interactive per-item flows
