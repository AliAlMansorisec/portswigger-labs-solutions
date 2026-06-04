# Lab: Bypassing access controls using email address parsing discrepancies

> **Category:** Business Logic  
> **Difficulty:** EXPERT  
> **Lab Link:** [PortSwigger Lab - Bypassing access controls using email address parsing discrepancies](https://portswigger.net/web-security/logic-flaws/lab-access-control-bypass-via-email-address-parsing-discrepancies)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة منطق العمل (Business Logic) ناتجة عن **اختلاف في تفسير (parsing) عناوين البريد الإلكتروني** بين نظام التسجيل (الذي يتحقق من النطاق `@ginandjuice.shop`) ونظام إرسال البريد الإلكتروني. باستخدام ترميز **UTF-7**، يمكننا تسجيل بريد إلكتروني يبدو كأنه من النطاق المسموح، ولكن البريد الفعلي يصل إلى خادمنا.

**المطلوب:** تسجيل حساب يمنح صلاحيات إدارية (أو الوصول إلى `/admin`) وحذف `carlos`.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم قيود التسجيل

- افتح المختبر
- اضغط **Register**
- جرب التسجيل بـ: `foo@exploit-server.net`

**النتيجة:** يتم رفض التسجيل، خطأ: "email domain must be ginandjuice.shop"

### الخطوة 2: تجربة ترميز Q (iso-8859-1)

- جرب التسجيل بـ:

```
=?iso-8859-1?q?=61=62=63?=foo@ginandjuice.shop
```

هذا يمثل: `abcfoo@ginandjuice.shop` (جزء `abc` مشفر بـ Q encoding)

**النتيجة:** يتم رفض التسجيل، خطأ: "Registration blocked for security reasons"

### الخطوة 3: تجربة ترميز UTF-8

- جرب التسجيل بـ:

```
=?utf-8?q?=61=62=63?=foo@ginandjuice.shop
```

**النتيجة:** نفس الخطأ، تم اكتشاف محاولة التلاعب.

### الخطوة 4: تجربة ترميز UTF-7 (الذي يعمل)

- جرب التسجيل بـ:

```
=?utf-7?q?&AGEAYgBj-?=foo@ginandjuice.shop
```

**النتيجة:** **لا يوجد خطأ** ✅ تم قبول الطلب!

**الاستنتاج:** الخادم لا يتعرف على ترميز UTF-7 كتهديد أمني.

### الخطوة 5: فهم ترميز UTF-7

UTF-7 يستخدم لترميز الأحرف غير ASCII. الرمز `@` يمكن ترميزه كـ `&AEA-` والمسافة (` `) كـ `&ACA-`.

**الهدف:** نريد أن يبدو البريد كـ `attacker@...@ginandjuice.shop` حيث:
- التحقق من النطاق يرى `@ginandjuice.shop` في النهاية → يجتاز الفلتر
- نظام البريد يرسل إلى `attacker@EXPLOIT-SERVER`

### الخطوة 6: إنشاء البريد المشفر بتنسيق UTF-7

نريد أن يصل البريد إلى خادمنا، لذلك نستخدم نطاق خادم الاستغلال (exploit server).

صيغة البريد:

```
=?utf-7?q?attacker&AEA-[YOUR-EXPLOIT-SERVER_ID]&ACA-?=@ginandjuice.shop
```

**مثال (للتوضيح):**
إذا كان معرف خادم الاستغلال الخاص بك هو `exploit-abc123`:

```
=?utf-7?q?attacker&AEA-exploit-abc123&ACA-?=@ginandjuice.shop
```

**شرح التشفير:**
- `attacker` → نص عادي
- `&AEA-` → يمثل `@`
- `YOUR-EXPLOIT-SERVER_ID` → معرف خادمك
- `&ACA-` → يمثل مسافة
- `?=` → نهاية الترميز
- `@ginandjuice.shop` → النطاق المطلوب للتحقق

عند تفسير هذا البريد:
- **نظام التسجيل** يرى النطاق `@ginandjuice.shop` ← يجتاز التحقق ✅
- **نظام البريد** يفك الترميز ← يرسل إلى `attacker@EXPLOIT-SERVER`

### الخطوة 7: التسجيل بالبريد المشفر

- سجل حساباً جديداً باستخدام البريد المشفر أعلاه
- أكمل بيانات التسجيل (أي كلمة مرور)

### الخطوة 8: تأكيد البريد الإلكتروني

- اضغط على **Email client** في شريط المختبر
- ستجد رسالة تأكيد من المختبر
- اضغط على الرابط لإكمال التسجيل

### الخطوة 9: تسجيل الدخول

- سجل الدخول بالحساب الجديد

### الخطوة 10: الوصول إلى /admin

- بعد تسجيل الدخول، اذهب إلى **Admin panel**
- ستظهر لوحة التحكم ✅

### الخطوة 11: حذف المستخدم carlos

- اضغط على زر **Delete** بجانب `carlos`

### الخطوة 12: حل المختبر

بعد حذف `carlos`، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو الـ parsing discrepancy؟** | اختلاف في كيفية تفسير عنوان البريد الإلكتروني بين نظام التحقق (validation) ونظام إرسال البريد (mailer) |
| **لماذا ترميز UTF-7؟** | الخادم لا يتعرف على UTF-7 كتهديد، بينما يمنع UTF-8 و Q encoding |
| **كيف يعمل التجاوز؟** | نستخدم UTF-7 لتضمين `@` ومسافة في اسم المستخدم، فينكسر البريد إلى جزئين |
| **ماذا يرى نظام التحقق؟** | `...@ginandjuice.shop` في النهاية → يجتاز الفلتر |
| **ماذا يرى نظام البريد؟** | `attacker@EXPLOIT-SERVER` → يرسل التأكيد إلينا |

---

## 📊 شرح ترميز UTF-7 المستخدم

| الحرف | ترميز UTF-7 |
|-------|-------------|
| `@` | `&AEA-` |
| مسافة (` `) | `&ACA-` |

**مثال للتفكيك:**

```
attacker@exploit-server
```

بعد ترميز `@` والمسافة:

```
attacker&AEA-exploit-server&ACA-
```

ثم نضيف `?=@ginandjuice.shop` في النهاية:

```
=?utf-7?q?attacker&AEA-exploit-server&ACA-?=@ginandjuice.shop
```

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

1. عدم دعم ترميز UTF-7 في نظام التحقق من البريد (بينما يدعمه نظام البريد)
2. الاعتماد على التحقق من النطاق في نهاية البريد فقط

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/register.js - التحقق من البريد الإلكتروني
export default function registerHandler(req, res) {
  const { email, password } = req.body;
  
  // خطأ: يتحقق فقط من أن البريد ينتهي بـ @ginandjuice.shop
  if (!email.endsWith('@ginandjuice.shop')) {
    return res.status(400).json({ error: 'Invalid email domain' });
  }
  
  // منع بعض الترميزات المعروفة
  if (email.includes('=?utf-8?') || email.includes('=?iso-8859-1?')) {
    return res.status(400).json({ error: 'Registration blocked for security reasons' });
  }
  
  // خطأ: لا يمنع UTF-7
  // لا يتحقق من البريد بعد فك الترميز
  
  // إرسال بريد التأكيد إلى البريد الأصلي (غير المفكك)
  sendConfirmationEmail(email, token);
  createUser(email, password);
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/register.js - التحقق من البريد الإلكتروني
import * as EmailValidator from 'email-validator';

// تطبيع (normalize) البريد الإلكتروني قبل التحقق
function normalizeEmail(email) {
  // 1. فك ترميز UTF-7 إذا وجد
  let normalized = email.replace(/=\?utf-7\?q\?([^?]+)\?=/g, (match, encoded) => {
    return decodeUTF7(encoded);
  });
  
  // 2. فك ترميز Q-encoding
  normalized = normalized.replace(/=\?([^?]+)\?q\?([^?]+)\?=/gi, (match, charset, encoded) => {
    return decodeQEncoding(encoded);
  });
  
  // 3. إزالة أي فواصل أو مسافات غير مرغوب فيها
  normalized = normalized.replace(/\s/g, '');
  
  return normalized;
}

export default function registerHandler(req, res) {
  let { email, password } = req.body;
  
  // التصحيح 1: تطبيع البريد الإلكتروني أولاً
  const normalizedEmail = normalizeEmail(email);
  
  // التصحيح 2: التحقق من النطاق بعد التطبيع
  if (!normalizedEmail.endsWith('@ginandjuice.shop')) {
    return res.status(400).json({ error: 'Invalid email domain' });
  }
  
  // التصحيح 3: استخدام مكتبة موثوقة للتحقق من صحة البريد
  if (!EmailValidator.validate(normalizedEmail)) {
    return res.status(400).json({ error: 'Invalid email format' });
  }
  
  // التصحيح 4: منع جميع أنواع الترميز
  if (email.includes('=?') && email.includes('?=')) {
    // تسجيل المحاولة المشبوهة
    logSuspiciousAttempt(req.ip, email);
    return res.status(400).json({ error: 'Invalid email format' });
  }
  
  // إرسال بريد التأكيد إلى البريد المُطبع
  sendConfirmationEmail(normalizedEmail, token);
  createUser(normalizedEmail, password);
}

// دالة لفك ترميز UTF-7
function decodeUTF7(encoded) {
  // تنفيذ فك ترميز UTF-7
  // ملاحظة: يجب استخدام مكتبة آمنة لهذا الغرض
  return decoded;
}
```

**الخلاصة:** 
1. طبّع (normalize) البريد الإلكتروني قبل التحقق منه
2. امنع جميع أنواع الترميز، بما فيها UTF-7
3. استخدم مكتبة موثوقة للتحقق من البريد الإلكتروني
4. أرسل بريد التأكيد إلى البريد المُطبع (normalized) وليس الأصلي

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **تطبيع البريد الإلكتروني:** فك جميع أنواع الترميز قبل التحقق.
2. **منع الترميزات غير الآمنة:** قائمة بيضاء بالحروف المسموحة فقط (ASCII printable).
3. **لا تعتمد على النطاق فقط:** استخدم قائمة بيضاء بالنطاقات المسموحة بعد التطبيع.
4. **استخدام مكتبة قوية:** مثل `email-validator` التي تتعامل مع حالات الترميز المختلفة.

---

## 📊 ملخص لابات Business Logic

| # | اللاب | نوع الثغرة | طريقة الاستغلال |
|---|-------|-----------|----------------|
| 01 | Excessive trust in client-side controls | الثقة المفرطة بمدخلات العميل | تغيير معامل `price` في طلب `POST /cart` |
| 02 | High-level logic vulnerability | قبول كميات سالبة | استخدام `quantity` سالبة لجعل السعر الإجمالي سالباً |
| 03 | Inconsistent security controls | صلاحيات تعتمد على البريد الإلكتروني | تغيير البريد إلى `@dontwannacry.com` |
| 04 | Flawed enforcement of business rules | تطبيق غير صحيح لقواعد الخصم | التناوب بين كوبونين (`NEWCUST5` و `SIGNUP30`) |
| 05 | Low-level logic flaw | Integer Overflow | إضافة كميات كبيرة جداً حتى يتجاوز السعر الحد الأقصى |
| 06 | Inconsistent handling of exceptional input | معالجة غير متناسقة للمدخلات الطويلة | استخدام بريد أطول من 255 حرفاً مع UTF-7 |
| 07 | Weak isolation on dual-use endpoint | عزل ضعيف لنقطة نهاية مزدوجة | حذف `current-password` وتغيير `username` |
| 08 | Insufficient workflow validation | التحقق غير الكافي من تسلسل العملية | تخطي خطوة الدفع وإرسال طلب التأكيد مباشرة |
| 09 | Authentication bypass via flawed state machine | آلة حالة معطوبة | إسقاط (drop) طلب `GET /role-selector` |
| 10 | Infinite money logic flaw | ثغرة ربح غير محدود | شراء بطاقات هدية بخصم واستردادها بقيمتها الكاملة |
| 11 | Authentication bypass via encryption oracle | كشف محور تشفير (encryption oracle) | تشفير `administrator:timestamp` واستخدامه كـ `stay-logged-in` |
| 12 | Bypassing access controls using email address parsing discrepancies | اختلاف في تفسير عناوين البريد | استخدام ترميز UTF-7 لتسجيل بريد يصل إلى خادمنا مع نطاق مسموح |

---


## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/logic-flaws/lab-access-control-bypass-via-email-address-parsing-discrepancies)
- [Splitting the Email Atom (Whitepaper)](https://portswigger.net/research/splitting-the-email-atom)
- [Business Logic Vulnerabilities](https://portswigger.net/web-security/logic-flaws)
