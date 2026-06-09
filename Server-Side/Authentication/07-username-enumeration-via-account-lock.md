# Lab: Username enumeration via account lock

> **Category:** Authentication  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Username enumeration via account lock](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-account-lock)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة تعداد أسماء المستخدمين (Username Enumeration) من خلال آلية **قفل الحساب (Account Locking)**. عند إرسال 5 محاولات فاشلة لاسم مستخدم صالح، يتم قفله وتظهر رسالة خطأ مختلفة: "You have made too many incorrect login attempts".

**المطلوب:** العثور على اسم مستخدم صالح، ثم كسر كلمة المرور بعد انتظار إعادة تعيين القفل.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم آلية قفل الحساب

- جرب تسجيل الدخول باسم مستخدم عشوائي عدة مرات
- **النتيجة:** بعد 5 محاولات فاشلة، يظهر قفل الحساب لاسم مستخدم صالح فقط

### الخطوة 2: إعداد Intruder (Cluster Bomb)

- أرسل طلب `POST /login` إلى **Intruder**
- اختر **Attack type:** Cluster bomb
- ضع علامات `§` حول `username`:

```
username=§invalid§&password=example
```

### الخطوة 3: إضافة Payload position فارغ

- حدد نهاية الطلب (بعد كلمة المرور)
- اضغط **Add §** لإضافة علامات فارغة:

```
username=§invalid§&password=example§§
```

### الخطوة 4: إعداد Payloads

**Payload position 1 (username):**
- **Payload type:** Simple list
- الصق قائمة **Candidate usernames**

**Payload position 2 (الفارغ):**
- **Payload type:** Null payloads
- **Generate:** `5` payloads

**ماذا يحدث؟**  
سيتم تكرار كل اسم مستخدم 5 مرات (5 محاولات فاشلة متتالية).

### الخطوة 5: بدء الهجوم

- اضغط **Start attack**

### الخطوة 6: تحليل النتائج

- في نافذة النتائج، انظر إلى عمود **Length** (طول الرد)
- ستجد أن **اسماً واحداً** له طول رد مختلف عن الباقي

| username | المحاولة | الطول | الرسالة |
|----------|---------|-------|---------|
| carlos | 1 | 2500 | Invalid username or password |
| carlos | 2 | 2500 | Invalid username or password |
| carlos | 3 | 2500 | Invalid username or password |
| carlos | 4 | 2500 | Invalid username or password |
| **carlos** | **5** | **2800** | **You have made too many incorrect login attempts** |

**الاستنتاج:** اسم المستخدم `carlos` صالح، وتم قفله بعد 5 محاولات.

### الخطوة 7: تسجيل الاسم الصالح

- سجل اسم المستخدم الذي ظهرت له رسالة القفل

### الخطوة 8: إعداد Intruder جديد (لكسر كلمة المرور)

- أرسل طلب `POST /login` جديد إلى **Intruder**
- اختر **Attack type:** Sniper
- ضع علامات `§` حول `password`:

```
username=carlos&password=§123§
```

### الخطوة 9: إعداد Grep - Extract لاستخراج رسالة الخطأ

- اذهب إلى **Settings** → **Grep - Extract** → **Add**
- حدد رسالة الخطأ من الـ response

### الخطوة 10: إعداد Payloads لكلمات المرور

- **Payload type:** Simple list
- الصق قائمة **Candidate passwords**

### الخطوة 11: بدء الهجوم الثاني

- اضغط **Start attack**

### الخطوة 12: تحليل النتائج

- انظر إلى عمود **Grep extract** (رسالة الخطأ)
- ستجد أن **طلب واحد** لا يحتوي على أي رسالة خطأ (أو به رسالة نجاح)

| password | status | error message |
|----------|--------|---------------|
| 123456 | 200 | Invalid username or password |
| password | 200 | Invalid username or password |
| **qwerty** | **302** | **(no error message)** |

### الخطوة 13: تسجيل كلمة المرور الصحيحة

- سجل كلمة المرور التي لم ترجع رسالة خطأ (رمز 302)

### الخطوة 14: انتظار إعادة تعيين القفل

- انتظر دقيقة واحدة (أو حتى ينتهي وقت القفل)
- **ملاحظة:** الحساب مقفل حالياً، يجب الانتظار قبل محاولة تسجيل الدخول

### الخطوة 15: تسجيل الدخول

