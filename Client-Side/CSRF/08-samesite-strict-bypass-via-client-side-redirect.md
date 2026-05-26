# Lab: SameSite Strict bypass via client-side redirect

> **Category:** CSRF  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - SameSite Strict bypass via client-side redirect](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions/lab-samesite-strict-bypass-via-client-side-redirect)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة CSRF في وظيفة تغيير البريد الإلكتروني. الموقع يستخدم **SameSite=Strict** للكوكي، مما يمنع إرسال الكوكي في أي طلب عبر المواقع. ولكننا سنجد **قطعة (gadget)** تؤدي إلى إعادة توجيه (redirect) من جانب العميل، مما يسمح لنا بإرسال طلب GET مع الكوكي.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول وفهم إعدادات SameSite

- سجل الدخول بـ `wiener:peter`
- افتح **DevTools** في المتصفح (F12) → تبويب **Application** → **Cookies**
- ستجد أن كوكي `session` له خاصية `SameSite=Strict`

هذا يعني: الكوكي لن يرسل في أي طلب من موقع خارجي (لا GET ولا POST).

### الخطوة 2: البحث عن "قطعة" (gadget) تعيد التوجيه

نحتاج قطعة في الموقع نفسه تسبب **إعادة توجيه (redirect)** باستخدام مدخلات من المستخدم.

**كيف نجدها؟**

- تصفح الموقع وجرب جميع الوظائف
- وجدنا أن إضافة تعليق (Post comment) تعرض صفحة تأكيد ثم تعيد التوجيه تلقائياً

### الخطوة 3: تحليل آلية إعادة التوجيه

- اذهب إلى أي مقال (blog post)
- اكتب تعليقاً عادياً
- لاحظ أنك تُنقل إلى:
  ```
  /post/comment/confirmation?postId=1
  ```
- بعد 5 ثوانٍ، يتم توجيهك تلقائياً إلى المقال مرة أخرى

### الخطوة 4: فحص كود JavaScript المسؤول عن التوجيه

- افتح **DevTools** → تبويب **Network**
- ابحث عن ملف: `/resources/js/commentConfirmationRedirect.js`
- محتواه:

```javascript
const postId = new URLSearchParams(window.location.search).get('postId');
setTimeout(() => {
    window.location = `/post/${postId}`;
}, 5000);
```

**هذه هي القطعة!** ✅
- تأخذ `postId` من الرابط
- تبني رابط التوجيه منه مباشرة بدون تحقق

### الخطوة 5: اختبار Path Traversal في postId

جرب هذا الرابط:
```
/post/comment/confirmation?postId=1/../../my-account
```

**النتيجة:** بعد 5 ثوانٍ، يتم توجيهك إلى `/my-account` ✅

**لماذا نجح؟** لأن المتصفح يعالج المسار ويزيل `..` فتصبح: `/post/1/../../my-account` = `/my-account`

### الخطوة 6: اختبار تغيير البريد عبر GET

جرب تغيير البريد مباشرة عبر الرابط:
```
/my-account/change-email?email=hacked@attacker.net
```

**النتيجة:** ينجح! ✅ الموقع يقبل GET لتغيير البريد.

### الخطوة 7: دمج كل شيء في هجوم واحد

نحتاج إلى:
1. رابط يبدأ بـ `/post/comment/confirmation`
2. ويدخل إلى `postId` مسار يؤدي إلى تغيير البريد

الهجوم النهائي:
```
/post/comment/confirmation?postId=1/../../my-account/change-email?email=victim@attacker.net%26submit=1
```

**ملاحظة:** نستخدم `%26` بدل `&` حتى لا ينقطع معامل `postId`.

### الخطوة 8: إنشاء صفحة الهجوم على Exploit Server

```html
<script>
    document.location = "https://YOUR-LAB-ID.web-security-academy.net/post/comment/confirmation?postId=1/../../my-account/change-email?email=victim%40attacker.net%26submit=1";
</script>
```

### الخطوة 9: رفع الهجوم إلى Exploit Server

- اضغط **Go to exploit server**
- في حقل **Body**، الصق الـ HTML
- اضغط **Store**

### الخطوة 10: تجربة الهجوم على نفسك

- اضغط **View exploit**
- انتظر 5 ثوانٍ
- ارجع إلى حسابك → لاحظ أن بريدك تغير ✅

### الخطوة 11: تسليم الهجوم للضحية

