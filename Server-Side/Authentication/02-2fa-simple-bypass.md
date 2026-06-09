# Lab: 2FA simple bypass

> **Category:** Authentication  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - 2FA simple bypass](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-simple-bypass)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة في آلية التحقق بخطوتين (2FA). بعد تسجيل الدخول باسم المستخدم وكلمة المرور، يتم إعادة التوجيه إلى صفحة إدخال رمز التحقق. ولكن يمكننا **تخطي (bypass)** هذه الصفحة بالانتقال مباشرة إلى `/my-account`.

**المعطيات:**
- حسابنا: `wiener:peter`
- حساب الضحية: `carlos:montoya`

**المطلوب:** الوصول إلى صفحة حساب `carlos`.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول بحسابنا (لفهم العملية)

- سجل الدخول بـ `wiener:peter`
- سيُطلب منك إدخال رمز التحقق (2FA)
- اذهب إلى **Email client** لاستلام الرمز
- أدخل الرمز وانتقل إلى صفحة `/my-account`

**ملاحظة:** لاحظ الرابط بعد تسجيل الدخول: `https://YOUR-LAB-ID.web-security-academy.net/my-account`

### الخطوة 2: تسجيل الخروج

- قم بتسجيل الخروج (Log out)

### الخطوة 3: تسجيل الدخول بحساب carlos

- سجل الدخول بـ `carlos:montoya`
- سيُطلب منك إدخال رمز التحقق

### الخطوة 4: تجاوز صفحة 2FA

- **بدلاً من إدخال الرمز**، غيّر الرابط في شريط العناوين إلى:

```
https://YOUR-LAB-ID.web-security-academy.net/my-account
```

- اضغط **Enter**

### الخطوة 5: حل المختبر

**النتيجة:** يتم تحميل صفحة حساب `carlos` مباشرة ويتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **أين الثغرة؟** | التطبيق يتحقق من وجود جلسة (session) صالحة بعد إدخال كلمة المرور، ولكنه لا يتحقق من إكمال خطوة 2FA قبل الوصول إلى `/my-account` |
| **كيف نستغلها؟** | بعد تسجيل الدخول باسم المستخدم وكلمة المرور، ننتقل مباشرة إلى `/my-account` |
| **لماذا نجح هذا؟** | الخادم يعتبر أن أي طلب مع جلسة صالحة (حتى قبل إدخال رمز 2FA) يمكنه الوصول إلى الصفحات المحمية |
| **ما هو المطلوب؟** | فقط اسم المستخدم وكلمة المرور الصحيحة، بدون الحاجة إلى رمز 2FA |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

التطبيق يتحقق من صحة الجلسة (session) فقط، دون التحقق من إكمال خطوة التحقق بخطوتين (2FA).

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// middleware.js - التحقق من المصادقة
export function middleware(req) {
  const sessionId = req.cookies.session;
  const user = getSession(sessionId);
  
  // خطأ: يتحقق فقط من وجود جلسة صالحة
  if (!user) {
    return Response.redirect(new URL('/login', req.url));
  }
  
  // يسمح بالوصول إلى /my-account حتى لو لم يكتمل الـ 2FA
  return NextResponse.next();
}

// pages/api/login.js - عملية تسجيل الدخول
export default function handler(req, res) {
  const { username, password } = req.body;
  const user = authenticate(username, password);
  
  if (!user) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  // إنشاء جلسة (حتى قبل إكمال 2FA)
  const sessionId = createSession(user.id);
  res.setHeader('Set-Cookie', `session=${sessionId}`);
  
  // التوجيه إلى صفحة 2FA
  res.redirect('/2fa');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// middleware.js - التحقق من المصادقة
export function middleware(req) {
  const sessionId = req.cookies.session;
  const session = getSession(sessionId);
  const url = req.nextUrl.pathname;
  
  // الصفحات العامة المسموحة بدون مصادقة
  const publicPaths = ['/login', '/2fa'];
  if (publicPaths.includes(url)) {
    return NextResponse.next();
  }
  
  // خطأ: غير مسجل دخول
  if (!session) {
    return Response.redirect(new URL('/login', req.url));
  }
  
  // التصحيح: التحقق من إكمال 2FA
  if (session.requires2FA && !session.is2FAVerified) {
    return Response.redirect(new URL('/2fa', req.url));
  }
  
  return NextResponse.next();
}

// pages/api/2fa-verify.js - التحقق من رمز 2FA
export default function handler(req, res) {
  const { code } = req.body;
  const sessionId = req.cookies.session;
  const session = getSession(sessionId);
  
  if (!session) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  const isValid = verify2FACode(session.userId, code);
  
  if (!isValid) {
    return res.status(401).json({ error: 'Invalid code' });
  }
  
  // التصحيح: تحديث حالة الجلسة بإكمال 2FA
  session.is2FAVerified = true;
  session.requires2FA = false;
  updateSession(sessionId, session);
  
  res.redirect('/my-account');
}
```

**الخلاصة:** 
1. لا تسمح بالوصول إلى الصفحات المحمية إلا بعد إكمال خطوة 2FA
2. استخدم حقل حالة في الجلسة (مثل `is2FAVerified`)
3. تحقق من هذه الحالة في الـ middleware

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **تتبع حالة 2FA في الجلسة** | أضف حقلاً مثل `2fa_completed` في الجلسة |
| **التحقق في middleware** | امنع الوصول إلى `/my-account` إذا لم يكتمل 2FA |
| **إبطال الجلسة بعد الخروج** | تأكد من أن تسجيل الخروج يلغي حالة 2FA |
| **صلاحية زمنية لـ 2FA** | اجعل رمز 2FA منتهياً بعد فترة قصيرة |
| **لا تعتمد على التوجيه فقط** | اختبر الصلاحية في كل نقطة نهاية محمية |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-simple-bypass)
- [2FA Bypass](https://portswigger.net/web-security/authentication/multi-factor#2fa-bypass)
