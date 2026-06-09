# Lab: 2FA broken logic

> **Category:** Authentication  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - 2FA broken logic](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-broken-logic)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة منطقية (Logic Flaw) في آلية التحقق بخطوتين (2FA). معامل `verify` في طلب `GET /login2` يحدد المستخدم الذي يتم إنشاء رمز 2FA له. يمكننا استخدام هذا لإنشاء رمز لـ `carlos`، ثم استخدامه لتسجيل الدخول.

**المعطيات:**
- حسابنا: `wiener:peter`
- حساب الضحية: `carlos`

**المطلوب:** الوصول إلى صفحة حساب `carlos`.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم آلية 2FA

- سجل الدخول بـ `wiener:peter`
- أدخل رمز 2FA الخاص بك (من **Email client**)

لاحظ أنه عند الوصول إلى `GET /login2`، هناك معامل `verify` يحدد المستخدم:

```http
GET /login2?verify=wiener HTTP/1.1
```

### الخطوة 2: إنشاء رمز 2FA لـ carlos

- قم بتسجيل الخروج (Log out)
- أرسل طلب `GET /login2?verify=carlos` إلى **Repeater**

```http
GET /login2?verify=carlos HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

- أرسل الطلب

**النتيجة:** تم إنشاء رمز 2FA مؤقت لـ `carlos` في الخادم.

### الخطوة 3: بدء تسجيل الدخول بـ carlos

- اذهب إلى صفحة تسجيل الدخول
- سجل الدخول باستخدام:
  - **Username:** `carlos`
  - **Password:** `montoya`

### الخطوة 4: اعتراض طلب إدخال رمز 2FA

- بعد إدخال كلمة المرور، سيُطلب منك رمز 2FA
- في هذه المرحلة، اعترض طلب `POST /login2` في Burp

```http
POST /login2 HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

verify=carlos&mfa-code=123456
```

### الخطوة 5: إرسال الطلب إلى Intruder

- زر الماوس الأيمن → **Send to Intruder**
- اختر **Attack type:** Sniper
- ضع علامات `§` حول `mfa-code`:

```
verify=carlos&mfa-code=§123456§
```

### الخطوة 6: إعداد Payloads

- **Payload type:** Numbers
- **From:** `0`
- **To:** `9999`
- **Step:** `1`
- **Min integer digits:** `4`

**ملاحظة:** رمز 2FA عادة ما يكون 4 أرقام (0000 إلى 9999).

### الخطوة 7: بدء الهجوم

- اضغط **Start attack**

سيقوم Intruder بتجربة جميع الرموز من 0000 إلى 9999.

### الخطوة 8: تحليل النتائج

- ابحث عن طلب برمز **302 Found** (إعادة توجيه)

| رمز 2FA | Status | النتيجة |
|---------|--------|---------|
| 0001 | 200 | ❌ فشل |
| 0002 | 200 | ❌ فشل |
| ... | ... | ... |
| **8472** | **302** | ✅ نجاح |

### الخطوة 9: تسجيل الرمز الصحيح

- سجل الرمز الذي ظهر معه رمز `302`

### الخطوة 10: إكمال تسجيل الدخول

- في المتصفح، أدخل الرمز الصحيح في صفحة 2FA
- سيتم توجيهك إلى صفحة حساب `carlos`

### الخطوة 11: حل المختبر

بعد الوصول إلى صفحة حساب `carlos`، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **أين الثغرة المنطقية؟** | معامل `verify` يتحكم في المستخدم الذي يتم إنشاء رمز 2FA له، ويمكن تزويره |
| **كيف نستغلها؟** | نطلب رمز 2FA لـ `carlos` أولاً (دون تسجيل دخول)، ثم نستخدمه لتسجيل الدخول |
| **لماذا نحتاج إلى Brute-Force؟** | رمز 2FA مكون من 4 أرقام (10000 احتمال) يمكن كسره بسهولة |
| **ما هو دور `GET /login2`؟** | يقوم بإنشاء رمز 2FA للمستخدم المحدد في معامل `verify` |

---

## 📊 شرح آلية الهجوم

```
1. GET /login2?verify=carlos
   → الخادم ينشئ رمز 2FA لـ carlos (دون أن يعرف carlos)

2. POST /login (username=carlos, password=montoya)
   → التحقق من كلمة المرور

