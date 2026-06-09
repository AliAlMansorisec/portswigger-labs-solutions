# Lab: Broken brute-force protection, IP block

> **Category:** Authentication  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Broken brute-force protection, IP block](https://portswigger.net/web-security/authentication/password-based/lab-broken-brute-force-protection-ip-block)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة منطقية (Logic Flaw) في آلية حماية القوة الغاشمة (Brute-Force Protection). الخادم يحظر الـ IP بعد 3 محاولات فاشلة، ولكن يمكن **إعادة تعيين العداد** (reset counter) بتسجيل الدخول بنجاح قبل الوصول إلى الحد الأقصى.

**المعطيات:**
- حسابنا: `wiener:peter`
- حساب الضحية: `carlos`
- قائمة كلمات المرور المرشحة (Candidate passwords)

**المطلوب:** العثور على كلمة مرور `carlos`، ثم تسجيل الدخول بحسابه.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم آلية الحماية

- جرب تسجيل الدخول بـ `carlos` وكلمة مرور خاطئة 3 مرات
- **النتيجة:** يتم حظر الـ IP مؤقتاً بعد 3 محاولات فاشلة

### الخطوة 2: اكتشاف الثغرة المنطقية

- جرب تسجيل الدخول بـ `wiener:peter` (حساب صحيح)
- لاحظ أن **عداد المحاولات الفاشلة يُعاد تعيينه** (reset)
- يمكننا استخدام هذا لإعادة التعيين بعد كل محاولتين فاشلتين لـ `carlos`

### الخطوة 3: بناء قائمة الهجوم (Pitchfork)

نحتاج إلى إرسال طلبات بالتسلسل التالي:

| الطلب | username | password | النتيجة المتوقعة |
|-------|----------|----------|------------------|
| 1 | wiener | peter | ✅ نجاح (يعيد تعيين العداد) |
| 2 | carlos | guess1 | ❌ فشل |
| 3 | carlos | guess2 | ❌ فشل |
| 4 | wiener | peter | ✅ نجاح (يعيد تعيين العداد) |
| 5 | carlos | guess3 | ❌ فشل |
| 6 | carlos | guess4 | ❌ فشل |
| 7 | wiener | peter | ✅ نجاح ... وهكذا |

### الخطوة 4: إعداد Intruder (Pitchfork)

- أرسل طلب `POST /login` إلى **Intruder**
- اختر **Attack type:** Pitchfork
- ضع علامات `§` حول `username` و `password`:

```
username=§wiener§&password=§peter§
```

### الخطوة 5: إعداد Payload position 1 (الأسماء)

**Payload position 1 (username):**
- **Payload type:** Simple list
- قم ببناء قائمة بالتسلسل التالي (كررها 100 مرة على الأقل):

```
wiener
carlos
carlos
wiener
carlos
carlos
wiener
carlos
carlos
...
```

**شرح:** كل 3 طلبات، نجاح واحد (`wiener`) يعيد تعيين العداد، ثم محاولتين لـ `carlos`.

### الخطوة 6: إعداد Payload position 2 (كلمات المرور)

**Payload position 2 (password):**
- **Payload type:** Simple list
- قم ببناء قائمة بالتسلسل التالي:

```
peter
password1
password2
peter
password3
password4
peter
password5
password6
...
```

**شرح:** كلمة مرور `wiener` (`peter`) تتزامن مع اسم `wiener`، وكلمات مرور التخمين تتزامن مع `carlos`.

### الخطوة 7: إعداد Resource Pool (مهم!)

- اذهب إلى **Resource Pool** tab
- أضف الهجوم إلى **resource pool** مع:
  - **Maximum concurrent requests:** `1`

**لماذا؟** لضمان إرسال الطلبات بالترتيب الصحيح (حتى لا يختلط التسلسل).

### الخطوة 8: بدء الهجوم

- اضغط **Start attack**

### الخطوة 9: تحليل النتائج

- في نافذة النتائج، ابحث عن طلبات `username=carlos` التي أعادت رمز **302 Found**
- قم بتصفية النتائج لإخفاء `200 OK`

| username | payload2 | status | النتيجة |
|----------|----------|--------|---------|
| carlos | password1 | 200 | ❌ فشل |
| carlos | password2 | 200 | ❌ فشل |
| carlos | password3 | 302 | ✅ نجاح |

### الخطوة 10: تسجيل كلمة المرور الصحيحة

- سجل كلمة المرور التي ظهرت مع رمز `302`

### الخطوة 11: تسجيل الدخول بحساب carlos

- استخدم `carlos` وكلمة المرور التي وجدتها لتسجيل الدخول

### الخطوة 12: حل المختبر

بعد تسجيل الدخول، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي الثغرة المنطقية؟** | عداد المحاولات الفاشلة يُعاد تعيينه بعد تسجيل دخول ناجح، حتى من حساب آخر |
| **كيف نستغلها؟** | نتناوب بين محاولات تخمين كلمة مرور `carlos` وتسجيل دخول ناجح بحساب `wiener` |
| **لماذا نستخدم Pitchfork؟** | لإقران اسم المستخدم مع كلمة المرور المناسبة في كل طلب |
| **لماذا `concurrent requests = 1`؟** | لضمان الترتيب الصحيح للطلبات ومنع اختلاط التسلسل |

---

## 📊 شرح التسلسل (نمط الهجوم)

| # | username | password | النتيجة | عداد المحاولات |
|---|----------|----------|---------|----------------|
| 1 | wiener | peter | ✅ نجاح | 0 (إعادة تعيين) |
| 2 | carlos | guess1 | ❌ فشل | 1 |
| 3 | carlos | guess2 | ❌ فشل | 2 |
| 4 | wiener | peter | ✅ نجاح | 0 (إعادة تعيين) |
| 5 | carlos | guess3 | ❌ فشل | 1 |
| 6 | carlos | guess4 | ❌ فشل | 2 |
| 7 | wiener | peter | ✅ نجاح | 0 (إعادة تعيين) |
| 8 | carlos | guess5 | ❌ فشل | 1 |
| 9 | carlos | guess6 | ❌ فشل | 2 |
| ... | ... | ... | ... | ... |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

عداد المحاولات الفاشلة (failed attempts counter) مرتبط بالـ IP فقط، ويمكن إعادة تعيينه بتسجيل دخول ناجح من أي حساب.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// middleware/rate-limit.js
const failedAttempts = new Map(); // IP -> count

export function rateLimit(req, res, next) {
  const ip = req.ip;
  const attempts = failedAttempts.get(ip) || 0;
  
  // خطأ: حظر بعد 3 محاولات فقط
  if (attempts >= 3) {
    return res.status(429).json({ error: 'Too many attempts' });
  }
  
  next();
}

// pages/api/login.js
export default function handler(req, res) {
  const { username, password } = req.body;
  const ip = req.ip;
  
  const user = getUserByUsername(username);
  
  if (!user || user.password !== password) {
    // خطأ: زيادة العداد للمحاولات الفاشلة فقط
    const attempts = failedAttempts.get(ip) || 0;
    failedAttempts.set(ip, attempts + 1);
    return res.status(200).json({ error: 'Invalid credentials' });
  }
  
  // خطأ: إعادة تعيين العداد عند النجاح (حتى من حسابات مختلفة)
  failedAttempts.delete(ip);
  
  req.session.userId = user.id;
  res.redirect('/my-account');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// middleware/rate-limit.js
const failedAttempts = new Map(); // username -> { count, firstAttempt }

export function rateLimit(req, res, next) {
  const { username } = req.body;
  
  if (!username) return next();
  
  const record = failedAttempts.get(username);
  
  if (record) {
    // التصحيح 1: حظر بعد 3 محاولات فاشلة لنفس المستخدم
    if (record.count >= 3) {
      // التصحيح 2: حظر لمدة 15 دقيقة
      if (Date.now() - record.firstAttempt < 15 * 60 * 1000) {
        return res.status(429).json({ error: 'Too many attempts for this user' });
      } else {
        // انتهى وقت الحظر
        failedAttempts.delete(username);
      }
    }
  }
  
  next();
}

// pages/api/login.js
export default function handler(req, res) {
  const { username, password } = req.body;
  
  const user = getUserByUsername(username);
  const record = failedAttempts.get(username);
  
  if (!user || user.password !== password) {
    // التصحيح 3: زيادة العداد للمستخدم المحدد
    if (!record) {
      failedAttempts.set(username, { count: 1, firstAttempt: Date.now() });
    } else {
      failedAttempts.set(username, { 
        count: record.count + 1, 
        firstAttempt: record.firstAttempt 
      });
    }
    return res.status(200).json({ error: 'Invalid credentials' });
  }
  
  // التصحيح 4: إعادة تعيين العداد للمستخدم نفسه فقط عند النجاح
  if (record) {
    failedAttempts.delete(username);
  }
  
  req.session.userId = user.id;
  res.redirect('/my-account');
}
```

**الخلاصة:** 
1. اربط عداد المحاولات بـ **اسم المستخدم**، وليس بالـ IP
2. لا تسمح بإعادة تعيين العداد إلا بتسجيل دخول ناجح **لنفس المستخدم**
3. أضف مهلة زمنية للحظر (مثلاً 15 دقيقة)

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **ربط العداد بالمستخدم** | تتبع المحاولات الفاشلة لكل اسم مستخدم، وليس لكل IP |
| **عدم إعادة التعيين بحسابات مختلفة** | نجاح مستخدم مختلف لا يعيد تعيين عداد مستخدم آخر |
| **مهلة زمنية للحظر** | حظر لمدة 15 دقيقة بعد 3 محاولات فاشلة |
| **تسجيل المحاولات** | سجل جميع محاولات تسجيل الدخول للكشف عن الأنماط |
| **استخدام CAPTCHA** | بعد عدد معين من المحاولات الفاشلة |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/authentication/password-based/lab-broken-brute-force-protection-ip-block)
- [Brute-Force Protection](https://portswigger.net/web-security/authentication/password-based#brute-force-attacks)
