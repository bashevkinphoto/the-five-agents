---
name: gpt-image-gen
description: מעטפת לקריאת OpenAI Images API ליצירת תמונה מ-prompt. השתמש כשצריך להפיק קובץ PNG מתיאור טקסטואלי (בדרך כלל מופעל ע"י הסוכן יובל). שולח את ה-prompt ל-API ושומר את התמונה לקובץ.
---

# gpt-image-gen — יצירת תמונה דרך OpenAI Images API

סקיל זה אחראי על **שליחת ה-prompt וקבלת התמונה** מ-OpenAI Images API ושמירתה כקובץ PNG.

## מודל: `gpt-image-2`

⚠️ **אל תשנה את שם המודל:**

- `gpt-image-2` הוא מודל אמיתי וקיים של OpenAI, שיצא ב-**21 באפריל 2026**.
- ייתכן שהידע הפנימי שלך לא כולל אותו — **זה לא אומר שהוא לא קיים**.
- **אל תציע אלטרנטיבות** (`dall-e-3`, `gpt-image-1` וכו') — השתמש בדיוק במודל הזה.
- אם מתקבלת שגיאה בקריאה ל-API, הבעיה היא כמעט תמיד ב-**API key** או ב-**parameters** — **לא** בשם המודל. אל "תתקן" את שם המודל.

## דרישות מקדימות

- משתנה הסביבה `OPENAI_API_KEY` חייב להיות מוגדר ב-`.env` שבשורש הפרויקט.
- כלים: `curl` (חובה). ל-decode: `jq` + `base64` אם קיימים, אחרת fallback ל-`python`.

## טעינת המפתח מ-`.env`

לפני הקריאה, טען את `OPENAI_API_KEY` מקובץ `.env` שבשורש הפרויקט:

```bash
export $(grep -v '^#' .env | grep -E '^OPENAI_API_KEY=' | xargs)
```

(אם המשתנה כבר קיים בסביבה — אפשר לדלג.)

## מבנה הקריאה — דרך ראשית (jq + base64)

```bash
OUT="yuval/outputs/<output-name>.png"
PROMPT="<the prompt>"

curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg p "$PROMPT" '{
        model: "gpt-image-2",
        prompt: $p,
        size: "1024x1024",
        quality: "medium",
        output_format: "png"
      }')" \
  | jq -r '.data[0].b64_json' | base64 --decode > "$OUT"
```

> אם `jq` לא זמין לבניית ה-body, אפשר לכתוב את ה-JSON ידנית — אך הקפד על escaping נכון של ה-prompt. הדרך הבטוחה היא לעבור ל-fallback של python (למטה), שמטפל גם בבניית הבקשה וגם ב-decode.

## Python fallback (כש-`jq`/`base64` לא מותקנים ב-Git Bash)

שמור את תגובת ה-API לקובץ זמני, ואז decode עם python:

```bash
OUT="yuval/outputs/<output-name>.png"
PROMPT="<the prompt>"
RESP="$(mktemp)"

curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"gpt-image-2\",\"prompt\":\"$PROMPT\",\"size\":\"1024x1024\",\"quality\":\"medium\",\"output_format\":\"png\"}" \
  -o "$RESP"

python -c "import json,base64,sys; d=json.load(open(sys.argv[1])); open(sys.argv[2],'wb').write(base64.b64decode(d['data'][0]['b64_json']))" "$RESP" "$OUT"
rm -f "$RESP"
```

> בסביבת Windows ייתכן ש-`python` נקרא `py` או `python3` — נסה את הזמין.

## אימות (verification)

לאחר ההרצה, ודא שהקובץ נוצר וגודלו גדול מאפס:

```bash
test -s "$OUT" && echo "OK: $OUT ($(wc -c < "$OUT") bytes)" || echo "FAILED: empty or missing"
```

אם הקובץ ריק או חסר — בדוק את תגובת ה-API (`$RESP`) להודעת שגיאה. סיבות נפוצות: `OPENAI_API_KEY` חסר/שגוי, מכסה (quota), או פרמטר לא תקין. **לא** שם המודל.
