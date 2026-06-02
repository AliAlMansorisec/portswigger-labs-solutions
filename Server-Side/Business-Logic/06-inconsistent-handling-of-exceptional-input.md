# Lab: Inconsistent handling of exceptional input

> **Category:** Business Logic  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Inconsistent handling of exceptional input](https://portswigger.net/web-security/logic-flaws/lab-inconsistent-handling-of-exceptional-input)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة منطق العمل (Business Logic) ناتجة عن **عدم تناسق معالجة المدخلات الطويلة جداً** (long email addresses). الخادم يقوم بقطع البريد الإلكتروني إلى 255 حرفاً، بينما نظام التأكيد (email client) يستخدم البريد الكامل. هذا يسمح لنا بالتسجيل بـ `@dontwannacry.com` (لنيل صلاحيات المدير) مع استلام رابط التأكيد على بريدنا الفعلي.

**المطلوب:** الوصول إلى لوحة التحكم `/admin` وحذف المستخدم `carlos`.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: اكتشاف مسار /admin

- استخدم **Content Discovery** في Burp
- الطريقة: **Target** > **Site map** > زر الماوس الأيمن على النطاق > **Engagement tools** > **Discover content**
- ستجد أن الأداة اكتشفت المسار `/admin`

### الخطوة 2: محاولة الوصول إلى /admin

- جرب زيارة `/admin`:

```
https://YOUR-LAB-ID.web-security-academy.net/admin
```

**النتيجة:** رسالة خطأ تقول أن الوصول مسموح فقط لمستخدمي **DontWannaCry**.

### الخطوة 3: الحصول على نطاق البريد الخاص بالمختبر

- اضغط على زر **Email client** في شريط المختبر
- ستجد نطاق البريد الخاص بك: `@YOUR-EMAIL-ID.web-security-academy.net`

### الخطوة 4: فهم الثغرة

الخادم يقطع البريد الإلكتروني إلى **255 حرفاً** فقط عند تخزينه في قاعدة البيانات. لكن **نظام التأكيد** (إرسال البريد) يستخدم البريد الإلكتروني الكامل (غير المقطوع).

إذا استطعنا أن نجعل البريد المقطوع ينتهي بـ `@dontwannacry.com`، سنحصل على صلاحيات المدير.

### الخطوة 5: حساب عدد الأحرف المطلوب

نريد:
- البريد الأصلي (المرسل إلى Email client): `very-long-string@dontwannacry.com.YOUR-EMAIL-ID.web-security-academy.net`
- البريد بعد التقطيع (255 حرفاً): `very-long-string@dontwannacry.com`

**الحساب:**

لنفترض أن `dontwannacry.com` هو `18` حرفاً (مع @).

نحتاج إلى `X` حرفاً في `very-long-string` بحيث:

```
X + 18 = 255
X = 237
```

لكن يجب أن نضيف النطاق الأصلي بعد `dontwannacry.com` حتى يصل البريد إلى بريدنا:

```
very-long-string@dontwannacry.com.OUR-EMAIL-ID.web-security-academy.net
```

الجزء `dontwannacry.com.OUR-EMAIL-ID.web-security-academy.net` له طول محدد، و `very-long-string` يملأ الباقي حتى يصبح البريد الكامل بعد القطع هو `very-long-string@dontwannacry.com`.

### الخطوة 6: إنشاء البريد الإلكتروني الطويل

- ابدأ ببناء البريد الإلكتروني:

```
[237 حرفاً عشوائياً]@dontwannacry.com.YOUR-EMAIL-ID.web-security-academy.net
```

مثال (للتوضيح فقط، استخدم طولاً مناسباً):
```
aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa@dontwannacry.com.YOUR-EMAIL-ID.web-security-academy.net
```

**ملاحظة:** 
- عد الأحرف بدقة بحيث:
  - البريد **الكامل** يصل إلى بريدك
  - البريد **المقطوع** (أول 255 حرفاً) ينتهي بـ `@dontwannacry.com`

### الخطوة 7: إنشاء الحساب الجديد

- اذهب إلى صفحة التسجيل (Register)
- استخدم البريد الإلكتروني الطويل الذي بنيته
- أكمل التسجيل

### الخطوة 8: تأكيد البريد الإلكتروني

- اذهب إلى **Email client**
- ستجد رسالة تأكيد
- اضغط على الرابط لإكمال التسجيل

### الخطوة 9: تسجيل الدخول

- سجل الدخول بالحساب الجديد
- اذهب إلى **My account** → لاحظ أن بريدك الإلكتروني المعروض ينتهي بـ `@dontwannacry.com` ✅

### الخطوة 10: الوصول إلى /admin

- الآن اذهب إلى `/admin` مرة أخرى:

```
https://YOUR-LAB-ID.web-security-academy.net/admin
```

**النتيجة:** أصبح لديك وصول إلى لوحة التحكم! ✅

### الخطوة 11: حذف المستخدم carlos

- اضغط على زر **Delete** بجانب `carlos`

### الخطوة 12: حل المختبر

بعد حذف `carlos`، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **أين الثغرة؟** | عدم تناسق معالجة المدخلات الطويلة: التقطيع عند التخزين (255 حرفاً) ولكن الإرسال للبريد يستخدم الطول الكامل |
| **لماذا هذا خطأ؟** | يجب أن تتم معالجة المدخلات بنفس الطريقة في جميع أجزاء التطبيق |
| **كيف نستغلها؟** | نرسل بريداً طويلاً بحيث بعد التقطيع يصبح في نطاق `@dontwannacry.com` |
| **ما هو inconsistent handling؟** | معالجة غير متناسقة لنفس البيانات في نقاط مختلفة من التطبيق |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يقوم بتقطيع البريد الإلكتروني إلى 255 حرفاً عند التخزين في قاعدة البيانات، ولكن نظام إرسال البريد يستخدم القيمة الأصلية غير المقطوعة.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/register.js - التسجيل
export default function registerHandler(req, res) {
  const { email, password } = req.body;
  
  // خطأ: تقطيع البريد الإلكتروني إلى 255 حرفاً
  const truncatedEmail = email.substring(0, 255);
  
  // تخزين البريد المقطوع في قاعدة البيانات
  createUser(truncatedEmail, password);
  
  // إرسال بريد التأكيد إلى البريد الأصلي (غير المقطوع)
  sendConfirmationEmail(email, token);
  
  res.json({ success: true });
}

// التحقق من صلاحية المدير
export default function adminHandler(req, res) {
  const { sessionId } = req.cookies;
  const user = getUserBySession(sessionId);
  
  // خطأ: التحقق من البريد المقطوع في قاعدة البيانات
  if (!user || !user.email.endsWith('@dontwannacry.com')) {
    return res.status(403).send('Access denied');
  }
  
  res.send(adminPanel);
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/register.js - التسجيل
export default function registerHandler(req, res) {
  let { email, password } = req.body;
  
  // التصحيح 1: التحقق من طول البريد قبل التخزين
  if (email.length > 255) {
    return res.status(400).json({ error: 'Email address too long' });
  }
  
  // التصحيح 2: التحقق من صحة البريد (بدون تقطيع)
  if (!isValidEmail(email)) {
    return res.status(400).json({ error: 'Invalid email address' });
  }
  
  // تخزين البريد الأصلي (غير مقطوع)
  createUser(email, password);
  
  // إرسال بريد التأكيد إلى نفس البريد المخزن
  sendConfirmationEmail(email, token);
  
  res.json({ success: true });
}

// التحقق من صلاحية المدير
export default function adminHandler(req, res) {
  const { sessionId } = req.cookies;
  const user = getUserBySession(sessionId);
  
  // التصحيح 3: التحقق من دور المستخدم (role) وليس من البريد
  if (!user || user.role !== 'admin') {
    return res.status(403).send('Access denied');
  }
  
  res.send(adminPanel);
}
```

**الخلاصة:** 
1. لا تقم بتقطيع المدخلات دون تحقق مسبق
2. استخدم نفس القيمة في جميع أجزاء التطبيق
3. لا تعتمد على البريد الإلكتروني لمنح صلاحيات إدارية

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **التحقق من الطول مسبقاً:** ارفض المدخلات الطويلة جداً بدلاً من تقطيعها.
2. **معالجة متناسقة:** استخدم نفس القيمة في جميع السياقات.
3. **لا تعتمد على البريد لمنح الصلاحيات:** استخدم أدوار (roles) مخزنة في قاعدة البيانات.
4. **تطبيع المدخلات (Normalization):** استخدم شكلاً موحداً للبيانات قبل معالجتها.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/logic-flaws/lab-inconsistent-handling-of-exceptional-input)
- [Business Logic Vulnerabilities](https://portswigger.net/web-security/logic-flaws)
