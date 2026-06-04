# Lab: Information disclosure on debug page

> **Category:** Information Disclosure  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - Information disclosure on debug page](https://portswigger.net/web-security/information-disclosure/lab-information-disclosure-on-debug-page)

---

## 🎯 الهدف الرئيسي

العثور على صفحة تصحيح أخطاء (debug page) مخفية في الموقع تكشف معلومات حساسة عن التطبيق، ثم استخراج متغير البيئة `SECRET_KEY` وإرساله.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فتح الصفحة الرئيسية

- افتح المختبر في المتصفح

### الخطوة 2: البحث عن تعليقات HTML

- في Burp → **Target** > **Site map**
- زر الماوس الأيمن على النطاق الرئيسي → **Engagement tools** → **Find comments**

### الخطوة 3: العثور على رابط Debug

ستظهر التعليقات التي تحتوي على:

```html
<!-- <a href="/cgi-bin/phpinfo.php">Debug</a> -->
```

**النتيجة:** تم العثور على صفحة تصحيح الأخطاء في `/cgi-bin/phpinfo.php`

### الخطوة 4: إرسال الطلب إلى Repeater

- في Site map، ابحث عن `/cgi-bin/phpinfo.php`
- زر الماوس الأيمن → **Send to Repeater**

### الخطوة 5: زيارة صفحة phpinfo

- في Repeater، أرسل الطلب:

```http
GET /cgi-bin/phpinfo.php HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

### الخطوة 6: البحث عن SECRET_KEY

ستظهر صفحة `phpinfo()` تحتوي على جميع معلومات PHP، بما فيها:

```
SECRET_KEY = gd7s9f8asdf7as8df9as8df7asd8f
```

### الخطوة 7: إرسال الإجابة

- ارجع إلى صفحة المختبر
- اضغط **Submit solution**
- الصق قيمة `SECRET_KEY`

### الخطوة 8: حل المختبر

بعد إرسال المفتاح الصحيح، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي صفحة phpinfo()؟** | صفحة PHP تعرض معلومات كاملة عن الخادم والإصدارات والمتغيرات البيئية |
| **كيف اكتشفناها؟** | من خلال تعليق HTML في الصفحة الرئيسية |
| **ما هي المعلومات المسربة؟** | `SECRET_KEY` (مفاتيح سرية) وإصدارات البرامج ومسار الملفات |
| **لماذا هذا خطير؟** | المفاتيح السرية يمكن استخدامها لاختراق الجلسات أو تزوير التوكنات |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

وجود ملف `phpinfo.php` أو أي صفحة تصحيح أخطاء في بيئة الإنتاج (production) يمكن الوصول إليها من قبل أي شخص.

---

### ❌ Non-Compliant Code (Configuration)

```php
// ملف: phpinfo.php - موجود في بيئة الإنتاج
<?php
// خطأ: عرض جميع معلومات PHP لأي شخص
phpinfo();
?>
```

```nginx
# إعدادات Nginx - لا تمنع الوصول إلى phpinfo.php
location ~ \.php$ {
    include fastcgi_params;
    fastcgi_pass php-fpm;
}
```

---

### ✅ Compliant Code (Configuration)

```php
// ملف: phpinfo.php - إما حذفه أو حمايته
<?php
// التصحيح 1: حماية بكلمة مرور
$valid_user = 'admin';
$valid_password = 'complex_password_hash';

if (!isset($_SERVER['PHP_AUTH_USER']) || 
    $_SERVER['PHP_AUTH_USER'] !== $valid_user || 
    $_SERVER['PHP_AUTH_PW'] !== $valid_password) {
    header('WWW-Authenticate: Basic realm="Admin"');
    header('HTTP/1.0 401 Unauthorized');
    echo 'Access denied';
    exit;
}

// التصحيح 2: السماح فقط من IPs معينة
$allowed_ips = ['127.0.0.1', '192.168.1.100'];
if (!in_array($_SERVER['REMOTE_ADDR'], $allowed_ips)) {
    die('Access denied');
}

phpinfo();
?>
```

```nginx
# إعدادات Nginx - منع الوصول إلى phpinfo.php
location ~ /(phpinfo|debug)\.php$ {
    allow 127.0.0.1;
    deny all;
    return 403;
}
```

```javascript
// في Next.js، لا تترك صفحات تصحيح أخطاء في الإنتاج
// إذا احتجت debug page، استخدم بيئة تطوير فقط

// next.config.js
module.exports = {
  // فقط في بيئة التطوير
  ...(process.env.NODE_ENV === 'development' && {
    rewrites: async () => [
      {
        source: '/debug',
        destination: '/api/debug',
      },
    ],
  }),
};
```

**الخلاصة:** 
1. احذف صفحات `phpinfo()` أو صفحات debug من خادم الإنتاج
2. إذا كان لابد من وجودها، قم بحمايتها بكلمة مرور أو قيود IP
3. استخدم متغيرات البيئة لتحديد بيئة التشغيل

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **لا تترك صفحات debug في الإنتاج:** احذف `phpinfo()` وملفات الاختبار.
2. **استخدم متغيرات البيئة:** `NODE_ENV=production` لتعطيل ميزات debug.
3. **حماية صفحات الإدارة:** استخدم Basic Auth أو قيود IP للمسارات الحساسة.
4. **مراجعة التعليقات:** لا تترك تعليقات تحتوي على روابط داخلية في الصفحات العامة.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/information-disclosure/lab-information-disclosure-on-debug-page)
- [PHP phpinfo() Security](https://www.php.net/manual/en/function.phpinfo.php)
- [Information Disclosure Cheat Sheet](https://portswigger.net/web-security/information-disclosure)
