# Lab: Blind SSRF with out-of-band detection

> **Category:** SSRF  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Blind SSRF with out-of-band detection](https://portswigger.net/web-security/ssrf/blind/lab-out-of-band-detection)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة **Blind SSRF** حيث لا يعيد الخادم نتيجة الطلب في الرد (response)، ولكن يمكننا اكتشافها باستخدام **Burp Collaborator** لمراقبة الطلبات الصادرة. التطبيق يستخدم برنامج تحليلات (analytics) يجلب الرابط الموجود في هيدر `Referer`.

**المطلوب:** إرسال طلب HTTP إلى خادم Burp Collaborator وإثبات حدوثه.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فتح صفحة منتج

- اذهب إلى أي منتج في المتجر
- اعترض الطلب في Burp

### الخطوة 2: إرسال الطلب إلى Repeater

- زر الماوس الأيمن → **Send to Repeater**

الطلب يبدو هكذا:

```http
GET /product?productId=1 HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Referer: https://YOUR-LAB-ID.web-security-academy.net/
```

### الخطوة 3: إدراج Collaborator Payload في Referer

- في Repeater، حدد قيمة هيدر `Referer`
- زر الماوس الأيمن → **Insert Collaborator Payload**

سيتم استبدال النطاق بـ:

```
Referer: https://YOUR-COLLABORATOR-PAYLOAD.oastify.com/
```

### الخطوة 4: إرسال الطلب

- اضغط **Send**

### الخطوة 5: فحص Collaborator

- اذهب إلى **Burp** → **Collaborator** tab
- اضغط **Poll now**

### الخطوة 6: انتظار النتائج

إذا لم تظهر تفاعلات (interactions) فوراً، انتظر بضع ثوانٍ ثم اضغط **Poll now** مرة أخرى.

**النتيجة المتوقعة:** ظهور DNS و HTTP interactions من الخادم:

```
DNS lookup: YOUR-COLLABORATOR-PAYLOAD.oastify.com
HTTP request: YOUR-COLLABORATOR-PAYLOAD.oastify.com
```

### الخطوة 7: حل المختبر

بمجرد رؤية التفاعلات (interactions) في Collaborator، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي Blind SSRF؟** | ثغرة SSRF لا يعاد فيها محتوى الرد إلى العميل، ولكن الطلب يحدث في الخلفية |
| **كيف نكتشفها؟** | باستخدام Burp Collaborator لمراقبة الطلبات الصادرة من الخادم |
| **لماذا نستخدم Referer؟** | التطبيق يقرأ رابط التحليلات من هيدر Referer |
| **ماذا يفعل Collaborator؟** | يوفر خادماً خارجياً يراقب DNS و HTTP requests |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

التطبيق يقرأ رابط التحليلات من هيدر `Referer` (الذي يمكن التحكم به من قبل المستخدم) ويقوم بإرسال طلب HTTP إليه دون تحقق.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/product.js - تحميل صفحة المنتج
export default function productHandler(req, res) {
  const { productId } = req.query;
  const referer = req.headers.referer;
  
  // خطأ: إرسال طلب إلى الرابط الموجود في Referer دون تحقق
  if (referer) {
    // إرسال طلب غير متزامن (async) - لا ينتظر الرد
    fetch(referer).catch(() => {});
  }
  
  // عرض صفحة المنتج
  res.send(productPage);
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/product.js - تحميل صفحة المنتج
export default function productHandler(req, res) {
  const { productId } = req.query;
  const referer = req.headers.referer;
  
  // التصحيح 1: التحقق من وجود Referer وكونه من نفس النطاق
  if (referer) {
    try {
      const refererUrl = new URL(referer);
      const allowedHost = req.headers.host;
      
      if (refererUrl.hostname !== allowedHost) {
        // التصحيح 2: تجاهل Referer من نطاقات خارجية
        console.warn(`Blocked external referer: ${referer}`);
        // لا ترسل أي طلب
      } else {
        // التصحيح 3: فقط إذا كان النطاق موثوقاً، قم بإرسال الطلب
        // مع إضافة timeout و limit
        const controller = new AbortController();
        const timeoutId = setTimeout(() => controller.abort(), 3000);
        
        fetch(referer, { signal: controller.signal })
          .catch(() => {})
          .finally(() => clearTimeout(timeoutId));
      }
    } catch {
      // عنوان غير صالح
    }
  }
  
  res.send(productPage);
}
```

**الخلاصة:** 
1. لا تثق في هيدرات يمكن التحكم بها من قبل المستخدم (`Referer`, `Origin`, `Host`)
2. تحقق من أن الرابط من نفس النطاق قبل إرسال الطلب
3. استخدم timeouts لمنع الطلبات البطيئة

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **التحقق من النطاق** | تأكد أن الرابط من نفس النطاق (same-origin) |
| **قائمة بيضاء (Allowlist)** | حدد النطاقات المسموح بها للطلبات الصادرة |
| **إزالة الهيدرات غير الضرورية** | لا تقرأ `Referer` إذا لم تكن بحاجة إليه |
| **استخدام timeouts** | حدد مهلة زمنية للطلبات الصادرة |
| **مراقبة الطلبات** | سجل الطلبات الصادرة للكشف عن السلوك المشبوه |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/ssrf/blind/lab-out-of-band-detection)
- [Burp Collaborator](https://portswigger.net/burp/documentation/desktop/tools/collaborator)
- [SSRF Cheat Sheet](https://portswigger.net/web-security/ssrf)
