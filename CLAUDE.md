# מערכת ניהול התורים של הדר בק — הקשר לקלוד קוד

קובץ זה נטען אוטומטית כשפותחים את התיקייה ב-Claude Code. הוא נותן את כל ההקשר הדרוש כדי לעבוד על המערכת בלי לשבור אותה.

---

## מה זה

מערכת הזמנת תורים מלאה למטפלת בעיסוי (הדר בק, באר שבע, לנשים בלבד). שלושה דפי לקוח + דשבורד ניהול, מעל Google Sheet אחד.

## ארכיטקטורה — חשוב להבין לפני כל שינוי

```
דפדפן (HTML סטטי)  ──JSONP──►  Google Apps Script (apps-script.js)  ──►  Google Sheet + יומן Google
```

- **צד-לקוח**: 3 קבצי HTML סטטיים (אין build, וניל JS, RTL עברית). מתארחים ב-GitHub Pages.
- **צד-שרת**: `apps-script.js` — קובץ אחד שרץ ב-Google Apps Script, מעל Google Sheet עם 6 גיליונות (הזמנות · ימי_חופש · הצהרות_בריאות · לקוחות · תיק_מטופל · שעות_זמינות).
- **תקשורת**: JSONP (תגית `<script>`) כדי לעקוף CORS. כל פעולה היא `?action=...&callback=...`.

### מפת קבצים
| קובץ | תפקיד |
|---|---|
| `index.html` | דף נחיתה / הפניה |
| `booking.html` | קביעת תור ללקוחה (4 שלבים) + הצהרת בריאות + חתימה |
| `health.html` | הצהרת בריאות עצמאית |
| `admin.html` | דשבורד הדר (מוגן PIN): תורים · שעות · הצהרות · לקוחות · תיק מטופל |
| `update.html` | שירות עצמי ללקוחה — מציאת תור ושינוי מועד (ללא PIN, אימות לפי שם) |
| `apps-script.js` | **השרת** — לא מתארח ב-GitHub, חי בעורך Apps Script |
| `brand-kit.html` | ספר מותג רשמי (צבעים/פונטים/קול) |
| `SETUP.md` | מדריך התקנה מאפס |
| `HANDOFF.md` | מדריך מסירה (איזה חשבון Google מארח את השרת) |

---

## 🚨 שלושה דברים קריטיים שחייבים לדעת

### 1. שינוי ב-apps-script.js לא משפיע עד Redeploy ידני
הקובץ `apps-script.js` כאן הוא **עותק**. השרת האמיתי הוא מה שמודבק בעורך Apps Script של החשבון המארח. **כל שינוי בקוד השרת מחייב:**
> עורך Apps Script → הדבק את הקוד → Deploy → Manage deployments → עיפרון ✏️ → New version → Deploy.

בלי זה — שינויים ב-`apps-script.js` כאן לא עושים כלום בפועל. שינויים ב-HTML (frontend) כן מתעדכנים מיד ב-push ל-GitHub.

### 2. Google Sheets מַמיר תאריך/שעה — תמיד לנרמל
Sheets ממיר אוטומטית תאי תאריך/שעה לאובייקטי `Date`, מה שמחזיר ISO מעוות (`1899-12-30T...`, או תאריך מוסט באזור זמן). זה כבר שבר תצוגה, זיהוי שעות תפוסות (כפל הזמנות!) ותזכורות.
**הפתרון שכבר מוטמע:** הפונקציות `asDateStr(v)` ו-`asTimeStr(v)` ב-apps-script.js מחזירות תמיד `yyyy-MM-dd` ו-`HH:mm` (עם אפס מוביל). **כל קריאה של עמודת תאריך/שעה מ-Sheet חייבת לעבור דרכן.** בכתיבה — מאלצים פורמט טקסט: `sheet.getRange('G:H').setNumberFormat('@')`. בצד-לקוח יש מקבילות: `fmtDateLabel`, `fmtTime`, `fmtDateInput` ב-admin.html.

### 3. מצב WhatsApp = manual (לא שולח לבד)
`WA_PROVIDER = 'manual'` — המערכת **לא** שולחת WhatsApp אוטומטית. במקום זה היא מחזירה `wa_url` והדשבורד פותח את WhatsApp ללחיצת הדר. הודעת האישור ללקוחה נשלחת **רק** כשהדר לוחצת "אישור" בדשבורד (`update_status` → `msgApproved`). אם רוצים שליחה אוטומטית אמיתית — להגדיר Green API (`GREENAPI_INSTANCE`/`TOKEN`).

---

## אירוח (במצב הנוכחי)

האתר חי תחת הריפו של מיטל: `https://meytalp-dev.github.io/ort-training/clients/hadar-massage/`.
שינויי frontend מתפרסמים ב-commit+push לריפו הזה (GitHub Pages, ~1 דק' לעדכון). אם הדר רוצה אירוח עצמאי (דומיין `hadarback.co.il`) — ראו HANDOFF.md.

ה-`const SHEETS_URL` (כתובת ה-/exec של השרת) מופיע ב-3 קבצים: `booking.html`, `health.html`, `admin.html`. אם מחליפים שרת — לעדכן בשלושתם.

---

## קבועים לעריכה (בראש apps-script.js)
- `ADMIN_PIN` — קוד הכניסה לדשבורד (כרגע `1234` — להחליף).
- `HADAR_EMAIL` — ריק. למלא כדי לקבל התראות מייל על תורים/הצהרות + כדי שתזכורת מוצ"ש תגיע.
- `HADAR_PHONE` — לתזכורת השבועית דרך WhatsApp (אם Green API פעיל).
- `WA_PROVIDER` — `'manual'` או `'greenapi'`.

## טריגרים (Apps Script → Triggers)
- `dailyReminders` — תזכורת 24ש' ללקוחות. Time-driven, יומי, בוקר.
- `weeklyHoursReminder` — תזכורת מוצ"ש להדר לפתוח שעות. Time-driven, שבועי, שבת 21:00.

---

## מותג (לכל שינוי ויזואלי)
ספר המותג: `brand-kit.html`. צבע חתימה **טורקיז-אבן עמוק `#3D5A60`**, משני מרווה-אבן `#87A697`, אקסנט חרסית `#D8A793`, רקע פשתן `#F1EBE2`. פונטים: Frank Ruhl Libre + Heebo. קול: ישיר, בהיר, מעצים, לשון נקבה. מיצוב: עיסוי ממוקד תנועה ותפקוד (לא "פינוק").

## כללי עבודה
- עברית RTL בכל מקום (`dir="rtl"`, `lang="he"`). אסור `flex-direction: row-reverse` ב-RTL.
- אין תלויות / build — וניל JS בלבד.
- אחרי שינוי: בדיקת RTL + קישורים שבורים. שינוי frontend → commit+push. שינוי backend → גם Redeploy (ראו #1).
- בדיקת backend מהירה ללא דפדפן: `curl "<SHEETS_URL>?action=list&pin=<PIN>&callback=t"`.

## ✅ מצב נוכחי (יוני 2026)
- המערכת חיה ועובדת. דשבורד, הזמנות, שעות, לקוחות, תיק מטופל, הצהרות — פעילים.
- תוקנו: באג תאריך/שעה, ריפוד שעות + כפילויות, מלכודת מסך-לבן בהתחברות.
- נוסף: תזכורת מוצ"ש (`weeklyHoursReminder`), שירות עצמי ללקוחה (`update.html`).
- **ממתין**: החלפת PIN, מילוי HADAR_EMAIL, חיבור הטריגרים, ו-Redeploy אחרון של apps-script.js.
