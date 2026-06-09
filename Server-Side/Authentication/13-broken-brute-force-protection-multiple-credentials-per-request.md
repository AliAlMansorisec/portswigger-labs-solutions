# Lab: Broken brute-force protection, multiple credentials per request

> **Category:** Authentication  
> **Difficulty:** EXPERT  
> **Lab Link:** [PortSwigger Lab - Broken brute-force protection, multiple credentials per request](https://portswigger.net/web-security/authentication/password-based/lab-broken-brute-force-protection-multiple-credentials-per-request)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة منطقية (Logic Flaw) في آلية حماية القوة الغاشمة (Brute-Force Protection). الخادم يقبل طلبات JSON حيث يكون معامل `password` مصفوفة (array) من كلمات المرور بدلاً من قيمة واحدة. يقوم الخادم باختبار جميع كلمات المرور في المصفوفة، وعندما يجد كلمة المرور الصحيحة، يقوم بتسجيل الدخول وإعادة التوجيه (302).

**المعطيات:**
- حساب الضحية: `carlos`
- قائمة كلمات المرور المرشحة (Candidate passwords)

**المطلوب:** العثور على كلمة مرور `carlos` وتسجيل الدخول بحسابه.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم هيكل طلب تسجيل الدخول

- في Burp، اعترض طلب `POST /login`:

```http
POST /login HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/json

{"username":"carlos","password":"123456"}
```

**ملاحظة:** الطلب بصيغة JSON، وليس `application/x-www-form-urlencoded`.

### الخطوة 2: إرسال الطلب إلى Repeater

- زر الماوس الأيمن → **Send to Repeater**

### الخطوة 3: تغيير password إلى مصفوفة

- غيّر قيمة `password` من نص (string) إلى **مصفوفة (array)** تحتوي على جميع كلمات المرور المرشحة:

```json
{
    "username": "carlos",
    "password": [
        "123456",
        "password",
        "qwerty",
        "admin",
        "letmein",
        "monkey",
        ...
    ]
}
```

### الخطوة 4: إرسال الطلب

- اضغط **Send**

### الخطوة 5: تحليل الرد

**النتيجة:** يجب أن ترى رمز **302 Found** (إعادة توجيه) بدلاً من 401 Unauthorized.

### الخطوة 6: عرض الرد في المتصفح

- زر الماوس الأيمن على الطلب → **Show response in browser**
- انسخ الرابط
- الصقه في المتصفح

### الخطوة 7: حل المختبر

**النتيجة:** يتم تسجيل الدخول تلقائياً كـ `carlos` ويتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي الثغرة؟** | الخادم يقبل مصفوفة من كلمات المرور ويختبرها جميعاً في طلب واحد |
| **لماذا هذا خطأ؟** | هذا يسمح للمهاجم بتجربة مئات أو آلاف كلمات المرور في طلب واحد |
| **كيف نستغلها؟** | نرسل مصفوفة تحتوي على جميع كلمات المرور المرشحة في طلب واحد |
| **ما هي الميزة؟** | يتجاوز حماية القوة الغاشمة (IP block, rate limiting, account lockout) |

---

## 📊 مقارنة بين الطلب العادي والثغرة

| الطلب العادي | الطلب المستغل |
|--------------|---------------|
| `{"username":"carlos","password":"123456"}` | `{"username":"carlos","password":["123456","password","qwerty",...]}` |
| طلب واحد لكل كلمة مرور | طلب واحد لجميع كلمات المرور |
| يكتشفه نظام الحماية | يتجاوز نظام الحماية |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يقبل مصفوفة (array) بدلاً من نص (string) في معامل `password`، ويقوم باختبار جميع القيم حتى يجد القيمة الصحيحة.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/login.js
export default function handler(req, res) {
  const { username, password } = req.body;
  
  const user = getUserByUsername(username);
  
  // خطأ: يقبل password كمصفوفة
  if (Array.isArray(password)) {
    // يختبر جميع كلمات المرور في المصفوفة
    for (const pwd of password) {
      if (user && user.password === pwd) {
        // يسجل الدخول عند أول كلمة مرور صحيحة
        req.session.userId = user.id;
        return res.redirect('/my-account');
      }
    }
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  // الحالة العادية (نص واحد)
  if (!user || user.password !== password) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  req.session.userId = user.id;
  res.redirect('/my-account');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/login.js
export default function handler(req, res) {
  let { username, password } = req.body;
  
  // التصحيح 1: التحقق من نوع البيانات
  if (!username || typeof username !== 'string') {
    return res.status(400).json({ error: 'Invalid username format' });
  }
  
  // التصحيح 2: رفض أي طلب password ليس من نوع string
  if (!password || typeof password !== 'string') {
    return res.status(400).json({ error: 'Invalid password format' });
  }
  
  // التصحيح 3: التحقق من طول كلمة المرور (منع الهجمات)
  if (password.length > 128) {
    return res.status(400).json({ error: 'Password too long' });
  }
  
  const user = getUserByUsername(username);
  
  // التصحيح 4: استخدام خوارزمية بطيئة (bcrypt) مع تأخير ثابت
  const isValid = user && await bcrypt.compare(password, user.passwordHash);
  
  if (!isValid) {
    // تأخير ثابت لمنع timing attacks
    await new Promise(resolve => setTimeout(resolve, 100));
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  req.session.userId = user.id;
  res.redirect('/my-account');
}
```

```javascript
// middleware/validate-input.js - التحقق من صحة المدخلات
export function validateInput(req, res, next) {
  // التصحيح 5: التحقق من صحة JSON قبل المعالجة
  if (req.headers['content-type'] === 'application/json') {
    const body = req.body;
    
    // رفض أي حقل password هو مصفوفة
    if (body.password && Array.isArray(body.password)) {
      return res.status(400).json({ error: 'Invalid request format' });
    }
  }
  
  next();
}
```

**الخلاصة:** 
1. تحقق من نوع البيانات (type checking) لمعامل `password`
2. ارفض أي طلب يكون فيه `password` مصفوفة (array)
3. استخدم خوارزميات بطيئة (bcrypt) لجميع محاولات تسجيل الدخول
4. أضف تأخيراً ثابتاً لمنع timing attacks

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **التحقق من نوع البيانات** | تأكد من أن `password` من نوع `string` وليس `array` |
| **رفض المصفوفات** | ارفض أي طلب يحتوي على مصفوفة في حقل حساس |
| **تقييد الطول** | حدد أقصى طول لكلمة المرور (مثلاً 128 حرفاً) |
| **استخدام Schema Validation** | استخدم مكتبات مثل Joi أو Zod للتحقق من صحة المدخلات |
| **خوارزميات بطيئة** | استخدم bcrypt أو Argon2 لتجزئة كلمات المرور |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/authentication/password-based/lab-broken-brute-force-protection-multiple-credentials-per-request)
- [Mass Assignment Vulnerabilities](https://portswigger.net/web-security/api-testing#mass-assignment)
