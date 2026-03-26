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

**Cost quote created:**
```
✅ הצעת מחיר עלות נוצרה!
📄 הצעת מחיר עלות: [driveLink from create_quote with quote_type="cost"]
```

**Client quote created:**
```
✅ הצעת מחיר ללקוח נוצרה!
📄 הצעת מחיר ללקוח: [driveLink from create_quote with quote_type="client"]
```

**BOQ Cost quote created:**
```
✅ הצעת מחיר עלות מכתב כמויות נוצרה!
📄 הצעת מחיר עלות: [driveLink]
```

**BOQ Client quote created:**
```
✅ הצעת מחיר ללקוח מכתב כמויות נוצרה!
📄 הצעת מחיר ללקוח: [driveLink]
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

## BOQ Komplet Items Pricing Request

```
נמצאו [N_total] פריטי קומפלט:

📏 פריטים שניתן להתאים לקטלוג — נדרשת כמות ([N_B] פריטים):

1. *[ID]* [Description] ([Category])
   הערות: [Notes]
   ← כמה [unit — e.g., מ"ר / מטר / יחידות]?

2. *[ID]* [Description] ([Category])
   ← כמה [unit]?
...

💰 פריטים לתמחור ידני ([N_C] פריטים):

1. *[ID]* [Description] ([Category])
   הערות: [Notes]

2. *[ID]* [Description] ([Category])
...

עבור כל פריט, אנא ספק:
• עלות (מחיר שלך)
• מחיר ללקוח
פריטים שלא ניתן לתמחר כרגע — אמור ״יתומחר בהמשך״.

---

⏳ יתומחר בהמשך ([N_A] פריטים):

1. *[ID]* [Description] ([Category])
   סיבה: [reason from Notes]

2. *[ID]* [Description] ([Category])
   סיבה: [reason]
...

פריטים אלו נשמרים עם עלות 0 ויתומחרו בהמשך.
```

**Section visibility**: omit any group section (including header) if it has 0 items. Order always: B → C → A.

---

## BOQ Not-Komplet Items Summary

```
נמצאו [N] פריטים עם כמויות — מתאים לקטלוג:

1. [ID] [Description] — [Unit]
   התאמה: [catalog_match_name] — עלות: ₪[cost], ללקוח: ₪[client_price]

2. [ID] [Description] — [Unit]
   התאמה: לא נמצא — נדרש תמחור ידני
...
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

### BOQ Items Save
```
⏳ שומר נתוני כתב כמויות...
```

### Offer Correction (עדכון שורות)
```
⏳ מעדכן הצעת מחיר...
```

**Rules:**
- Send BEFORE the operation starts, not after
- One update per phase — only send a new update when entering a different operation type
- Never send for single-room operations, lead/project creation, or interactive per-item flows
