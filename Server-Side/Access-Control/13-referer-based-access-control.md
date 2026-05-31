# Lab: Referer-based access control

> **Category:** Access Control  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Referer-based access control](https://portswigger.net/web-security/access-control/lab-referer-based-access-control)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة التحكم في الوصول (Access Control) التي تعتمد على **هيدر Referer**. الخادم يتحقق من أن الطلب يأتي من صفحة `/admin`، ولكن يمكننا **تزوير** هيدر Referer لتجاوز الحماية.

**المعطيات:**
- حساب المدير: `administrator:admin` (لمعرفة كيف تعمل الوظيفة)
- حسابنا: `wiener:peter`

**المطلوب:** ترقية حساب `wiener` ليصبح مديراً (administrator).

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول بحساب المدير (للفهم)

- سجل الدخول بـ `administrator:admin`
- اذهب إلى لوحة التحكم `/admin`
- لاحظ أن ترقية مستخدم (مثل `carlos`) تتم عبر طلب إلى `/admin-roles`

### الخطوة 2: اعتراض طلب الترقية في Burp

- قم بترقية مستخدم (مثل `carlos`)
- اعترض الطلب في Burp:

```http
POST /admin-roles HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Referer: https://YOUR-LAB-ID.web-security-academy.net/admin
Cookie: session=ADMIN_SESSION_COOKIE
Content-Type: application/x-www-form-urlencoded

username=carlos&action=upgrade
```

- أرسل هذا الطلب إلى **Repeater**

### الخطوة 3: اختبار أهمية Referer

- في Repeater، احذف هيدر `Referer`
- أرسل الطلب:

**النتيجة:** `401 Unauthorized` (ممنوع) ❌

هذا يثبت أن الخادم **يعتمد على Referer** للتحقق من الصلاحية.

### الخطوة 4: تسجيل الدخول بحساب wiener

- افتح نافذة خاصة (Incognito/Private)
- سجل الدخول بـ `wiener:peter`

### الخطوة 5: استبدال كوكي الجلسة

- في Repeater، استبدل كوكي الجلسة (session) بـ **كوكي wiener**
- غيّر `username=carlos` إلى `username=wiener`
- **أضف هيدر Referer** (نفسه كما في طلب المدير):

```http
POST /admin-roles HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Referer: https://YOUR-LAB-ID.web-security-academy.net/admin
Cookie: session=WIENER_SESSION_COOKIE
Content-Type: application/x-www-form-urlencoded

username=wiener&action=upgrade
```

### الخطوة 6: إرسال الطلب

- اضغط **Send**

**النتيجة:** تم ترقية `wiener` إلى مدير ✅

### الخطوة 7: التحقق من الصلاحية

- في المتصفح (حساب wiener)، اذهب إلى `/admin`
- ستظهر لوحة التحكم ✅

### الخطوة 8: حل المختبر

بمجرد الوصول إلى لوحة التحكم (أو ترقية نفسك)، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو Referer header؟** | يشير إلى عنوان الصفحة التي أرسلت الطلب |
| **كيف يتحقق الخادم؟** | يتوقع أن يكون Referer مساوياً لصفحة `/admin` |
| **أين الثغرة؟** | الخادم يعتمد على هيدر يمكن **تزويره** بسهولة من قبل العميل |
| **كيف نستغلها؟** | نضيف `Referer: https://LAB-ID.web-security-academy.net/admin` إلى الطلب |
| **لماذا نجحت؟** | الخادم يثق بـ Referer كآلية تحقق، رغم أنه يمكن تزويره |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يستخدم هيدر `Referer` (الذي يمكن تزويره من قبل العميل) كآلية وحيدة للتحقق من الصلاحية.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/admin-roles.js
export default function handler(req, res) {
  const { sessionId } = req.cookies;
  const { username, action } = req.body;
  const referer = req.headers.referer;
  
  // خطأ: التحقق من Referer فقط
  if (!referer || !referer.includes('/admin')) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  // لا يوجد تحقق من أن المستخدم الحقيقي هو مدير!
  if (action === 'upgrade') {
    upgradeUser(username);
  }
  
  res.json({ success: true });
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/admin-roles.js
import { getSession } from 'next-auth/react';

export default async function handler(req, res) {
  const session = await getSession({ req });
  const { username, action } = req.body;
  
  // التصحيح: التحقق الفعلي من الصلاحية
  if (!session || session.user.role !== 'admin') {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  // Referer يمكن استخدامه كطبقة دفاع إضافية فقط
  const referer = req.headers.referer;
  if (!referer || !referer.includes('/admin')) {
    // سجل المخالفة ولكن لا تمنع الطلب
    console.warn(`Suspicious request from ${referer} by user ${session.user.id}`);
  }
  
  if (action === 'upgrade') {
    upgradeUser(username);
  }
  
  res.json({ success: true });
}
```

**الخلاصة:** 
1. لا تعتمد على هيدر `Referer` كآلية أمان وحيدة
2. استخدم المصادقة والصلاحيات (authentication & authorization) الفعلية
3. Referer يمكن استخدامه كطبقة دفاع إضافية (ليس كحماية أساسية)

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **استخدام المصادقة والصلاحيات:** تحقق من دور المستخدم (role) في كل طلب حساس.
2. **لا تعتمد على Referer:** يمكن تزويره بسهولة باستخدام أدوات مثل Burp.
3. **استخدم CSRF Tokens:** كحماية إضافية.
4. **التحقق من Origin أيضاً:** كطبقة دفاع إضافية، لكن لا تعتمد عليه وحده.

---

## 📊 ملخص لابات Access Control

| # | اللاب | نوع الثغرة | طريقة الاستغلال |
|---|-------|-----------|----------------|
| 01 | Unprotected admin functionality | معلومات حساسة في robots.txt | زيارة `/robots.txt` ثم `/administrator-panel` |
| 02 | Unprotected admin functionality with unpredictable URL | رابط عشوائي مكشوف في المصدر | البحث في HTML/JS عن الرابط الإداري |
| 03 | User role controlled by request parameter | صلاحية المستخدم في كوكي قابلة للتزوير | تغيير `Admin=false` إلى `true` |
| 04 | User role can be modified in user profile | صلاحية المستخدم في طلب JSON | إضافة `"roleid":2` إلى طلب تحديث البريد |
| 05 | User ID controlled by request parameter | Horizontal Privilege Escalation | تغيير `id=wiener` إلى `id=carlos` |
| 06 | User ID controlled with unpredictable user IDs | Horizontal (مع GUIDs) | البحث عن GUID كارلوس في `/blogs` ثم استخدامه |
| 07 | Data leakage in redirect | تسرب بيانات في رد الـ redirect | تغيير `id=carlos` وقراءة API key من Body الـ 302 |
| 08 | User ID controlled with password disclosure | تسرب كلمة المرور | تغيير `id=administrator` وقراءة قيمة `password` |
| 09 | Insecure direct object references | IDOR | تغيير رقم ملف المحادثة من `2.txt` إلى `1.txt` |
| 10 | URL-based access control can be circumvented | تجاوز التحكم بالوصول عبر URL | استخدام هيدر `X-Original-URL: /admin` |
| 11 | Method-based access control can be circumvented | تجاوز التحكم بالوصول عبر Method | تحويل `POST` إلى `GET` |
| 12 | Multi-step process with no access control | ثغرة في خطوة واحدة من عملية متعددة | تخطي الخطوة الأولى وتنفيذ طلب التأكيد مباشرة |
| 13 | Referer-based access control | التحكم بالوصول عبر Referer | إضافة `Referer: .../admin` إلى الطلب |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/access-control/lab-referer-based-access-control)
- [Access Control Cheat Sheet](https://portswigger.net/web-security/access-control)
- [Referer Header Spoofing](https://portswigger.net/web-security/access-control#referer-based-access-control)
