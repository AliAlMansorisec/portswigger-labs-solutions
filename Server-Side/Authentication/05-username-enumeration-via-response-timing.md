# Lab: Username enumeration via response timing

> **Category:** Authentication  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Username enumeration via response timing](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-response-timing)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة تعداد أسماء المستخدمين (Username Enumeration) من خلال **توقيت الاستجابة (Response Timing)**. الخادم يقوم بعمليات حسابية تستغرق وقتاً أطول عند وجود اسم مستخدم صالح (خاصة مع كلمة مرور طويلة). كما توجد حماية تعتمد على الـ IP، ولكن يمكن تجاوزها باستخدام هيدر `X-Forwarded-For`.

**المطلوب:** العثور على اسم مستخدم صالح وكلمة المرور الخاصة به، ثم تسجيل الدخول إلى صفحة الحساب.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم حماية الـ IP

- جرب تسجيل الدخول بعدة محاولات فاشلة
- **النتيجة:** سيتم حظر عنوان IP الخاص بك بعد عدة محاولات

### الخطوة 2: تجاوز حماية الـ IP باستخدام X-Forwarded-For

- أضف هيدر `X-Forwarded-For` إلى الطلب:

```http
POST /login HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
X-Forwarded-For: 1.2.3.4
Content-Type: application/x-www-form-urlencoded

username=invalid&password=123
```

**النتيجة:** يمكن تغيير قيمة الهيدر لكل طلب لتجنب الحظر.

### الخطوة 3: ملاحظة اختلاف التوقيت

- جرب تسجيل الدخول باسم مستخدم غير صالح (`invalid`) → وقت الاستجابة قصير
- جرب تسجيل الدخول باسم المستخدم الخاص بك (`wiener`) مع كلمة مرور طويلة (100 حرف) → وقت الاستجابة أطول

**السبب:** الخادم يقوم بعمليات حسابية (مثل hashing) فقط عند وجود اسم مستخدم صالح.

### الخطوة 4: إعداد Intruder (تعداد الأسماء)

- أرسل طلب `POST /login` إلى **Intruder**
- اختر **Attack type:** Pitchfork (لأننا نريد مصدرين للـ payloads)
- ضع علامات `§` حول:
  - قيمة `X-Forwarded-For`
  - قيمة `username`

