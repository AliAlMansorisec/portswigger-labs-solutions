# Lab: Brute-forcing a stay-logged-in cookie

> **Category:** Authentication  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Brute-forcing a stay-logged-in cookie](https://portswigger.net/web-security/authentication/other-mechanisms/lab-brute-forcing-a-stay-logged-in-cookie)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة في آلية "البقاء متصلاً" (stay logged in). الكوكي المستخدمة لهذه الوظيفة قابلة للهجوم بالقوة الغاشمة (Brute-Force) لأنها تتكون من `username:MD5(password)` مشفرة بـ Base64.

**المعطيات:**
- حسابنا: `wiener:peter`
- حساب الضحية: `carlos`
- قائمة كلمات المرور المرشحة (Candidate passwords)

**المطلوب:** العثور على كوكي صالحة لـ `carlos` والوصول إلى صفحة حسابه.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم بنية كوكي "stay-logged-in"

- سجل الدخول بـ `wiener:peter` مع تفعيل خيار **Stay logged in**
- في Burp، انظر إلى كوكي `stay-logged-in`:

```
stay-logged-in=d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

### الخطوة 2: فك تشفير الكوكي

- انسخ قيمة الكوكي
- في Burp **Decoder**، قم بـ **Base64 decode**:

```
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

### الخطوة 3: فهم البنية

- `wiener` = اسم المستخدم
- `51dc30ddc473d43a6011e9ebba6ca770` = MD5 hash لكلمة المرور `peter`

**تأكد:**  
`md5("peter")` = `51dc30ddc473d43a6011e9ebba6ca770`

**التركيب العام:**
```
stay-logged-in = base64(username + ":" + md5(password))
```

### الخطوة 4: تسجيل الخروج

- قم بتسجيل الخروج (Log out)

### الخطوة 5: إرسال الطلب إلى Intruder

- اعترض طلب `GET /my-account?id=wiener`
- أرسله إلى **Intruder**
- ضع علامات `§` حول قيمة كوكي `stay-logged-in`:

```
Cookie: stay-logged-in=§d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw§
```

### الخطوة 6: إعداد Payload (لاختبار البنية)

- **Payload type:** Simple list
- أضف كلمة مرور واحدة فقط: `peter`

### الخطوة 7: إعداد Payload Processing (سلسلة المعالجة)

أضف القواعد التالية **بالترتيب**:

| الترتيب | القاعدة | القيمة |
|---------|---------|--------|
| 1 | Hash | MD5 |
| 2 | Add prefix | `wiener:` |
| 3 | Encode | Base64-encode |

**شرح:** كل payload (`peter`) → MD5 (`51dc30...`) → `wiener:51dc30...` → Base64 (`d2llbmVy...`)

### الخطوة 8: إعداد Grep - Extract

- في **Settings** → **Grep - Extract** → **Add**
- حدد نص **Update email** من الـ response (يظهر فقط عند تسجيل الدخول بنجاح)

### الخطوة 9: اختبار الهجوم (على حسابنا)

- اضغط **Start attack**
- **النتيجة:** سينجح الطلب ويظهر `Update email` ✅

### الخطوة 10: تعديل الهجوم لحساب carlos

- غيّر `id=wiener` إلى `id=carlos` في الرابط:

```
GET /my-account?id=carlos
```

- غيّر قائمة payloads: أزل `peter` وأضف **Candidate passwords**
- غيّر **Add prefix** إلى `carlos:`

### الخطوة 11: بدء الهجوم على carlos

- اضغط **Start attack**

### الخطوة 12: تحليل النتائج

- ابحث عن الطلب الذي يحتوي على **Update email** في الـ response

| كلمة المرور | تحتوي على "Update email"? |
|-------------|---------------------------|
| 123456 | ❌ لا |
| password | ❌ لا |
| **qwerty** | **✅ نعم** |

### الخطوة 13: تسجيل الكوكي الصحيحة

- سجل قيمة كوكي `stay-logged-in` من الطلب الناجح

### الخطوة 14: استخدام الكوكي للوصول إلى حساب carlos

- في المتصفح، استبدل كوكي `stay-logged-in` بالقيمة التي وجدتها
- أو في Repeater، أرسل الطلب مع الكوكي الجديدة

### الخطوة 15: حل المختبر

بعد الوصول إلى صفحة حساب `carlos`، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي بنية الكوكي؟** | `base64(username + ":" + md5(password))` |
| **لماذا هذا ضعيف؟** | MD5 هي خوارزمية ضعيفة وسريعة، يمكن كسرها بالقوة الغاشمة |
| **كيف نستغلها؟** | نصنع كوكي لكل كلمة مرور محتملة ونختبرها |
| **كيف نعرف النجاح؟** | وجود نص "Update email" في صفحة الحساب |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

استخدام MD5 (خوارزمية ضعيفة وسريعة) لتجزئة كلمة المرور في كوكي "stay-logged-in"، مع تركيب يمكن تزويره بسهولة.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/login.js
import crypto from 'crypto';

export default function handler(req, res) {
  const { username, password, stayLoggedIn } = req.body;
  
  const user = getUserByUsername(username);
  
  if (!user || user.password !== password) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  // خطأ: استخدام MD5 (ضعيف) لإنشاء كوكي
  if (stayLoggedIn) {
    const hash = crypto.createHash('md5').update(password).digest('hex');
    const cookieValue = Buffer.from(`${username}:${hash}`).toString('base64');
    res.setHeader('Set-Cookie', `stay-logged-in=${cookieValue}; HttpOnly`);
  }
  
  req.session.userId = user.id;
  res.redirect('/my-account');
}

// التحقق من كوكي stay-logged-in (middleware)
export function authMiddleware(req, res, next) {
  const stayLoggedInCookie = req.cookies['stay-logged-in'];
  
  if (stayLoggedInCookie) {
    const decoded = Buffer.from(stayLoggedInCookie, 'base64').toString();
    const [username, hash] = decoded.split(':');
    const user = getUserByUsername(username);
    
    // خطأ: التحقق من MD5 (ضعيف وقابل للهجوم)
    if (user && hash === crypto.createHash('md5').update(user.password).digest('hex')) {
      req.user = user;
    }
  }
  
  next();
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/login.js
import crypto from 'crypto';
import { v4 as uuidv4 } from 'uuid';

export default function handler(req, res) {
  const { username, password, stayLoggedIn } = req.body;
  
  const user = getUserByUsername(username);
  
  if (!user || user.password !== password) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  if (stayLoggedIn) {
    // التصحيح 1: استخدام رمز عشوائي طويل بدلاً من MD5
    const token = crypto.randomBytes(32).toString('hex');
    const expiresAt = Date.now() + 30 * 24 * 60 * 60 * 1000; // 30 يوم
    
    // التصحيح 2: تخزين الرمز في قاعدة البيانات مرتبطاً بالمستخدم
    storeSessionToken(user.id, token, expiresAt);
    
    // التصحيح 3: استخدام رمز آمن في الكوكي (بدون معلومات المستخدم)
    res.setHeader('Set-Cookie', `stay-logged-in=${token}; HttpOnly; Secure; SameSite=Strict; Max-Age=${30*24*60*60}`);
  }
  
  req.session.userId = user.id;
  res.redirect('/my-account');
}

// التحقق من كوكي stay-logged-in (middleware)
export async function authMiddleware(req, res, next) {
  const token = req.cookies['stay-logged-in'];
  
  if (token) {
    // التصحيح 4: التحقق من الرمز في قاعدة البيانات
    const session = await getSessionByToken(token);
    
    if (session && session.expiresAt > Date.now()) {
      req.user = getUserById(session.userId);
    } else {
      // إزالة الكوكي المنتهية
      res.setHeader('Set-Cookie', 'stay-logged-in=; Max-Age=0');
    }
  }
  
  next();
}
```

**الخلاصة:** 
1. لا تستخدم MD5 أو خوارزميات ضعيفة للمصادقة
2. استخدم رموز عشوائية طويلة (tokens) بدلاً من تجزئة كلمة المرور
3. خزّن الرموز في قاعدة البيانات مع صلاحية زمنية
4. لا تضع معلومات تعريفية (username) في الكوكي

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **استخدام رموز عشوائية** | استخدم `crypto.randomBytes()` بدلاً من MD5 |
| **تخزين الرموز في قاعدة البيانات** | اربط الرمز بالمستخدم في DB |
| **صلاحية زمنية** | اجعل الرمز ينتهي بعد فترة (مثلاً 30 يوماً) |
| **HttpOnly + Secure + SameSite** | اجعل الكوكي آمنة |
| **لا تضع معلومات في الكوكي** | استخدم رمزاً عشوائياً لا يمكن فك تشفيره |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/authentication/other-mechanisms/lab-brute-forcing-a-stay-logged-in-cookie)
- [Stay Logged In Vulnerabilities](https://portswigger.net/web-security/authentication/other-mechanisms#stay-logged-in)
