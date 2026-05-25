# Lab: SameSite Lax bypass via method override

> **Category:** CSRF  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - SameSite Lax bypass via method override](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions/lab-samesite-lax-bypass-via-method-override)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة CSRF في وظيفة تغيير البريد الإلكتروني. الموقع يستخدم **SameSite=Lax** للكوكي، مما يمنع إرسال الكوكي في طلبات POST عبر المواقع. ولكننا سنستخدم **تجاوز طريقة الطلب (Method Override)** لتحويل الطلب إلى GET (الذي يسمح به SameSite=Lax في التنقلات العلوية).

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم آلية SameSite=Lax

- الـ session cookie ليس له SameSite محدد → المتصفح يطبق **SameSite=Lax** افتراضياً
- SameSite=Lax يرسل الكوكي فقط في:
  - طلبات GET مع **top-level navigation** (تغيير الصفحة)
  - وليس في طلبات POST عبر المواقع

### الخطوة 2: تجربة طلب POST العادي (سيفشل)

نحاول تغيير البريد عبر CSRF العادي:

```http
POST /my-account/change-email HTTP/1.1
Cookie: session=...
```

هذا الطلب **لن يرسل الكوكي** بسبب SameSite=Lax.

### الخطوة 3: استخدام Method Override

نلاحظ أن الموقع يدعم معامل `_method` لتجاوز طريقة الطلب:

```http
GET /my-account/change-email?email=hacked%40attacker.net&_method=POST HTTP/1.1
```

**النتيجة:** الطلب ينجح! لأن:
1. الطلب GET (يسمح بـ SameSite=Lax في top-level navigation)
2. `_method=POST` يخبر الخادم بمعاملته كـ POST

### الخطوة 4: إنشاء HTML للهجوم

نحتاج إلى **top-level navigation** (تغيير الصفحة) لكي يرسل المتصفح الكوكي:

```html
<script>
    document.location = "https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email?email=victim%40attacker.net&_method=POST";
</script>
```

### الخطوة 5: رفع الهجوم إلى Exploit Server

- اضغط **Go to exploit server**
- في حقل **Body**، الصق الـ HTML
- اضغط **Store**

### الخطوة 6: تجربة الهجوم على نفسك

- اضغط **View exploit**
- ستتغير صفحتك، ارجع إلى حسابك وتأكد من تغير البريد

### الخطوة 7: تسليم الهجوم للضحية

- غير البريد الإلكتروني إلى قيمة مختلفة
- اضغط **Store**
- اضغط **Deliver to victim**

### الخطوة 8: حل المختبر

بمجرد تسليم الهجوم، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو SameSite=Lax؟** | يرسل الكوكي فقط في طلبات GET مع top-level navigation |
| **كيف تجاوزناه؟** | استخدمنا GET + `_method=POST` لتغيير الطريقة |
| **لماذا نجح GET؟** | لأننا نستخدم `document.location` (top-level navigation) |
| **ما هو Method Override؟** | بعض الأطر تدعم `_method` لتجاوز طريقة HTTP الحقيقية |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

1. استخدام SameSite=Lax فقط بدون حماية إضافية
2. دعم Method Override (`_method`) للطلبات الحساسة

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// إعدادات الكوكي
res.setHeader('Set-Cookie', `session=${sessionId}; HttpOnly; Secure; SameSite=Lax`);

// وظيفة تغيير البريد - تدعم Method Override
export default function handler(req, res) {
  // خطأ: السماح بـ method override
  const method = req.query._method || req.method;
  
  if (method === 'POST') {
    const { email } = req.body;
    const { sessionId } = req.cookies;
    
    const user = getUserBySession(sessionId);
    updateUserEmail(user.id, email);
    res.send('Email updated');
  } else {
    res.status(405).send('Method not allowed');
  }
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// إعدادات الكوكي - SameSite=Strict أو Lax + CSRF Token
res.setHeader('Set-Cookie', `session=${sessionId}; HttpOnly; Secure; SameSite=Strict`);

// وظيفة تغيير البريد - لا تدعم Method Override
export default function handler(req, res) {
  // التصحيح: استخدام الطريقة الحقيقية فقط
  if (req.method !== 'POST') {
    return res.status(405).send('Method not allowed');
  }
  
  const { email, csrfToken } = req.body;
  const { sessionId } = req.cookies;
  
  // التحقق من CSRF Token
  if (!verifyCsrfToken(sessionId, csrfToken)) {
    return res.status(403).send('Invalid CSRF token');
  }
  
  const user = getUserBySession(sessionId);
  updateUserEmail(user.id, email);
  res.send('Email updated');
}
```

**الخلاصة:** 
1. لا تعتمد على SameSite فقط كحماية وحيدة
2. استخدم CSRF Tokens حتى مع SameSite=Strict
3. لا تدعم Method Override في endpoints حساسة

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **استخدام SameSite=Strict:** بدلاً من Lax لمنع إرسال الكوكي في أي طلب عبر المواقع.
2. **استخدام CSRF Tokens:** حتى مع SameSite، استخدم توكنات مربوطة بالجلسة.
3. **عدم دعم Method Override:** للطلبات الحساسة (تغيير البريد، كلمة المرور، إلخ).
4. **التحقق من Origin/Referer:** كطبقة دفاع إضافية.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions/lab-samesite-lax-bypass-via-method-override)
- [SameSite Cookies Explained](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions)