```
X-Forwarded-For: §1.2.3.4§
username=§invalid§&password=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

**ملاحظة:** استخدم كلمة مرور طويلة (100 حرف) لتضخيم فرق التوقيت.

### الخطوة 5: إعداد Payloads

**Payload position 1 (`X-Forwarded-For`):**
- **Payload type:** Numbers
- **From:** `1`
- **To:** `100`
- **Step:** `1`
- **Max fraction digits:** `0`

**Payload position 2 (`username`):**
- **Payload type:** Simple list
- الصق قائمة **Candidate usernames**

### الخطوة 6: بدء الهجوم

- اضغط **Start attack**

### الخطوة 7: تحليل التوقيت

- في نافذة النتائج، اضغط على **Columns** واختر **Response received** و **Response completed**
- سترى عمودين يظهران وقت الاستجابة

**النتيجة:** ستجد **اسم مستخدم واحد** وقت استجابته أطول بكثير من الباقي.

| اسم المستخدم | وقت الاستجابة |
|--------------|---------------|
| invalid1 | 50ms |
| invalid2 | 51ms |
| **carlos** | **450ms** |
| invalid3 | 52ms |

### الخطوة 8: تسجيل الاسم الصالح

- سجل الاسم الذي ظهر مع وقت استجابة أطول (مثل `carlos`)

### الخطوة 9: إعادة تكوين Intruder (لكسر كلمة المرور)

- أرسل طلباً جديداً إلى **Intruder**
- اختر **Attack type:** Pitchfork
- ضع علامات `§` حول:
  - قيمة `X-Forwarded-For`
  - قيمة `password`

```
X-Forwarded-For: §1.2.3.4§
username=carlos&password=§123§
```

### الخطوة 10: إعداد Payloads (مرة أخرى)

**Payload position 1 (`X-Forwarded-For`):**
- **Payload type:** Numbers
- **From:** `1`
- **To:** `100`
- **Step:** `1`

**Payload position 2 (`password`):**
- **Payload type:** Simple list
- الصق قائمة **Candidate passwords**

### الخطوة 11: بدء الهجوم الثاني

- اضغط **Start attack**

### الخطوة 12: تحليل النتائج

- ابحث عن طلب برمز **302 Found**

| Status | النتيجة |
|--------|---------|
| 200 OK | ❌ فشل (كلمة مرور خاطئة) |
| 302 Found | ✅ نجاح (تم تسجيل الدخول) |

### الخطوة 13: تسجيل كلمة المرور الصحيحة

- سجل كلمة المرور التي ظهرت مع رمز `302`

### الخطوة 14: تسجيل الدخول

- استخدم الاسم وكلمة المرور اللذين وجدتهما لتسجيل الدخول

### الخطوة 15: حل المختبر

بعد تسجيل الدخول، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **لماذا هناك فرق في التوقيت؟** | الخادم يقوم بعمليات حسابية (مثل bcrypt hashing) فقط عند وجود اسم مستخدم صالح |
| **كيف نضخم الفرق؟** | باستخدام كلمة مرور طويلة جداً (100 حرف) تزيد وقت المعالجة |
| **كيف نتجاوز حماية الـ IP؟** | باستخدام هيدر `X-Forwarded-For` لتغيير الـ IP لكل طلب |
| **لماذا اخترنا Pitchfork؟** | لأننا نريد إقران كل اسم مستخدم بـ IP مختلف |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يقوم بعمليات حسابية مكلفة (مثل hashing) فقط عند وجود اسم مستخدم صالح، مما يخلق فرقاً في وقت الاستجابة.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/login.js
import bcrypt from 'bcrypt';

export default async function handler(req, res) {
  const { username, password } = req.body;
  
  const user = getUserByUsername(username);
  
  // خطأ: التحقق من وجود المستخدم أولاً
  if (!user) {
    // رد سريع (بدون hashing)
    return res.status(200).json({ error: 'Invalid username or password' });
  }
  
  // فقط إذا كان المستخدم موجوداً، نقوم بـ hashing (مكلف)
  const isValid = await bcrypt.compare(password, user.passwordHash);
  
  if (!isValid) {
    return res.status(200).json({ error: 'Invalid username or password' });
  }
  
  req.session.userId = user.id;
  res.redirect('/my-account');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/login.js
import bcrypt from 'bcrypt';

export default async function handler(req, res) {
  const { username, password } = req.body;
  
  // التصحيح 1: جلب المستخدم (قد يكون null)
  const user = getUserByUsername(username);
  
  // التصحيح 2: إنشاء hash افتراضي للمستخدمين غير الموجودين
  const defaultHash = '$2b$10$...'; // hash لقيمة عشوائية ثابتة
  
  // التصحيح 3: استخدام نفس الدالة لجميع الحالات
  const hashToCompare = user?.passwordHash || defaultHash;
  
  // التصحيح 4: تنفيذ hashing دائماً (حتى لو المستخدم غير موجود)
  const isValid = await bcrypt.compare(password, hashToCompare);
  
  // التصحيح 5: إضافة تأخير عشوائي إضافي
  await new Promise(resolve => setTimeout(resolve, Math.random() * 50));
  
  // التصحيح 6: التحقق من المصادقة بعد الـ hashing
  const isAuthenticated = user && isValid;
  
  if (!isAuthenticated) {
    return res.status(200).json({ error: 'Invalid username or password' });
  }
  
  req.session.userId = user.id;
  res.redirect('/my-account');
}
```

**الخلاصة:** 
1. قم بتنفيذ نفس العمليات الحسابية (hashing) لجميع المحاولات
2. استخدم hash افتراضي للمستخدمين غير الموجودين
3. أضف تأخيراً عشوائياً لتوحيد التوقيت

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **توحيد العمليات الحسابية** | قم بتنفيذ نفس العمليات (hashing) لجميع المحاولات |
| **استخدام hash افتراضي** | للمستخدمين غير الموجودين |
| **تأخير عشوائي** | أضف تأخيراً عشوائياً لتجنب التوقيت الدقيق |
| **حماية IP حقيقية** | لا تعتمد فقط على `X-Forwarded-For` |
| **استخدام CAPTCHA** | بعد عدد معين من المحاولات |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-response-timing)
- [Timing Attacks](https://portswigger.net/web-security/authentication/password-based#timing-attacks)
