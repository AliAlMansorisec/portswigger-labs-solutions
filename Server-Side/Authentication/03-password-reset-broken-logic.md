# Lab: Password reset broken logic

> **Category:** Authentication  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - Password reset broken logic](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-broken-logic)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة في منطق إعادة تعيين كلمة المرور (Password Reset). الخادم لا يتحقق من رمز إعادة التعيين (reset token) عند إرسال كلمة المرور الجديدة، مما يسمح لنا بتغيير كلمة مرور أي مستخدم (بما في ذلك `carlos`).

**المعطيات:**
- حسابنا: `wiener:peter`
- اسم ضحية: `carlos`

**المطلوب:** تغيير كلمة مرور `carlos`، ثم تسجيل الدخول بحسابه.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: طلب إعادة تعيين كلمة المرور لحسابنا

- اضغط على **Forgot your password?**
- أدخل اسم المستخدم الخاص بنا: `wiener`

### الخطوة 2: استلام رابط إعادة التعيين

- اضغط **Email client** لفتح البريد
- انسخ رابط إعادة التعيين (أو افتحه)

### الخطوة 3: تغيير كلمة المرور (لحسابنا)

- افتح الرابط
- غيّر كلمة المرور إلى أي قيمة
- اعترض طلب `POST /forgot-password` في Burp

الطلب يبدو هكذا:

```http
POST /forgot-password?temp-forgot-password-token=TOKEN_VALUE HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=wiener&new-password-1=123&new-password-2=123
```

### الخطوة 4: إرسال الطلب إلى Repeater

- زر الماوس الأيمن → **Send to Repeater**

### الخطوة 5: اختبار إزالة التوكن

- احذف قيمة `temp-forgot-password-token` من الرابط (URL) واتركها فارغة:

```http
POST /forgot-password?temp-forgot-password-token= HTTP/1.1
```

- أرسل الطلب

**النتيجة:** تم تغيير كلمة المرور بنجاح ✅

هذا يعني أن الخادم **لا يتحقق** من رمز إعادة التعيين!

### الخطوة 6: طلب إعادة تعيين جديدة (لحساب wiener مرة أخرى)

- في المتصفح، اطلب إعادة تعيين كلمة مرور جديدة لحساب `wiener`
- اعترض طلب `POST /forgot-password` مرة أخرى وأرسله إلى Repeater جديد

### الخطوة 7: تغيير username إلى carlos

- في الطلب الجديد، احذف قيمة `temp-forgot-password-token` من الرابط
- غيّر `username=wiener` إلى `username=carlos`
- حدد كلمة مرور جديدة (مثل `hacked`)

```http
POST /forgot-password?temp-forgot-password-token= HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=carlos&new-password-1=hacked&new-password-2=hacked
```

### الخطوة 8: إرسال الطلب

- اضغط **Send**

**النتيجة:** تم تغيير كلمة مرور `carlos` بنجاح ✅

### الخطوة 9: تسجيل الدخول بحساب carlos

- استخدم اسم المستخدم `carlos` وكلمة المرور الجديدة (`hacked`) لتسجيل الدخول

### الخطوة 10: حل المختبر

بعد تسجيل الدخول والوصول إلى صفحة `My account`، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **أين الثغرة؟** | الخادم لا يتحقق من رمز إعادة التعيين (reset token) عند إرسال كلمة المرور الجديدة |
| **كيف نستغلها؟** | نطلب رابط إعادة تعيين لحسابنا، لكننا نغير `username` إلى `carlos` ونتجاهل التوكن |
| **لماذا نحتاج طلب إعادة تعيين جديد؟** | لأن الرابط الأول كان صالحاً فقط لفترة زمنية محدودة |
| **ما هو المطلوب فقط؟** | اسم المستخدم المستهدف (`carlos`) دون الحاجة إلى أي رمز |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم لا يتحقق من صحة رمز إعادة التعيين (reset token) عند معالجة طلب تغيير كلمة المرور.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/forgot-password.js
export default function handler(req, res) {
  const { username, 'new-password-1': newPassword1, 'new-password-2': newPassword2 } = req.body;
  const { 'temp-forgot-password-token': token } = req.query;
  
  // خطأ: لا يتحقق من صحة التوكن
  // يتم تجاهل التوكن تماماً
  
  if (newPassword1 !== newPassword2) {
    return res.status(400).json({ error: 'Passwords do not match' });
  }
  
  const user = getUserByUsername(username);
  if (!user) {
    return res.status(404).json({ error: 'User not found' });
  }
  
  updateUserPassword(user.id, newPassword1);
  res.redirect('/login');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/forgot-password.js
import { getPasswordResetToken, deletePasswordResetToken } from './tokens';

export default function handler(req, res) {
  const { username, 'new-password-1': newPassword1, 'new-password-2': newPassword2 } = req.body;
  const { 'temp-forgot-password-token': token } = req.query;
  
  if (newPassword1 !== newPassword2) {
    return res.status(400).json({ error: 'Passwords do not match' });
  }
  
  // التصحيح 1: التحقق من وجود التوكن
  if (!token) {
    return res.status(400).json({ error: 'Reset token is required' });
  }
  
  // التصحيح 2: التحقق من صحة التوكن وارتباطه بالمستخدم
  const resetRequest = getPasswordResetToken(token);
  
  if (!resetRequest) {
    return res.status(400).json({ error: 'Invalid or expired reset token' });
  }
  
  // التصحيح 3: التحقق من أن التوكن يخص المستخدم المطلوب
  if (resetRequest.username !== username) {
    return res.status(400).json({ error: 'Invalid reset token for this user' });
  }
  
  // التصحيح 4: التحقق من صلاحية التوكن (زمنياً)
  if (resetRequest.expiresAt < Date.now()) {
    deletePasswordResetToken(token);
    return res.status(400).json({ error: 'Reset token has expired' });
  }
  
  const user = getUserByUsername(username);
  if (!user) {
    return res.status(404).json({ error: 'User not found' });
  }
  
  updateUserPassword(user.id, newPassword1);
  
  // التصحيح 5: حذف التوكن بعد الاستخدام
  deletePasswordResetToken(token);
  
  res.redirect('/login');
}
```

**الخلاصة:** 
1. تحقق من وجود رمز إعادة التعيين (reset token)
2. تحقق من صحة الرمز وارتباطه بالمستخدم
3. تحقق من صلاحية الرمز زمنياً (expiry)
4. احذف الرمز بعد الاستخدام (single-use)

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **تحقق من الرمز (Token validation)** | تأكد من وجود الرمز وصلاحيته |
| **اربط الرمز بالمستخدم** | لا تقبل رمزاً لمستخدم مختلف |
| **صلاحية زمنية محدودة** | اجعل الرمز ينتهي بعد فترة قصيرة (مثلاً 15 دقيقة) |
| **استخدام الرمز لمرة واحدة** | احذف الرمز بعد الاستخدام |
| **استخدم توكنات عشوائية قوية** | استخدم `crypto.randomBytes()` لتوليد الرمز |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-broken-logic)
- [Password Reset Vulnerabilities](https://portswigger.net/web-security/authentication/other-mechanisms#password-reset-vulnerabilities)