- استخدم `carlos` وكلمة المرور التي وجدتها لتسجيل الدخول

### الخطوة 16: حل المختبر

بعد تسجيل الدخول، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي آلية قفل الحساب؟** | بعد 5 محاولات فاشلة، يتم قفل الحساب مؤقتاً |
| **لماذا نستخدم Cluster Bomb؟** | لتكرار كل اسم مستخدم 5 مرات (لتفعيل آلية القفل) |
| **كيف نعرف الاسم الصالح؟** | اسم المستخدم الصالح يظهر رسالة قفل بعد المحاولة الخامسة |
| **لماذا ننتظر بعد إيجاد كلمة المرور؟** | لأن الحساب مقفل، ويجب انتظار إعادة التعيين |

---

## 📊 شرح آلية القفل

| المحاولة | اسم صالح (carlos) | اسم غير صالح (invalid) |
|----------|-------------------|----------------------|
| 1 | Invalid username or password | Invalid username or password |
| 2 | Invalid username or password | Invalid username or password |
| 3 | Invalid username or password | Invalid username or password |
| 4 | Invalid username or password | Invalid username or password |
| 5 | **Account locked** ❌ | Invalid username or password |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

آلية قفل الحساب تكشف ما إذا كان اسم المستخدم صالحاً أم لا من خلال رسالة خطأ مختلفة.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/login.js
const failedAttempts = new Map(); // username -> count

export default function handler(req, res) {
  const { username, password } = req.body;
  
  const attempts = failedAttempts.get(username) || 0;
  
  // خطأ: رسالة خطأ مختلفة عند قفل الحساب
  if (attempts >= 5) {
    return res.status(200).json({ error: 'You have made too many incorrect login attempts' });
  }
  
  const user = getUserByUsername(username);
  
  if (!user || user.password !== password) {
    failedAttempts.set(username, attempts + 1);
    return res.status(200).json({ error: 'Invalid username or password' });
  }
  
  // تسجيل الدخول الناجح
  failedAttempts.delete(username);
  req.session.userId = user.id;
  res.redirect('/my-account');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/login.js
const failedAttempts = new Map(); // username -> { count, lockUntil }

export default function handler(req, res) {
  const { username, password } = req.body;
  
  const record = failedAttempts.get(username);
  
  // التصحيح 1: رسالة خطأ عامة حتى عند القفل
  const errorMessage = 'Invalid username or password';
  
  // التصحيح 2: التحقق من القفل دون الكشف عن السبب
  if (record && record.lockUntil > Date.now()) {
    // نفس رسالة الخطأ (لا تظهر أن الحساب مقفل)
    return res.status(200).json({ error: errorMessage });
  }
  
  const user = getUserByUsername(username);
  
  if (!user || user.password !== password) {
    const newCount = (record?.count || 0) + 1;
    
    // التصحيح 3: قفل الحساب بعد 5 محاولات، ولكن برسالة عامة
    if (newCount >= 5) {
      failedAttempts.set(username, {
        count: newCount,
        lockUntil: Date.now() + 15 * 60 * 1000 // 15 دقيقة
      });
    } else {
      failedAttempts.set(username, { count: newCount, lockUntil: 0 });
    }
    
    // التصحيح 4: نفس الرسالة لجميع الحالات
    return res.status(200).json({ error: errorMessage });
  }
  
  // تسجيل الدخول الناجح
  failedAttempts.delete(username);
  req.session.userId = user.id;
  res.redirect('/my-account');
}
```

**الخلاصة:** 
1. استخدم **نفس رسالة الخطأ** لجميع الحالات (حتى عند قفل الحساب)
2. لا تكشف سبب الفشل (مثل "account locked")
3. قم بتسجيل القفل في الخلفية (backend) دون إعلام المستخدم

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **رسالة خطأ عامة** | استخدم نفس الرسالة لجميع حالات الفشل (بما فيها القفل) |
| **قفل تدريجي** | زد وقت القفل مع كل محاولات فاشلة (تصاعدي) |
| **إعلام غير مباشر** | لا تخبر المستخدم أن حسابه مقفل |
| **تسجيل القفل** | سجل في الخلفية واتخذ إجراءات أخرى (تنبيه المسؤول) |
| **CAPTCHA بعد المحاولات** | بدلاً من القفل، استخدم CAPTCHA |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-account-lock)
- [Account Lockout](https://portswigger.net/web-security/authentication/password-based#account-lockout)
