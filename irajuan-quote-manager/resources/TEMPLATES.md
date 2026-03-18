# Message Templates

## Quote Summary (WhatsApp — sent to customer)

```
📋 הצעת מחיר — [project name]
👤 לקוח: [client name]
📅 תאריך: [date]

🏠 פריטים כלליים:
  • [item] — [qty] [unit] × ₪[client_price] = ₪[total]
  ...

🔧 עבודות מיוחדות:
  • [job] — [days] ימים × ₪[daily_rate] = ₪[total]
  ...

📍 [room name] ([sqm] מ"ר):
  • [item] — [qty] [unit] × ₪[client_price] = ₪[total]
  ...
  סה"כ חדר: ₪[room_total]

[repeat for each room]

💰 סה"כ הצעה: ₪[grand_total_client]
```

**Rules:**
- Use מחיר ללקוח only — never include עלות
- Omit sections with no items (e.g., skip עבודות מיוחדות if empty)
- Omit sqm from room header if not set
- Format prices with comma separator for thousands (e.g., ₪12,500)

---

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

## Send Confirmation Prompt

```
הנה ההודעה שתישלח ללקוח בוואטסאפ:

[formatted quote message]

לשלוח?
```

---

## Success Confirmations

**Quote created:**
```
✅ הצעת מחיר נוצרה בהצלחה!
```

**Message sent:**
```
✅ ההצעה נשלחה בהצלחה ל-[phone]!
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
