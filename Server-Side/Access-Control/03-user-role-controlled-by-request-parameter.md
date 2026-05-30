# Lab: User role controlled by request parameter

> **Category:** Access Control  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - User role controlled by request parameter](https://portswigger.net/web-security/access-control/lab-user-role-controlled-by-request-parameter)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة التحكم في الصلاحيات (Access Control) حيث يتم تحديد دور المستخدم (admin أو مستخدم عادي) عبر **كوكي قابلة للتزوير**، ثم استخدامها للوصول إلى لوحة التحكم وحذف المستخدم `carlos`.

**الحساب:** `wiener:peter`

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: محاولة الوصول إلى /admin

- جرب زيارة `/admin` مباشرة:

```
https://YOUR-LAB-ID.web-security-academy.net/admin
```

**النتيجة:** سيتم رفض الوصول.

### الخطوة 2: تسجيل الدخول بحساب wiener

- سجل الدخول بـ `wiener:peter`

### الخطوة 3: تشغيل اعتراض الردود (Response Interception) في Burp

- في Burp → **Proxy** → **Options**
- تأكد من أن **Intercept responses** مفعل
- أو ببساطة: شغّل الـ Intercept وستتمكن من اعتراض الطلبات والردود معاً

### الخطوة 4: اعتراض رد تسجيل الدخول

- سجل الدخول عبر المتصفح
- في Burp، اعترض **الرد (Response)** لطلب تسجيل الدخول

### الخطوة 5: تعديل كوكي Admin

في رد الخادم، ستجد:

```
Set-Cookie: Admin=false; ...
```

**غيّر `false` إلى `true`**:

```
Set-Cookie: Admin=true; ...
```

### الخطوة 6: إعادة توجيه الطلب

- اضغط **Forward** لإرسال الرد المعدل إلى المتصفح

### الخطوة 7: الوصول إلى لوحة التحكم

- الآن اذهب إلى `/admin`:

```
https://YOUR-LAB-ID.web-security-academy.net/admin
```

- ستظهر صفحة الإدارة ✅

### الخطوة 8: حذف المستخدم carlos

- اضغط على زر **Delete** بجانب `carlos`

### الخطوة 9: حل المختبر

بعد حذف `carlos`، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **أين الثغرة؟** | الخادم يثق بقيمة كوكي `Admin` لتحديد صلاحيات المستخدم |
| **كيف نستغلها؟** | نعدل قيمة الكوكي من `false` إلى `true` في رد تسجيل الدخول |
| **لماذا هذا خطأ؟** | أي معلومات حساسة لتحديد الصلاحيات يجب أن تكون في الخادم، وليست في كوكي يمكن تزويرها |
| **كيف نصل إلى /admin؟** | بعد تعديل الكوكي، نزور الرابط مباشرة |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يحدد صلاحيات المستخدم بناءً على **كوكي قابلة للتعديل من قبل المستخدم** (`Admin=false/true`).

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// بعد تسجيل الدخول - خطأ: تخزين الصلاحية في كوكي
export default function loginHandler(req, res) {
  const { username, password } = req.body;
  const user = authenticate(username, password);
  
  // خطأ: الصلاحية في كوكي يمكن تزويرها
  res.setHeader('Set-Cookie', `Admin=${user.isAdmin}; HttpOnly`);
  res.redirect('/my-account');
}

// التحقق من الصلاحية
export default function adminHandler(req, res) {
  const { Admin } = req.cookies;
  
  // خطأ: يثق بقيمة الكوكي
  if (Admin !== 'true') {
    return res.status(403).send('Access denied');
  }
  
  // عرض لوحة التحكم
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// بعد تسجيل الدخول - تخزين الصلاحية في الجلسة على الخادم
import { getSession } from 'next-auth/react';

export default function loginHandler(req, res) {
  const { username, password } = req.body;
  const user = authenticate(username, password);
  
  // تخزين الصلاحية في الجلسة (آمن)
  req.session.userId = user.id;
  req.session.isAdmin = user.isAdmin;
  res.redirect('/my-account');
}

// التحقق من الصلاحية
export default async function adminHandler(req, res) {
  const session = await getSession({ req });
  
  // التحقق من الجلسة على الخادم
  if (!session || session.user.role !== 'admin') {
    return res.status(403).send('Access denied');
  }
  
  // عرض لوحة التحكم
}
```

**الخلاصة:** 
1. لا تخزن الصلاحيات في كوكي يمكن تزويرها
2. خزّن الصلاحيات في الجلسة على الخادم
3. تحقق من الصلاحيات من قاعدة البيانات أو الجلسة، وليس من مدخلات المستخدم

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **تخزين الصلاحيات في الخادم (Session):** لا تثق أبداً بمدخلات المستخدم لتحديد الصلاحيات.
2. **استخدام `HttpOnly` للكوكيز:** لكن هذا لا يمنع التزوير، الحل هو تخزين الصلاحية خلف الجلسة.
3. **التحقق من الصلاحية من قاعدة البيانات:** في كل طلب حساس، تأكد من صلاحية المستخدم الحقيقية.
4. **استخدام أدوار (Roles):** ميز بين المستخدم العادي والمدير.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/access-control/lab-user-role-controlled-by-request-parameter)
- [Access Control Cheat Sheet](https://portswigger.net/web-security/access-control)