- غير البريد الإلكتروني إلى قيمة مختلفة
- اضغط **Store**
- اضغط **Deliver to victim**

### الخطوة 12: حل المختبر

بعد بضع ثوانٍ، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو SameSite=Strict؟** | يمنع إرسال الكوكي في أي طلب عبر المواقع |
| **كيف تجاوزناه؟** | باستخدام إعادة توجيه من جانب العميل (client-side redirect) |
| **لماذا نجح؟** | الإعادة الأولى (خارج الموقع) لا تحمل كوكي، لكن الإعادة الثانية (داخل الموقع) تحمل كوكي |
| **ما هي القطعة (gadget)؟** | صفحة `/post/comment/confirmation` التي تعيد التوجيه ديناميكياً |
| **لماذا استخدمنا `%26`؟** | لتشفير `&` حتى لا ينقطع معامل `postId` |

---

## 📊 ملخص آلية التجاوز

| الطلب | يحمل الكوكي؟ | السبب |
|-------|-------------|-------|
| الرابط الأول (من موقع المهاجم) | ❌ لا | SameSite=Strict |
| إعادة التوجيه داخل نفس الموقع | ✅ نعم | لأنها من نفس الموقع |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

1. استخدام SameSite=Strict فقط بدون حماية إضافية
2. وجود قطعة (gadget) تسمح بـ open redirect أو path traversal في إعادة التوجيه
3. تغيير البريد يدعم طلبات GET

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// إعدادات الكوكي - SameSite=Strict فقط
res.setHeader('Set-Cookie', `session=${sessionId}; HttpOnly; Secure; SameSite=Strict`);

// قطعة إعادة التوجيه - ملف commentConfirmationRedirect.js
// خطأ: استخدام مدخلات المستخدم مباشرة في window.location
const postId = new URLSearchParams(window.location.search).get('postId');
setTimeout(() => {
    window.location = `/post/${postId}`; // Open redirect + Path traversal!
}, 5000);

// وظيفة تغيير البريد - تسمح بـ GET
export default function handler(req, res) {
  const { email, submit } = req.query; // تدعم GET
  const { sessionId } = req.cookies;
  
  const user = getUserBySession(sessionId);
  updateUserEmail(user.id, email);
  res.send('Email updated');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// إعدادات الكوكي - SameSite=Strict + CSRF Token
res.setHeader('Set-Cookie', `session=${sessionId}; HttpOnly; Secure; SameSite=Strict`);

// قطعة إعادة التوجيه - إصلاح: التحقق من المدخلات
const postId = new URLSearchParams(window.location.search).get('postId');

// السماح فقط بأرقام صحيحة (whitelist)
if (/^\d+$/.test(postId)) {
  setTimeout(() => {
    window.location = `/post/${postId}`;
  }, 5000);
} else {
  // إذا كان المدخل غير صالح، لا تعيد التوجيه
  window.location = `/post/404`;
}

// وظيفة تغيير البريد - إصلاح: رفض GET
export default function handler(req, res) {
  // رفض أي طلب ليس POST
  if (req.method !== 'POST') {
    return res.status(405).send('Method not allowed');
  }
  
  const { email, csrfToken } = req.body;
  const { sessionId } = req.cookies;
  
  // التحقق من CSRF Token
  if (!csrfToken || !verifyCsrfToken(sessionId, csrfToken)) {
    return res.status(403).send('Invalid CSRF token');
  }
  
  const user = getUserBySession(sessionId);
  updateUserEmail(user.id, email);
  res.send('Email updated');
}
```

**الخلاصة:** 
1. لا تعتمد على SameSite=Strict فقط
2. امنع open redirect و path traversal في إعادة التوجيه
3. استخدم CSRF Tokens وارفض GET للطلبات الحساسة

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **استخدام CSRF Tokens:** حتى مع SameSite=Strict.
2. **منع open redirect:** لا تستخدم مدخلات المستخدم في `window.location`.
3. **رفض GET للطلبات الحساسة:** تغيير البريد يجب أن يكون POST فقط.
4. **التحقق من المدخلات (Whitelist):** استخدم whitelist للأرقام أو المعرفات الصالحة فقط.
5. **استخدام SameSite=Lax أو Strict + CSRF Token معاً.**

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions/lab-samesite-strict-bypass-via-client-side-redirect)
- [SameSite Cookies Explained](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions)
