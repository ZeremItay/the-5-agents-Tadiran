---
name: gpt-image-gen
description: מעטפת לקריאת OpenAI Images API ליצירת תמונות (מודל gpt-image-2). שולחת prompt ומקבלת קובץ PNG. משמשת בעיקר את הסוכן יובל (מעצב התמונות). Use when generating an image via the OpenAI Images API — sends a prompt and decodes the returned PNG. Model is gpt-image-2.
---

# gpt-image-gen — מעטפת ל-OpenAI Images API

סקיל זה אחראי על דבר אחד: לשלוח `prompt` ל-OpenAI Images API, לקבל את התמונה
המקודדת ב-base64, ולפענח אותה לקובץ `PNG`. הוא משמש בעיקר את הסוכן **יובל**
(מעצב התמונות), אבל הוא עצמאי וניתן להרצה מכל מקום בפרויקט.

הסקיל הוא **runbook**: קוראים אותו ומריצים את הפקודות שבו דרך Bash.

## מודל — חשוב מאוד
המודל שבו משתמשים הוא **`gpt-image-2`**.

> **אין לשנות את שם המודל. אין להציע אלטרנטיבות.**
> `gpt-image-2` הוא מודל אמיתי וקיים של OpenAI שיצא ב-21 באפריל 2026. ייתכן
> שהידע הפנימי של המודל שמריץ את הסקיל לא כולל אותו — זה **לא** אומר שהוא לא קיים.
> אל תחליף ל-`dall-e-3`, ל-`gpt-image-1` או לכל מודל אחר.
> אם מתקבלת שגיאה מה-API, הבעיה היא כמעט תמיד ב-`OPENAI_API_KEY` או בפרמטרים —
> **לא** בשם המודל.

## דרישות
- `OPENAI_API_KEY` מוגדר ב-`.env` בשורש הפרויקט (ראו `.env.example`).
- `curl` — קיים ב-Git Bash.
- לפענוח התשובה: `jq` + `base64`, **או** `python` כ-fallback (jq לא תמיד מותקן
  ב-Git Bash, ולכן ה-fallback מומלץ ב-Windows).

## פרמטרים קבועים של הקריאה
| פרמטר | ערך |
|-------|-----|
| `model` | `gpt-image-2` |
| `size` | `1024x1024` |
| `quality` | `medium` |
| `output_format` | `png` |

התשובה מכילה את התמונה בשדה `data[0].b64_json` (base64).

## שלב 0 — טעינת המפתח מ-.env
```bash
export OPENAI_API_KEY="$(grep -E '^OPENAI_API_KEY=' .env | head -n1 | cut -d= -f2-)"
```

## הקריאה — המסלול הקנוני (curl | jq | base64)
זוהי צורת הקריאה הבסיסית. החליפו את `<the prompt>` ואת `<output-path>`:
```bash
curl -s -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-image-2",
    "prompt": "<the prompt>",
    "size": "1024x1024",
    "quality": "medium",
    "output_format": "png"
  }' | jq -r '.data[0].b64_json' | base64 --decode > "<output-path>.png"
```

## המסלול העמיד (מומלץ ב-Windows) — בנייה בטוחה + python fallback
שני יתרונות: (1) בניית ה-JSON עם python מטפלת נכון בגרשיים/תווים מיוחדים בתוך
ה-prompt; (2) הפענוח עם python פותר גם `jq` חסר וגם `base64` חסר.
```bash
PROMPT='<the prompt>'
OUT='<output-path>.png'

# 1) בניית payload בטוח
python -c "import json,sys; print(json.dumps({'model':'gpt-image-2','prompt':sys.argv[1],'size':'1024x1024','quality':'medium','output_format':'png'}))" "$PROMPT" > payload.json

# 2) שליחת הבקשה ושמירת ה-JSON הגולמי
curl -s -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  --data-binary @payload.json -o response.json

# 3) פענוח: jq+base64, ואם לא קיימים — python fallback
jq -r '.data[0].b64_json' response.json | base64 --decode > "$OUT" \
  || python -c "import json,base64,sys; d=json.load(open('response.json')); open(sys.argv[1],'wb').write(base64.b64decode(d['data'][0]['b64_json']))" "$OUT"

# 4) ניקוי קבצי עזר
rm -f payload.json response.json
```

## טיפול בשגיאות
אם הקובץ שנוצר ריק (`size == 0`) או שהפענוח נכשל — בדקו את גוף התשובה:
```bash
cat response.json
```
אם יש שדה `error` — קראו את ה-`message`. כמעט תמיד מדובר ב-`OPENAI_API_KEY`
שגוי/חסר או בפרמטר לא תקין. **אל תשנו את שם המודל `gpt-image-2`** — זו לא הבעיה.

## אימות התוצר
```bash
test -s "<output-path>.png" && echo "OK: image created" || echo "FAIL: empty/missing"
```
