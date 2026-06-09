# Lab: Password brute-force via password change

> **Category:** Authentication  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Password brute-force via password change](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-brute-force-via-password-change)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة في وظيفة تغيير كلمة المرور (Password Change) حيث يمكننا تمييز كلمة المرور الصحيحة الحالية من خلال اختلاف رسائل الخطأ عند إدخال كلمتي مرور جديدتين غير متطابقتين.

**المعطيات:**
- حسابنا: `wiener:peter`
- حساب الضحية: `carlos`
- قائمة كلمات المرور المرشحة (Candidate passwords)

**المطلوب:** العثور على كلمة مرور `carlos`، ثم تسجيل الدخول بحسابه.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم وظيفة تغيير كلمة المرور

- سجل الدخول بـ `wiener:peter`
- اذهب إلى **My account** → **Change password**

الطلب يبدو هكذا:

```http
POST /my-account/change-password HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=wiener&current-password=peter&new-password-1=123&new-password-2=123
```

### الخطوة 2: تحليل سلوك رسائل الخطأ

| الحالة | current-password | new-password-1 | new-password-2 | رسالة الخطأ |
|--------|------------------|----------------|----------------|--------------|
| 1 | ✅ صحيحة | متطابقة | متطابقة | (نجاح - تغيير كلمة المرور) |
| 2 | ✅ صحيحة | مختلفة | مختلفة | `New passwords do not match` |
| 3 | ❌ خاطئة | متطابقة | متطابقة | `Current password is incorrect` (ويقفل الحساب) |
| 4 | ❌ خاطئة | مختلفة | مختلفة | `Current password is incorrect` |

**الاستنتاج:**  
إذا أدخلنا `new-password-1` و `new-password-2` **مختلفتين**، فعندما تكون `current-password` صحيحة، نحصل على رسالة `New passwords do not match`. وعندما تكون خاطئة، نحصل على `Current password is incorrect`.

هذا يسمح لنا بتعداد كلمات المرور!

### الخطوة 3: إعداد طلب الهجوم

- في Burp، اعترض طلب `POST /my-account/change-password`
- أرسله إلى **Intruder**

**الطلب المعدل:**

```http
POST /my-account/change-password HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=carlos&current-password=§incorrect§&new-password-1=123&new-password-2=abc
```

**ملاحظات مهمة:**
- `username=carlos` (بدلاً من `wiener`)
- `new-password-1=123` و `new-password-2=abc` (قيمتان **مختلفتان**)
- علامات `§` حول `current-password`

### الخطوة 4: إعداد Payloads

- **Payload type:** Simple list
- الصق قائمة **Candidate passwords**

### الخطوة 5: إعداد Grep - Extract

- في **Settings** → **Grep - Extract** → **Add**
- حدد النص `New passwords do not match`

### الخطوة 6: بدء الهجوم

- اضغط **Start attack**

### الخطوة 7: تحليل النتائج

- ابحث عن الطلب الذي يحتوي على `New passwords do not match`

| كلمة المرور | تحتوي على "New passwords do not match"? |
|-------------|------------------------------------------|
| 123456 | ❌ لا (Current password is incorrect) |
| password | ❌ لا |
| **qwerty** | **✅ نعم** |
| admin123 | ❌ لا |

### الخطوة 8: تسجيل كلمة المرور الصحيحة

- سجل كلمة المرور التي ظهرت مع الرسالة `New passwords do not match`

### الخطوة 9: تسجيل الدخول بحساب carlos

- سجل الخروج من حساب `wiener`
- سجل الدخول باستخدام:
  - **Username:** `carlos`
  - **Password:** كلمة المرور التي وجدتها

### الخطوة 10: حل المختبر

بعد تسجيل الدخول إلى صفحة `My account`، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **لماذا نستخدم new-password-1 و new-password-2 مختلفتين؟** | لمنع قفل الحساب (account lock) عند إدخال كلمة مرور خاطئة |
| **كيف نعرف أن كلمة المرور صحيحة؟** | عندما تظهر رسالة `New passwords do not match` (بدلاً من `Current password is incorrect`) |
| **ما هي الثغرة المنطقية؟** | رسائل الخطأ المختلفة تكشف ما إذا كانت كلمة المرور الحالية صحيحة أم لا |
| **لماذا لا يُقفل الحساب؟** | لأن الخادم يتحقق من تطابق كلمتي المرور الجديدتين أولاً |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

رسائل الخطأ المختلفة تكشف معلومات عن صحة كلمة المرور الحالية، مما يسمح بتعداد كلمات المرور.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/change-password.js
export default function handler(req, res) {
  const { username, currentPassword, newPassword1, newPassword2 } = req.body;
  
  const user = getUserByUsername(username);
  
  if (!user) {
    return res.status(404).json({ error: 'User not found' });
  }
  
  // خطأ: التحقق من تطابق كلمتي المرور الجديدتين أولاً
  if (newPassword1 !== newPassword2) {
    return res.status(200).json({ error: 'New passwords do not match' });
  }
  
  // خطأ: رسالة خطأ مختلفة لكلمة المرور الخاطئة
  if (user.password !== currentPassword) {
    return res.status(200).json({ error: 'Current password is incorrect' });
  }
  
  updateUserPassword(user.id, newPassword1);
  res.redirect('/my-account');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/change-password.js
export default function handler(req, res) {
  const { username, currentPassword, newPassword1, newPassword2 } = req.body;
  
  const user = getUserByUsername(username);
  
  if (!user) {
    return res.status(404).json({ error: 'User not found' });
  }
  
  // التصحيح 1: رسالة خطأ عامة لجميع حالات الفشل
  const errorMessage = 'Invalid current password or password mismatch';
  
  // التصحيح 2: التحقق من كلمة المرور الحالية أولاً
  const isCurrentPasswordValid = (user.password === currentPassword);
  
  // التصحيح 3: التحقق من تطابق كلمتي المرور الجديدتين
  const doNewPasswordsMatch = (newPassword1 === newPassword2);
  
  // التصحيح 4: نفس الرسالة لجميع حالات الفشل
  if (!isCurrentPasswordValid || !doNewPasswordsMatch) {
    return res.status(200).json({ error: errorMessage });
  }
  
  // التصحيح 5: إضافة تأخير عشوائي لمنع timing attacks
  await new Promise(resolve => setTimeout(resolve, Math.random() * 100));
  
  updateUserPassword(user.id, newPassword1);
  res.redirect('/my-account');
}
```

**الخلاصة:** 
1. استخدم رسالة خطأ عامة لجميع حالات الفشل (لا تفرق بين "كلمة مرور خاطئة" و "كلمتا المرور غير متطابقتين")
2. أعد ترتيب عمليات التحقق (تحقق من كلمة المرور الحالية أولاً)
3. أضف تأخيراً عشوائياً لمنع هجمات التوقيت

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **رسالة خطأ عامة** | استخدم نفس الرسالة لجميع حالات فشل تغيير كلمة المرور |
| **إعادة ترتيب التحقق** | تحقق من كلمة المرور الحالية أولاً |
| **تأخير عشوائي** | أضف تأخيراً عشوائياً قبل الرد |
| **قفل الحساب** | قفل الحساب بعد عدة محاولات فاشلة |
| **CAPTCHA** | بعد عدد معين من المحاولات |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-brute-force-via-password-change)
- [Password Change Vulnerabilities](https://portswigger.net/web-security/authentication/other-mechanisms#password-change-functionality)
