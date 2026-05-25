# Lab: SameSite Lax bypass via method override

> **Category:** CSRF  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - SameSite Lax bypass via method override](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions/lab-samesite-lax-bypass-via-method-override)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة CSRF في وظيفة تغيير البريد الإلكتروني. الموقع يستخدم **SameSite=Lax** (افتراضياً) للكوكي، مما يمنع إرسال الكوكي في طلبات POST عبر المواقع. ولكننا سنستخدم **تجاوز طريقة الطلب (Method Override)** لتحويل الطلب إلى GET (الذي يسمح به SameSite=Lax في التنقلات العلوية) مع إضافة `_method=POST` لإقناع الخادم بمعاملته كـ POST.

**الحساب:** `wiener:peter`

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول وفهم إعدادات SameSite

- سجل الدخول بـ `wiener:peter`
- افتح **DevTools** (F12) → تبويب **Application** → **Cookies**
- ستجد أن كوكي `session` **ليس له SameSite محدد**

هذا يعني: المتصفح يطبق **SameSite=Lax** افتراضياً.

**ماذا يعني SameSite=Lax؟**

| نوع الطلب | هل يرسل الكوكي؟ |
|-----------|-----------------|
| POST من موقع خارجي | ❌ لا يرسل |
| GET مع top-level navigation (تغيير الصفحة) | ✅ يرسل |

### الخطوة 2: فحص طلب تغيير البريد

- اذهب إلى **My account** → **Update email**
- اعترض الطلب في Burp:

```http
POST /my-account/change-email HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE

email=wiener%40normal-user.net
```

**ملاحظة:** لا يوجد أي CSRF token في الطلب.

### الخطوة 3: تحويل الطلب إلى GET

- أرسل الطلب إلى Repeater
- اضغط بزر الماوس الأيمن → **Change request method**
- الطلب يتحول إلى:

```http
GET /my-account/change-email?email=wiener%40normal-user.net HTTP/1.1
```

**جرب إرساله:** الخادم يرفض لأن endpoint يقبل POST فقط.

### الخطوة 4: استخدام Method Override

أضف معامل `_method=POST` إلى الرابط:

```http
GET /my-account/change-email?email=hacked%40attacker.net&_method=POST HTTP/1.1
```

**النتيجة:** الطلب ينجح! ✅

**لماذا؟** لأن الخادم يدعم `_method` لتجاوز طريقة الطلب الفعلية.

### الخطوة 5: التحقق من تغير البريد

- في المتصفح، ارجع إلى **My account**
- لاحظ أن بريدك الإلكتروني تغير إلى `hacked@attacker.net`

### الخطوة 6: فهم آلية التجاوز

| الطلب | يرسل الكوكي؟ | السبب |
|-------|-------------|-------|
| POST عادي (من موقع خارجي) | ❌ لا | SameSite=Lax يمنع POST |
| GET مع `_method=POST` | ✅ نعم | GET + top-level navigation مسموح |
| الخادم يعامل GET كـ POST | ✅ | بسبب `_method` |

### الخطوة 7: إنشاء HTML للهجوم

نحتاج إلى **top-level navigation** (تغيير الصفحة) لكي يرسل المتصفح الكوكي:

```html
<script>
    document.location = "https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email?email=victim%40attacker.net&_method=POST";
</script>
```

### الخطوة 8: رفع الهجوم إلى Exploit Server

- اضغط **Go to exploit server**
- في حقل **Body**، الصق الـ HTML
- استبدل `YOUR-LAB-ID` بمعرف مختبرك
- اضغط **Store**

### الخطوة 9: تجربة الهجوم على نفسك

- اضغط **View exploit**
- ستتغير صفحتك (top-level navigation)
- ارجع إلى **My account** → لاحظ أن بريدك تغير ✅

### الخطوة 10: تسليم الهجوم للضحية

- غير البريد الإلكتروني في الـ HTML إلى قيمة مختلفة
- اضغط **Store**
- اضغط **Deliver to victim**

### الخطوة 11: حل المختبر

بعد بضع ثوانٍ، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو SameSite=Lax؟** | يرسل الكوكي فقط في طلبات GET مع top-level navigation |
| **ما هو top-level navigation؟** | تغيير عنوان الصفحة (مثل `document.location` أو النقر على رابط) |
| **كيف تجاوزناه؟** | استخدمنا GET (مسموح) مع `_method=POST` |
| **لماذا يدعم الخادم `_method`؟** | بعض الأطر (مثل Rails, Laravel) تدعم Method Override |
| **الفرق عن SameSite=Strict؟** | Strict يمنع حتى GET، Lax يسمح بـ GET |

---

## 📊 ملخص آلية التجاوز

| الطلب | يرسل الكوكي؟ | النتيجة |
|-------|-------------|---------|
| POST عادي (من موقع خارجي) | ❌ لا | يفشل |
| GET + `_method=POST` + top-level navigation | ✅ نعم | ينجح |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

1. استخدام SameSite=Lax فقط بدون حماية إضافية
2. دعم Method Override (`_method`) للطلبات الحساسة
3. عدم وجود CSRF Token

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// إعدادات الكوكي - لم يتم تحديد SameSite (افتراضي Lax)
res.setHeader('Set-Cookie', `session=${sessionId}; HttpOnly; Secure`);

// وظيفة تغيير البريد - تدعم Method Override
const method = req.query._method || req.method;

if (method === 'POST') {
  const { email } = req.method === 'POST' ? req.body : req.query;
  const { sessionId } = req.cookies;
  
  const user = getUserBySession(sessionId);
  updateUserEmail(user.id, email);
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// إعدادات الكوكي - SameSite=Strict
res.setHeader('Set-Cookie', `session=${sessionId}; HttpOnly; Secure; SameSite=Strict`);

// وظيفة تغيير البريد - لا تدعم Method Override
if (req.method !== 'POST') {
  return res.status(405).send('Method not allowed');
}

const { email, csrfToken } = req.body;
const { sessionId } = req.cookies;

if (!verifyCsrfToken(sessionId, csrfToken)) {
  return res.status(403).send('Invalid CSRF token');
}

const user = getUserBySession(sessionId);
updateUserEmail(user.id, email);
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
5. **رفض GET للطلبات الحساسة:** أي طلب يغير البيانات يجب أن يكون POST فقط.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions/lab-samesite-lax-bypass-via-method-override)
- [SameSite Cookies Explained](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions)
