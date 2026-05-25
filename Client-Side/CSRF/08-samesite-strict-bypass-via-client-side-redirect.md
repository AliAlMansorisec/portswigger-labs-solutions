# Lab: SameSite Strict bypass via client-side redirect

> **Category:** CSRF  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - SameSite Strict bypass via client-side redirect](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions/lab-samesite-strict-bypass-via-client-side-redirect)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة CSRF في وظيفة تغيير البريد الإلكتروني. الموقع يستخدم **SameSite=Strict** للكوكي، مما يمنع إرسال الكوكي في أي طلب عبر المواقع. ولكننا سنجد **قطعة (gadget)** تؤدي إلى إعادة توجيه (redirect) من جانب العميل، مما يسمح لنا بإرسال طلب GET مع الكوكي.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم آلية SameSite=Strict

- الـ session cookie له `SameSite=Strict`
- هذا يمنع إرسال الكوكي في **أي طلب** عبر المواقع (GET أو POST)

### الخطوة 2: البحث عن قطعة (gadget) لإعادة التوجيه

نكتشف أن صفحة تأكيد التعليق تقوم بإعادة توجيه من جانب العميل:

```javascript
// /resources/js/commentConfirmationRedirect.js
// تأخذ postId من query parameter وتستخدمه لبناء رابط التوجيه
window.location = `/post/${postId}`;
```

### الخطوة 3: اختبار إعادة التوجيه

نزور الرابط التالي:
```
/post/comment/confirmation?postId=1/../../my-account
```

**النتيجة:** يتم توجيهنا إلى `/my-account` بنجاح ✅

### الخطوة 4: فهم كيف يتجاوز SameSite=Strict

1. الضحية يزور صفحتنا الخبيثة
2. صفحتنا توجهه إلى:
   ```
   /post/comment/confirmation?postId=xxx
   ```
3. هذا الطلب يأتي من **خارج الموقع**، لكنه **GET مع top-level navigation** (document.location)
4. SameSite=Strict يمنع الكوكي هنا ❌... ولكن بعد التحميل، الـ JavaScript ينفذ **إعادة توجيه داخل نفس الموقع**
5. الإعادة الثانية تأتي من **نفس الموقع** ✅ لذلك يتم إرسال الكوكي

### الخطوة 5: تجربة تغيير البريد عبر GET

نختبر إذا كان تغيير البريد يدعم GET:

```http
GET /my-account/change-email?email=hacked%40attacker.net HTTP/1.1
```

**النتيجة:** ينجح! ✅

### الخطوة 6: إنشاء الهجوم النهائي

نقوم بدمج كل شيء:

```html
<script>
    document.location = "https://YOUR-LAB-ID.web-security-academy.net/post/comment/confirmation?postId=1/../../my-account/change-email?email=victim%40attacker.net%26submit=1";
</script>
```

> **ملاحظة:** استخدمنا `%26` بدل `&` حتى لا ينقطع معامل `postId`.

### الخطوة 7: رفع الهجوم إلى Exploit Server

- اضغط **Go to exploit server**
- في حقل **Body**، الصق الـ HTML
- اضغط **Store**

### الخطوة 8: تجربة الهجوم على نفسك

- اضغط **View exploit**
- ارجع إلى حسابك وتأكد من تغير البريد

### الخطوة 9: تسليم الهجوم للضحية

- غير البريد الإلكتروني إلى قيمة مختلفة
- اضغط **Store**
- اضغط **Deliver to victim**

### الخطوة 10: حل المختبر

بعد بضع ثوانٍ، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو SameSite=Strict؟** | يمنع إرسال الكوكي في أي طلب عبر المواقع |
| **كيف تجاوزناه؟** | باستخدام إعادة توجيه من جانب العميل (client-side redirect) |
| **لماذا نجح؟** | الإعادة الأولى (خارج الموقع) لا تحمل كوكي، لكن الإعادة الثانية (داخل الموقع) تحمل كوكي |
| **ما هي القطعة (gadget)؟** | صفحة `/post/comment/confirmation` التي تعيد التوجيه ديناميكياً |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

1. استخدام SameSite=Strict فقط بدون حماية إضافية
2. وجود قطعة (gadget) تسمح بـ open redirect أو إعادة توجيه ديناميكية
3. تغيير البريد يدعم طلبات GET

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// إعدادات الكوكي - SameSite=Strict فقط
res.setHeader('Set-Cookie', `session=${sessionId}; HttpOnly; Secure; SameSite=Strict`);

// قطعة إعادة التوجيه - ملف commentConfirmationRedirect.js
// خطأ: استخدام مدخلات المستخدم مباشرة في window.location
const postId = new URLSearchParams(window.location.search).get('postId');
window.location = `/post/${postId}`; // Open redirect!

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

// السماح فقط بأرقام صحيحة
if (/^\d+$/.test(postId)) {
  window.location = `/post/${postId}`;
} else {
  window.location = `/post/404`;
}

// وظيفة تغيير البريد - إصلاح: رفض GET
export default function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).send('Method not allowed');
  }
  
  const { email, csrfToken } = req.body;
  const { sessionId } = req.cookies;
  
  if (!verifyCsrfToken(sessionId, csrfToken)) {
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
4. **التحقق من المدخلات:** استخدم whitelist للأرقام أو المعرفات الصالحة.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions/lab-samesite-strict-bypass-via-client-side-redirect)
- [SameSite Cookies Explained](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions)
