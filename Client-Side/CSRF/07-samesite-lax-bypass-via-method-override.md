# Lab: SameSite Lax bypass via method override

> **Category:** CSRF  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - SameSite Lax bypass via method override](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions/lab-samesite-lax-bypass-via-method-override)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة CSRF في وظيفة تغيير البريد الإلكتروني. الموقع يستخدم **SameSite=Lax** (افتراضياً) للكوكي، مما يمنع إرسال الكوكي في طلبات POST عبر المواقع. ولكننا سنستخدم **تجاوز طريقة الطلب (Method Override)** لتحويل الطلب إلى GET (الذي يسمح به SameSite=Lax في التنقلات العلوية) مع إضافة `_method=POST` لإقناع الخادم بمعاملته كـ POST.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول وفهم إعدادات SameSite

- سجل الدخول بـ `wiener:peter`
- افتح **DevTools** في المتصفح (F12) → تبويب **Application** → **Cookies**
- ستجد أن كوكي `session` **ليس له SameSite محدد**

هذا يعني: المتصفح يطبق **SameSite=Lax** افتراضياً.

**ماذا يعني SameSite=Lax؟**

| نوع الطلب | هل يرسل الكوكي؟ |
|-----------|-----------------|
| POST من موقع خارجي | ❌ لا يرسل |
| GET مع top-level navigation (تغيير الصفحة) | ✅ يرسل |

### الخطوة 2: فحص طلب تغيير البريد

- اذهب إلى **My account**
- غير بريدك الإلكتروني إلى أي قيمة
- في Burp → **Proxy > HTTP history**، ابحث عن الطلب:

```http
POST /my-account/change-email HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded

email=wiener%40normal-user.net
```

**ملاحظة:** لا يوجد أي CSRF token في الطلب.

### الخطوة 3: اختبار طلب POST العادي (سيفشل في CSRF)

لو حاولنا إرسال طلب POST من موقع خارجي، الكوكي لن يرسل بسبب SameSite=Lax.

### الخطوة 4: تحويل الطلب إلى GET

في Repeater:
- اضغط بزر الماوس الأيمن على الطلب
- اختر **Change request method**
- الطلب يتحول إلى:

```http
GET /my-account/change-email?email=wiener%40normal-user.net HTTP/1.1
```

**جرب إرساله:** الخادم يرفض لأن endpoint يقبل POST فقط.

### الخطوة 5: استخدام Method Override

أضف معامل `_method=POST` إلى الرابط:

```http
GET /my-account/change-email?email=hacked%40attacker.net&_method=POST HTTP/1.1
```

**النتيجة:** الطلب ينجح! ✅

**لماذا؟** لأن الخادم يدعم `_method` لتجاوز طريقة الطلب الفعلية.

### الخطوة 6: التحقق من تغير البريد

- في المتصفح، ارجع إلى **My account**
- لاحظ أن بريدك الإلكتروني تغير إلى `hacked@attacker.net`

### الخطوة 7: فهم آلية التجاوز

| الطلب | يرسل الكوكي؟ | السبب |
|-------|-------------|-------|
| POST عادي (من موقع خارجي) | ❌ لا | SameSite=Lax يمنع POST |
| GET مع `_method=POST` | ✅ نعم | GET + top-level navigation مسموح |

### الخطوة 8: إنشاء HTML للهجوم

نحتاج إلى **top-level navigation** (تغيير الصفحة) لكي يرسل المتصفح الكوكي:

```html
<script>
    document.location = "https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email?email=victim%40attacker.net&_method=POST";
</script>
```

### الخطوة 9: رفع الهجوم إلى Exploit Server

- اضغط **Go to exploit server**
- في حقل **Body**، الصق الـ HTML
- اضغط **Store**

### الخطوة 10: تجربة الهجوم على نفسك

- اضغط **View exploit**
- ستتغير صفحتك (top-level navigation)
- ارجع إلى **My account** وتأكد من تغير بريدك الإلكتروني

### الخطوة 11: تسليم الهجوم للضحية

- غير البريد الإلكتروني في الـ HTML إلى قيمة مختلفة (مثل `carlos@hacked.net`)
- اضغط **Store**
- اضغط **Deliver to victim**

### الخطوة 12: حل المختبر

بمجرد تسليم الهجوم، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو SameSite=Lax؟** | يرسل الكوكي فقط في طلبات GET مع top-level navigation |
| **ما هو top-level navigation؟** | تغيير عنوان الصفحة (مثل `document.location` أو النقر على رابط) |
| **كيف تجاوزناه؟** | استخدمنا GET (مسموح) مع `_method=POST` |
| **لماذا يدعم الخادم `_method`؟** | بعض الأطر (مثل Rails, Laravel) تدعم Method Override للتغلب على قيود HTML |
| **الفرق عن SameSite=Strict؟** | Strict يمنع حتى GET، Lax يسمح بـ GET |

---

## 📊 ملخص آلية التجاوز

| الخطوة | الطلب | يرسل الكوكي؟ |
|--------|-------|-------------|
| 1 | POST عادي (من موقع خارجي) | ❌ لا (يمنعه Lax) |
| 2 | GET + `_method=POST` + top-level navigation | ✅ نعم (مسموح) |
| 3 | الخادم يعامل GET كـ POST بسبب `_method` | ✅ يتم تغيير البريد |

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
export default function handler(req, res) {
  // خطأ: السماح بـ method override من query string
  const method = req.query._method || req.method;
  
  if (method === 'POST') {
    const { email } = req.method === 'POST' ? req.body : req.query;
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
  // التصحيح: رفض أي طلب ليس POST حقيقياً
  if (req.method !== 'POST') {
    return res.status(405).send('Method not allowed');
  }
  
  const { email, csrfToken } = req.body;
  const { sessionId } = req.cookies;
  
  // التحقق من CSRF Token
  if (!csrfToken || !verifyCsrfToken(sessionId, csrfToken)) {
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
4. استخدم `SameSite=Strict` بدلاً من Lax كلما أمكن

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
