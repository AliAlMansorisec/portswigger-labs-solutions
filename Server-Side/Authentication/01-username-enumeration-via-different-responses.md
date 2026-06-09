# Lab: Username enumeration via different responses

> **Category:** Authentication  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - Username enumeration via different responses](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-different-responses)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة تعداد أسماء المستخدمين (Username Enumeration) من خلال اختلاف رسائل الخطأ بين "Invalid username" و "Incorrect password". ثم استخدام هجوم القوة الغاشمة (Brute-Force) لكسر كلمة المرور.

**المطلوب:** العثور على اسم مستخدم صالح وكلمة المرور الخاصة به، ثم تسجيل الدخول إلى صفحة الحساب.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم صفحة تسجيل الدخول

- افتح المختبر في المتصفح
- جرب تسجيل الدخول باسم مستخدم غير صحيح وكلمة مرور عشوائية

**النتيجة:** رسالة خطأ: "Invalid username"

### الخطوة 2: اعتراض طلب تسجيل الدخول

- في Burp → **Proxy > HTTP history**
- ابحث عن طلب `POST /login`

```http
POST /login HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=invalid&password=123
```

### الخطوة 3: إرسال الطلب إلى Intruder (لتعداد الأسماء)

- زر الماوس الأيمن → **Send to Intruder**
- ضع علامات `§` حول معامل `username`:

```
username=§invalid§&password=123
```

### الخطوة 4: إعداد Payloads للأسماء

- **Payload type:** Simple list
- الصق قائمة **Candidate usernames**

### الخطوة 5: بدء الهجوم

- اضغط **Start attack**

### الخطوة 6: تحليل النتائج

- في نافذة النتائج، انظر إلى عمود **Length** (طول الرد)
- رتب النتائج حسب الطول (اضغط على العمود)
- ستجد **طلب واحد** بطول مختلف عن الباقي

| الطول | الرسالة | الاستنتاج |
|-------|---------|-----------|
| قصير | "Invalid username" | اسم المستخدم غير صالح ❌ |
| طويل | "Incorrect password" | اسم المستخدم صالح ✅ |

**مثال:** الرد الطويل يحتوي على `Incorrect password` بدلاً من `Invalid username`

### الخطوة 7: تسجيل الاسم الصالح

- سجل الاسم الذي ظهر في عمود **Payload** (مثل `carlos` أو `admin`)

### الخطوة 8: إعادة تكوين Intruder (لكسر كلمة المرور)

- امسح علامات `§` بالضغط على **Clear §**
- ضع علامات `§` حول معامل `password`:

```
username=carlos&password=§123§
```

### الخطوة 9: إعداد Payloads لكلمات المرور

- **Payload type:** Simple list
- الصق قائمة **Candidate passwords**

### الخطوة 10: بدء الهجوم الثاني

- اضغط **Start attack**

### الخطوة 11: تحليل النتائج (مرة أخرى)

- انظر إلى عمود **Status** (رمز الحالة)

| Status | النتيجة |
|--------|---------|
| 200 OK | ❌ فشل (كلمة مرور خاطئة) |
| 302 Found | ✅ نجاح (تم تسجيل الدخول) |

### الخطوة 12: تسجيل كلمة المرور الصحيحة

- سجل كلمة المرور التي ظهرت مع رمز `302`

### الخطوة 13: تسجيل الدخول

- استخدم الاسم وكلمة المرور اللذين وجدتهما لتسجيل الدخول

### الخطوة 14: حل المختبر

بعد تسجيل الدخول، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي Username Enumeration؟** | اكتشاف أسماء المستخدمين الصالحة من خلال اختلاف ردود الخادم |
| **لماذا نجح هذا؟** | الخادم يعطي رسالة مختلفة لاسم مستخدم غير صالح (`Invalid username`) مقابل كلمة مرور خاطئة (`Incorrect password`) |
| **كيف اكتشفنا الاسم الصالح؟** | من خلال اختلاف طول الرد (Response Length) |
| **كيف اكتشفنا كلمة المرور؟** | من خلال رمز الحالة 302 (إعادة توجيه) بدلاً من 200 |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يعطي رسائل خطأ مختلفة لحالات فشل المصادقة المختلفة، مما يسمح للمهاجم بمعرفة ما إذا كان اسم المستخدم صالحاً أم لا.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/login.js
export default function handler(req, res) {
  const { username, password } = req.body;
  
  const user = getUserByUsername(username);
  
  // خطأ: رسائل خطأ مختلفة
  if (!user) {
    return res.status(200).json({ error: 'Invalid username' });
  }
  
  if (user.password !== password) {
    return res.status(200).json({ error: 'Incorrect password' });
  }
  
  // تسجيل الدخول الناجح
  req.session.userId = user.id;
  res.redirect('/my-account');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/login.js
export default function handler(req, res) {
  const { username, password } = req.body;
  
  const user = getUserByUsername(username);
  
  // التصحيح: رسالة خطأ عامة (لا تفرق بين الحالتين)
  const isAuthenticated = user && user.password === password;
  
  if (!isAuthenticated) {
    // نفس الرسالة لكل من الاسم الخاطئ وكلمة المرور الخاطئة
    return res.status(200).json({ error: 'Invalid username or password' });
  }
  
  // إضافة تأخير عشوائي لمنع التوقيت (timing attacks)
  await new Promise(resolve => setTimeout(resolve, Math.random() * 100));
  
  req.session.userId = user.id;
  res.redirect('/my-account');
}
```

**الخلاصة:** 
1. استخدم رسالة خطأ عامة: `Invalid username or password`
2. لا تفرق بين الاسم الخاطئ وكلمة المرور الخاطئة
3. أضف تأخيراً عشوائياً لمنع هجمات التوقيت (timing attacks)

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **رسالة خطأ عامة** | استخدم نفس الرسالة لجميع حالات فشل المصادقة |
| **تأخير عشوائي** | أضف تأخيراً عشوائياً قبل الرد لمنع هجمات التوقيت |
| **قفل الحساب (Account Lockout)** | بعد عدد معين من المحاولات الفاشلة |
| **CAPTCHA بعد محاولات متعددة** | لمنع الهجمات الآلية |
| **مراقبة وتسجيل المحاولات** | لتحديد الأنماط المشبوهة |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-different-responses)
- [Username Enumeration](https://portswigger.net/web-security/authentication/password-based#username-enumeration)
