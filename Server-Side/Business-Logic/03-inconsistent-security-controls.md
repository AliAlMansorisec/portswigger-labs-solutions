# Lab: Inconsistent security controls

> **Category:** Business Logic  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - Inconsistent security controls](https://portswigger.net/web-security/logic-flaws/lab-inconsistent-security-controls)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة منطق العمل (Business Logic) حيث يتم منح صلاحيات إدارية بناءً على **البريد الإلكتروني** فقط. سنقوم بإنشاء حساب، ثم تغيير بريدنا الإلكتروني إلى نطاق `@dontwannacry.com` للحصول على صلاحيات المدير.

**المطلوب:** الوصول إلى لوحة التحكم `/admin` وحذف المستخدم `carlos`.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: اكتشاف مسار /admin

- استخدم **Content Discovery** في Burp
- الطريقة: **Target** > **Site map** > زر الماوس الأيمن على النطاق > **Engagement tools** > **Discover content**
- ابدأ الجلسة
- بعد فترة، ستجد أن الأداة اكتشفت المسار `/admin`

### الخطوة 2: محاولة الوصول إلى /admin

- جرب زيارة `/admin`:

```
https://YOUR-LAB-ID.web-security-academy.net/admin
```

**النتيجة:** رسالة خطأ تقول أن الوصول مسموح فقط لمستخدمي **DontWannaCry**.

### الخطوة 3: تسجيل حساب جديد

- اذهب إلى صفحة التسجيل (Register)
- لاحظ أن هناك رسالة: DontWannaCry employees use their company email address
- سجل بحساب باستخدام بريد إلكتروني من المختبر نفسه

**كيف نحصل على بريد من المختبر؟**
- اضغط على زر **Email client**
- ستجد نطاق البريد مثل: `@YOUR-LAB-ID.web-security-academy.net`

سجل باستخدام:
```
test@YOUR-LAB-ID.web-security-academy.net
```

### الخطوة 4: تأكيد البريد الإلكتروني

- اذهب إلى **Email client**
- ستجد رسالة تأكيد
- اضغط على الرابط لإكمال التسجيل

### الخطوة 5: تسجيل الدخول بالحساب الجديد

- سجل الدخول بالحساب الجديد

### الخطوة 6: تغيير البريد الإلكتروني إلى نطاق DontWannaCry

- اذهب إلى **My account**
- غيّر بريدك الإلكتروني إلى عنوان في نطاق `@dontwannacry.com`:

```
anything@dontwannacry.com
```

### الخطوة 7: الوصول إلى /admin

- الآن اذهب إلى `/admin` مرة أخرى:

```
https://YOUR-LAB-ID.web-security-academy.net/admin
```

**النتيجة:** أصبح لديك وصول إلى لوحة التحكم! ✅

### الخطوة 8: حذف المستخدم carlos

- اضغط على زر **Delete** بجانب `carlos`

### الخطوة 9: حل المختبر

بعد حذف `carlos`، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **أين الثغرة؟** | صلاحيات المدير تُمنح بناءً على **البريد الإلكتروني** فقط، وليس على دور حقيقي (role) مخزن في قاعدة البيانات |
| **لماذا هذا خطأ؟** | البريد الإلكتروني يمكن تغييره بسهولة من قبل المستخدم |
| **كيف نستغلها؟** | نسجل حساباً عادياً، ثم نغير بريدنا إلى نطاق `@dontwannacry.com` |
| **لماذا نحتاج تسجيل حساب أولاً؟** | لتكون لدينا جلسة (session) صالحة، ثم نغير البريد لنحصل على الصلاحية |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يتحقق من وجود بريد إلكتروني في نطاق معين (`@dontwannacry.com`) لمنح صلاحيات إدارية، دون التحقق من أن المستخدم موظف فعلاً أو لديه دور مناسب.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/admin.js - التحقق من الصلاحية
export default function adminHandler(req, res) {
  const { sessionId } = req.cookies;
  const user = getUserBySession(sessionId);
  
  // خطأ: التحقق فقط من نطاق البريد الإلكتروني
  if (!user || !user.email.endsWith('@dontwannacry.com')) {
    return res.status(403).send('Access denied');
  }
  
  // عرض لوحة التحكم
  res.send(adminPanel);
}

// pages/api/change-email.js - تغيير البريد الإلكتروني
export default function changeEmailHandler(req, res) {
  const { sessionId } = req.cookies;
  const { email } = req.body;
  
  // خطأ: يسمح بتغيير البريد إلى أي نطاق
  const user = getUserBySession(sessionId);
  updateUserEmail(user.id, email);
  res.json({ success: true });
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/admin.js - التحقق من الصلاحية
export default function adminHandler(req, res) {
  const { sessionId } = req.cookies;
  const user = getUserBySession(sessionId);
  
  // التصحيح: التحقق من دور المستخدم (role) المخزن في قاعدة البيانات
  if (!user || user.role !== 'admin') {
    return res.status(403).send('Access denied');
  }
  
  // عرض لوحة التحكم
  res.send(adminPanel);
}

// pages/api/change-email.js - تغيير البريد الإلكتروني
export default function changeEmailHandler(req, res) {
  const { sessionId } = req.cookies;
  const { email } = req.body;
  
  const user = getUserBySession(sessionId);
  
  // التصحيح: التحقق من أن البريد الجديد من النطاق المسموح (إذا كان مطلوباً)
  // ولكن لا نستخدمه لمنح صلاحيات إدارية
  if (!isValidEmail(email)) {
    return res.status(400).json({ error: 'Invalid email' });
  }
  
  updateUserEmail(user.id, email);
  res.json({ success: true });
}

// عند تسجيل مستخدم جديد (register)
export default function registerHandler(req, res) {
  const { email, password } = req.body;
  
  // إنشاء مستخدم مع دور أساسي (وليست إدارة)
  const role = email.endsWith('@dontwannacry.com') ? 'employee' : 'user';
  
  createUser(email, password, role);
  res.json({ success: true });
}
```

**الخلاصة:** 
1. لا تعتمد على البريد الإلكتروني لمنح صلاحيات إدارية
2. استخدم **دور (role)** مخزناً في قاعدة البيانات
3. صلاحيات المدير يجب أن تُمنح فقط من خلال آلية آمنة (مثل قاعدة بيانات)

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **استخدام أدوار (Roles):** خزّن صلاحيات المستخدم في قاعدة البيانات، وليس في البريد الإلكتروني.
2. **لا تثق بالبريد الإلكتروني:** يمكن تغييره بسهولة من قبل المستخدم.
3. **التحقق من الصلاحية في كل طلب:** استخدم middleware للتحقق من دور المستخدم.
4. **تقييد تغيير البريد الإلكتروني:** امنع تغيير البريد إلى نطاقات معينة إذا كانت تمنح صلاحيات.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/logic-flaws/lab-inconsistent-security-controls)
- [Business Logic Vulnerabilities](https://portswigger.net/web-security/logic-flaws)
