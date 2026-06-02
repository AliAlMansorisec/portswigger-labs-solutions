# Lab: Authentication bypass via encryption oracle

> **Category:** Business Logic  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Authentication bypass via encryption oracle](https://portswigger.net/web-security/logic-flaws/lab-authentication-bypass-via-encryption-oracle)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة منطق العمل (Business Logic) حيث يتم الكشف عن **محور تشفير (encryption oracle)** للمستخدمين. يمكننا استخدام هذا المحور لتشفير قيمة نريدها (مثل `administrator:timestamp`) والحصول على كوكي `stay-logged-in` صالح للمدير، ثم استخدامه لتسجيل الدخول وحذف `carlos`.

**الحساب:** `wiener:peter`

**المطلوب:** الوصول إلى لوحة التحكم `/admin` وحذف `carlos`.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول وفهم آلية "Stay logged in"

- سجل الدخول بـ `wiener:peter` مع تفعيل خيار **Stay logged in**
- في Burp، ادرس الطلبات والردود
- لاحظ أن هناك كوكي `stay-logged-in` مشفرة

### الخطوة 2: اكتشاف Encryption Oracle

#### 2.1 محاولة تعليق ببريد إلكتروني غير صالح
- اكتب تعليقاً (comment) في أي مقال
- استخدم بريداً إلكترونياً غير صالح (مثل `invalid-email`)
- أرسل التعليق

#### 2.2 ملاحظة الرد
- الرد يحتوي على `Set-Cookie` مع كوكي `notification` مشفرة
- ثم يتم إعادة التوجيه إلى المقال

#### 2.3 فشل التعليق
- عند الوصول إلى المقال، تظهر رسالة الخطأ:
```
Invalid email address: invalid-email
```

**الاستنتاج:** 
- رسالة الخطأ تعكس البريد الإلكتروني الذي أدخلناه
- الرسالة تأتي من فك تشفير كوكي `notification`

### الخطوة 3: إعداد طلبات التشفير وفك التشفير

#### 3.1 إرسال الطلبات إلى Repeater
- أرسل `POST /post/comment` (طلب التعليق) إلى Repeater → سمّه **encrypt**
- أرسل `GET /post?postId=x` (طلب المقال مع كوكي notification) إلى Repeater → سمّه **decrypt**

#### 3.2 فهم العلاقة
- **encrypt:** يستخدم معامل `email` لتشفير أي بيانات → الناتج في `Set-Cookie: notification`
- **decrypt:** يستخدم كوكي `notification` لفك التشفير → الناتج في رسالة الخطأ

### الخطوة 4: فك تشفير كوكي stay-logged-in الحالية

- في طلب **decrypt**، استبدل كوكي `notification` بـ **كوكي `stay-logged-in`** الخاصة بك
- أرسل الطلب

**النتيجة:** تظهر رسالة الخطأ تحتوي على:
```
wiener:1598530205184
```

هذا يعني أن كوكي `stay-logged-in` بتنسيق: `username:timestamp`

### الخطوة 5: نسخ الطابع الزمني (timestamp)

- انسخ الطابع الزمني من الرد (مثال: `1598530205184`)

### الخطوة 6: تشفير قيمة جديدة للمدير

- في طلب **encrypt**، غيّر معامل `email` إلى:
```
administrator:1598530205184
```

- أرسل الطلب
- انسخ كوكي `notification` الجديدة من الرد

### الخطوة 7: فك تشفير الكوكي الجديدة

- في طلب **decrypt**، الصق كوكي `notification` الجديدة
- أرسل الطلب

**النتيجة:** رسالة الخطأ تحتوي على:
```
Invalid email address: administrator:1598530205184
```

**ملاحظة:** البادئة `Invalid email address: ` (23 حرفاً) تضاف تلقائياً إلى أي قيمة.

### الخطوة 8: إزالة البادئة من التشفير

#### 8.1 إرسال الكوكي إلى Burp Decoder
- انسخ كوكي `notification` المشفرة
- افتح **Burp Decoder**
- الصق الكوكي → **URL decode** → **Base64 decode**

#### 8.2 حذف البادئة (Hex)
- غيّر العرض إلى **Hex**
- أول 23 بايت هي البادئة `Invalid email address: `
- حدد أول 23 بايت → زر الماوس الأيمن → **Delete selected bytes**

#### 8.3 إعادة التشفير
- **Encode** → **Base64 encode** → **URL encode**
- انسخ النتيجة

#### 8.4 اختبار فك التشفير
- في طلب **decrypt**، الصق الكوكي الجديدة
- أرسل الطلب

**النتيجة المتوقعة:** يجب أن تظهر `administrator:timestamp` فقط (بدون البادئة)

### الخطوة 9: التعامل مع حظر التشفير (Block cipher padding)

إذا ظهر خطأ "input length must be a multiple of 16"، فهذا يعني أن الخوارزمية تستخدم **block cipher (AES)**.

**الحل:** أضف أحرفاً إضافية في البداية بحيث يصبح عدد البايتات التي ستحذفها مضاعفاً لـ 16.

#### 9.1 تجربة الإضافة
- في طلب **encrypt**، أضف 9 أحرف `x` في بداية القيمة:
```
xxxxxxxxxadministrator:1598530205184
```

#### 9.2 التشفير والحذف
- شفر القيمة الجديدة
- في Decoder، احذف `32` بايت (23 للبادئة + 9 للأحرف المضافة)
- أعد التشفير
- اختبر في طلب **decrypt**

### الخطوة 10: إنشاء كوكي stay-logged-in للمدير

- بعد الحصول على الكوكي النظيفة (بدون بادئة)، استخدمها كـ `stay-logged-in` كوكي

### الخطوة 11: تسجيل الدخول كمدير

- في Repeater، أرسل طلب `GET /`
- احذف كوكي `session` بالكامل
- استبدل كوكي `stay-logged-in` بالكوكي الجديدة التي صنعتها

```http
GET / HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: stay-logged-in=YOUR_CRAFTED_COOKIE
```

- أرسل الطلب

**النتيجة:** أصبحت مسجلاً كـ `administrator`! ✅

### الخطوة 12: الوصول إلى /admin وحذف carlos

- اذهب إلى `/admin`:

```http
GET /admin HTTP/1.1
Cookie: stay-logged-in=YOUR_CRAFTED_COOKIE
```

- ثم احذف `carlos`:

```http
GET /admin/delete?username=carlos HTTP/1.1
Cookie: stay-logged-in=YOUR_CRAFTED_COOKIE
```

### الخطوة 13: حل المختبر

بعد حذف `carlos`، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو Encryption Oracle؟** | نظام يسمح للمستخدم بتشفير أو فك تشفير البيانات حسب رغبته |
| **أين الثغرة؟** | يمكن استخدام محور التشفير لإنشاء كوكي `stay-logged-in` صالح لأي مستخدم (بما فيهم المدير) |
| **كيف نستغلها؟** | نفك تشفير كوكينا، نعدل البيانات، نعيد تشفيرها، ثم نستخدمها كـ `stay-logged-in` |
| **ما هو التنسيق؟** | `username:timestamp` |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الكشف عن **encryption oracle** للمستخدمين (يمكنهم تشفير وفك تشفير البيانات حسب رغبتهم).

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// صفحة التعليق - تقوم بتشفير البريد الإلكتروني في كوكي notification
export default function commentHandler(req, res) {
  const { email, comment } = req.body;
  
  // التحقق من صحة البريد الإلكتروني
  if (!isValidEmail(email)) {
    // خطأ: تشفير البريد الإلكتروني غير الصالح وإرساله في الكوكي
    const encrypted = encrypt(email);
    res.setHeader('Set-Cookie', `notification=${encrypted}`);
    return res.redirect(`/post?postId=${postId}`);
  }
  
  // حفظ التعليق...
}

// صفحة المقال - تقوم بفك تشفير كوكي notification
export default function postHandler(req, res) {
  const { notification } = req.cookies;
  
  if (notification) {
    const decrypted = decrypt(notification);
    // خطأ: يعرض البيانات المفككة في رسالة الخطأ
    res.send(`<p>Invalid email address: ${decrypted}</p>`);
  }
}

// المصادقة - تستخدم كوكي stay-logged-in المشفر
export default function authMiddleware(req, res, next) {
  const { 'stay-logged-in': encryptedCookie } = req.cookies;
  
  if (encryptedCookie) {
    const decrypted = decrypt(encryptedCookie);
    const [username, timestamp] = decrypted.split(':');
    
    if (isValidTimestamp(timestamp)) {
      req.user = getUserByUsername(username);
    }
  }
  
  next();
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// صفحة التعليق - لا تكشف عن encryption oracle
export default function commentHandler(req, res) {
  const { email, comment } = req.body;
  
  if (!isValidEmail(email)) {
    // التصحيح: لا تشفر البريد الإلكتروني في الكوكي
    // فقط اعرض رسالة خطأ وأعد التوجيه
    req.session.error = `Invalid email address: ${email}`;
    return res.redirect(`/post?postId=${postId}`);
  }
  
  // حفظ التعليق...
}

// صفحة المقال - لا تفك تشفير أي كوكي
export default function postHandler(req, res) {
  const error = req.session.error;
  if (error) {
    delete req.session.error;
    res.send(`<p>${escapeHtml(error)}</p>`);
  }
}

// المصادقة - استخدم session آمنة بدلاً من encrypt/decrypt
export default function authMiddleware(req, res, next) {
  const sessionId = req.cookies.session;
  
  if (sessionId) {
    const session = getSession(sessionId);
    if (session && !session.isExpired()) {
      req.user = session.user;
    }
  }
  
  next();
}

// إذا احتجت "stay logged in"، استخدم JWT مع توقيع (signature) آمن
import jwt from 'jsonwebtoken';

function generateStayLoggedInCookie(username) {
  const payload = { username, exp: Date.now() + 30 * 24 * 60 * 60 * 1000 };
  return jwt.sign(payload, process.env.JWT_SECRET);
}

function verifyStayLoggedInCookie(cookie) {
  try {
    const payload = jwt.verify(cookie, process.env.JWT_SECRET);
    return payload;
  } catch {
    return null;
  }
}
```

**الخلاصة:** 
1. لا تكشف أبداً عن encryption oracle للمستخدمين
2. استخدم session آمنة (على الخادم) بدلاً من الكوكيز المشفرة
3. إذا احتجت إلى كوكيز مشفرة، استخدم JWT مع توقيع آمن، وليس تشفيراً قابلاً للتلاعب

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **لا تكشف عن encryption oracle:** لا تسمح للمستخدمين بتشفير/فك تشفير بيانات حسب رغبتهم.
2. **استخدم Session على الخادم:** خزّن حالة تسجيل الدخول على الخادم، وليس في كوكيز قابلة للتلاعب.
3. **استخدم JWT مع توقيع:** بدلاً من تشفير البيانات، استخدم التواقيع الرقمية (signatures) لمنع التلاعب.
4. **لا تعكس بيانات مفككة للمستخدم:** قد تكشف معلومات حساسة.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/logic-flaws/lab-authentication-bypass-via-encryption-oracle)
- [Business Logic Vulnerabilities](https://portswigger.net/web-security/logic-flaws)
- [Encryption Oracle Attacks](https://portswigger.net/web-security/logic-flaws/examples#encryption-oracles)