3. POST /login2 (verify=carlos, mfa-code=XXXX)
   → تجربة الرموز حتى العثور على الرمز الصحيح
```

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

معامل `verify` يتحكم في المستخدم الذي يتم إنشاء رمز 2FA له ويمكن تزويره، ولا يتم ربط الرمز بجلسة (session) آمنة.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/login2.js (GET - طلب صفحة 2FA)
export default function handler(req, res) {
  const { verify } = req.query;
  
  // خطأ: إنشاء رمز 2FA لأي مستخدم (بدون تحقق من الجلسة)
  const code = generate2FACode();
  
  // تخزين الرمز مرتبطاً بالمستخدم فقط
  twoFACodes.set(verify, code);
  
  // إرسال الرمز إلى بريد المستخدم
  sendEmail(verify, `Your code is ${code}`);
  
  res.render('2fa-page', { user: verify });
}

// pages/api/login2.js (POST - التحقق من الرمز)
export default function handler(req, res) {
  const { verify, 'mfa-code': code } = req.body;
  
  // خطأ: التحقق من الرمز المرتبط بالمستخدم فقط (بدون جلسة)
  const expectedCode = twoFACodes.get(verify);
  
  if (code !== expectedCode) {
    return res.status(200).json({ error: 'Invalid code' });
  }
  
  // خطأ: تسجيل الدخول الناجح بدون ربط بجلسة
  req.session.userId = getUserByUsername(verify).id;
  res.redirect('/my-account');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/login2.js (GET - طلب صفحة 2FA)
export default function handler(req, res) {
  const sessionId = req.cookies.session;
  const session = getSession(sessionId);
  
  // التصحيح 1: التحقق من وجود جلسة صالحة
  if (!session || !session.pending2FA) {
    return res.redirect('/login');
  }
  
  // التصحيح 2: استخدام sessionId لربط الرمز بالجلسة، وليس بالمستخدم
  const code = generate2FACode();
  twoFACodes.set(sessionId, {
    code,
    expiresAt: Date.now() + 5 * 60 * 1000 // 5 دقائق
  });
  
  // إرسال الرمز إلى البريد المسجل في الجلسة
  sendEmail(session.user.email, `Your code is ${code}`);
  
  res.render('2fa-page');
}

// pages/api/login2.js (POST - التحقق من الرمز)
export default function handler(req, res) {
  const sessionId = req.cookies.session;
  const { 'mfa-code': code } = req.body;
  
  const session = getSession(sessionId);
  
  // التصحيح 3: التحقق من وجود جلسة صالحة مع 2FA معلق
  if (!session || !session.pending2FA) {
    return res.redirect('/login');
  }
  
  const record = twoFACodes.get(sessionId);
  
  // التصحيح 4: التحقق من صلاحية الرمز زمنياً
  if (!record || record.expiresAt < Date.now()) {
    twoFACodes.delete(sessionId);
    return res.status(200).json({ error: 'Code expired' });
  }
  
  // التصحيح 5: التحقق من صحة الرمز
  if (code !== record.code) {
    return res.status(200).json({ error: 'Invalid code' });
  }
  
  // التصحيح 6: إكمال المصادقة
  twoFACodes.delete(sessionId);
  session.is2FAVerified = true;
  session.pending2FA = false;
  updateSession(sessionId, session);
  
  res.redirect('/my-account');
}
```

**الخلاصة:** 
1. اربط رمز 2FA بـ **session ID**، وليس بـ username
2. لا تسمح بإنشاء رمز 2FA بدون جلسة صالحة
3. أضف صلاحية زمنية للرمز (5 دقائق)
4. استخدم معدل محدود للمحاولات (rate limiting) لمنع Brute-Force

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **ربط الرمز بالجلسة** | استخدم session ID بدلاً من username |
| **صلاحية زمنية قصيرة** | اجعل الرمز صالحاً لمدة 5 دقائق فقط |
| **معدل محدود للمحاولات** | امنع أكثر من 3 محاولات خاطئة لكل رمز |
| **عدم الكشف عن الوجود** | لا تخبر المستخدم إذا كان الرمز صحيحاً أم لا |
| **إعادة تعيين الرمز بعد الفشل** | بعد 3 محاولات فاشلة، أعد إنشاء الرمز |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-broken-logic)
- [2FA Bypass](https://portswigger.net/web-security/authentication/multi-factor#2fa-bypass)
