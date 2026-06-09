# Lab: Username enumeration via subtly different responses

> **Category:** Authentication  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Username enumeration via subtly different responses](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-subtly-different-responses)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة تعداد أسماء المستخدمين (Username Enumeration) من خلال اختلاف **دقيق جداً** في رسائل الخطأ (خطأ إملائي بسيط: مسافة إضافية أو نقطة). ثم استخدام هجوم القوة الغاشمة (Brute-Force) لكسر كلمة المرور.

**المطلوب:** العثور على اسم مستخدم صالح وكلمة المرور الخاصة به، ثم تسجيل الدخول إلى صفحة الحساب.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم صفحة تسجيل الدخول

- افتح المختبر في المتصفح
- جرب تسجيل الدخول باسم مستخدم غير صحيح وكلمة مرور عشوائية

**النتيجة:** رسالة خطأ: "Invalid username or password."

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

### الخطوة 5: إعداد Grep - Extract لاستخراج رسالة الخطأ

- اذهب إلى **Settings** tab
- تحت **Grep - Extract**، اضغط **Add**
- في النافذة التي تظهر، انزل في الـ response حتى تجد رسالة الخطأ
- حدد النص **Invalid username or password.** (مع النقطة في النهاية)
- اضغط **OK**

### الخطوة 6: بدء الهجوم

- اضغط **Start attack**

### الخطوة 7: تحليل النتائج (البحث عن الاختلاف الطفيف)

- في نافذة النتائج، انظر إلى عمود استخراج الرسالة الذي أضفته
- ستجد أن **معظم الرسائل** تحتوي على `Invalid username or password.` (مع نقطة)
- ولكن **طلب واحد** يحتوي على `Invalid username or password ` (بدون نقطة، مع مسافة في النهاية)

| الرسالة المستخرجة | النتيجة |
|-------------------|---------|
| `Invalid username or password.` | ❌ اسم مستخدم غير صالح |
| `Invalid username or password ` | ✅ اسم مستخدم صالح (يوجد مسافة بدلاً من النقطة) |

### الخطوة 8: تسجيل الاسم الصالح

- سجل الاسم الذي ظهر في عمود **Payload** مع الاختلاف الطفيف

### الخطوة 9: إعادة تكوين Intruder (لكسر كلمة المرور)

- امسح علامات `§` بالضغط على **Clear §**
- ضع علامات `§` حول معامل `password`:

```
username=identified-user&password=§123§
```

### الخطوة 10: إعداد Payloads لكلمات المرور

- **Payload type:** Simple list
- الصق قائمة **Candidate passwords**

### الخطوة 11: بدء الهجوم الثاني

- اضغط **Start attack**

### الخطوة 12: تحليل النتائج (مرة أخرى)

- انظر إلى عمود **Status** (رمز الحالة)

| Status | النتيجة |
|--------|---------|
| 200 OK | ❌ فشل (كلمة مرور خاطئة) |
| 302 Found | ✅ نجاح (تم تسجيل الدخول) |

### الخطوة 13: تسجيل كلمة المرور الصحيحة

- سجل كلمة المرور التي ظهرت مع رمز `302`

### الخطوة 14: تسجيل الدخول

- استخدم الاسم وكلمة المرور اللذين وجدتهما لتسجيل الدخول

### الخطوة 15: حل المختبر

بعد تسجيل الدخول، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو الاختلاف الطفيف؟** | رسالة الخطأ لاسم المستخدم الصالح تحتوي على مسافة بدلاً من نقطة: `Invalid username or password ` (بدون نقطة) |
| **لماذا نجح هذا؟** | المطور أخطأ في كتابة رسالة الخطأ في حالة نجاح الاسم |
| **كيف اكتشفنا الاختلاف؟** | باستخدام ميزة **Grep - Extract** في Burp Intruder لاستخراج رسالة الخطأ ومقارنتها |
| **ما هي الفائدة؟** | حتى الاختلافات البسيطة (مسافة، نقطة، حالة حرف) يمكن استخدامها للتعداد |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يعطي رسالة خطأ مختلفة (ولو بشكل طفيف جداً) لاسم المستخدم الصالح مقابل غير الصالح.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/login.js
export default function handler(req, res) {
  const { username, password } = req.body;
  
  const user = getUserByUsername(username);
  
  // خطأ: رسائل مختلفة بفارق بسيط
  if (!user) {
    // ملاحظة: مسافة في النهاية أو نقطة مختلفة
    return res.status(200).json({ error: 'Invalid username or password ' });
  }
  
  if (user.password !== password) {
    return res.status(200).json({ error: 'Invalid username or password.' });
  }
  
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
  
  // التصحيح: رسالة خطأ متطابقة تماماً
  const errorMessage = 'Invalid username or password';
  
  const isAuthenticated = user && user.password === password;
  
  if (!isAuthenticated) {
    return res.status(200).json({ error: errorMessage });
  }
  
  // إضافة تأخير عشوائي لمنع timing attacks
  await new Promise(resolve => setTimeout(resolve, Math.random() * 100));
  
  req.session.userId = user.id;
  res.redirect('/my-account');
}
```

**الخلاصة:** 
1. استخدم رسالة خطأ **متطابقة تماماً** لجميع حالات الفشل
2. تحقق من عدم وجود مسافات إضافية أو نقاط أو اختلافات في حالة الأحرف
3. أضف تأخيراً عشوائياً لمنع هجمات التوقيت

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **رسالة خطأ متطابقة** | استخدم نفس السلسلة النصية تماماً في جميع حالات الفشل |
| **مراجعة النصوص** | تأكد من عدم وجود مسافات إضافية أو نقاط مختلفة |
| **تأخير عشوائي** | أضف تأخيراً عشوائياً قبل الرد |
| **قفل الحساب (Account Lockout)** | بعد عدد معين من المحاولات الفاشلة |
| **CAPTCHA** | لمنع الهجمات الآلية |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-subtly-different-responses)
- [Username Enumeration](https://portswigger.net/web-security/authentication/password-based#username-enumeration)
- [Burp Intruder Grep - Extract](https://portswigger.net/burp/documentation/desktop/tools/intruder/configure-attack/grep-extract)
