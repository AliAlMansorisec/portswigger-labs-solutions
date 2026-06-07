# Lab: SSRF with blacklist-based input filter

> **Category:** SSRF  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - SSRF with blacklist-based input filter](https://portswigger.net/web-security/ssrf/lab-ssrf-with-blacklist-filter)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة SSRF (Server-Side Request Forgery) في ميزة فحص المخزون (stock check). المطور قام بنشر حماية ضعيفة تعتمد على **قائمة سوداء (blacklist)** تمنع بعض العناوين. سنقوم بتجاوز هذه القائمة باستخدام تقنيات التشفير المختلفة.

**المطلوب:** الوصول إلى `http://localhost/admin` وحذف المستخدم `carlos`.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم ميزة فحص المخزون

- اذهب إلى أي منتج في المتجر
- اضغط على زر **Check stock**
- اعترض طلب `POST /product/stock` في Burp

### الخطوة 2: إرسال الطلب إلى Repeater

- زر الماوس الأيمن → **Send to Repeater**

الطلب الأصلي:

```http
POST /product/stock HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

stockApi=http://192.168.0.1/stock
```

### الخطوة 3: محاولة الوصول إلى localhost (ستفشل)

- غيّر `stockApi` إلى `http://127.0.0.1/admin`

**النتيجة:** يتم حظر الطلب ❌ (القائمة السوداء تمنع `127.0.0.1`)

### الخطوة 4: تجاوز الحظر الأول (اختصار IP)

- استخدم `http://127.1/` بدلاً من `127.0.0.1`:

```http
stockApi=http://127.1/
```

**النتيجة:** ينجح الطلب ✅ (تم تجاوز الحظر)

**شرح:** `127.1` هو اختصار لـ `127.0.0.1` في بعض الأنظمة.

### الخطوة 5: محاولة الوصول إلى /admin (ستفشل مرة أخرى)

- غيّر الرابط إلى `http://127.1/admin`

**النتيجة:** يتم حظر الطلب ❌ (القائمة السوداء تمنع كلمة `admin`)

### الخطوة 6: تجاوز الحظر الثاني (ترميز مزدوج)

- قم بـ **Double URL Encoding** لحرف `a` في كلمة `admin`:
  - `a` → `%61` (ترميز URL عادي)
  - `%61` → `%2561` (ترميز مزدوج)

- استخدم الرابط:

```http
stockApi=http://127.1/%2561dmin
```

أو:

```http
stockApi=http://127.1/%2561dmin/delete?username=carlos
```

### الخطوة 7: إرسال الطلب

- اضغط **Send**

**النتيجة:** تم حذف المستخدم `carlos` ✅

### الخطوة 8: حل المختبر

بعد حذف `carlos`، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو الفلتر المستخدم؟** | قائمة سوداء تمنع `127.0.0.1` وكلمة `admin` |
| **كيف تجاوزنا `127.0.0.1`؟** | باستخدام `127.1` (اختصار IP) |
| **كيف تجاوزنا كلمة `admin`؟** | باستخدام Double URL Encoding (`%2561`) |
| **لماذا Double URL Encoding؟** | الخادم يفك الترميز مرة واحدة، والفلتر يتحقق بعدها، لكننا نُدخل ترميزاً مزدوجاً |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

استخدام **قائمة سوداء (blacklist)** بدلاً من **قائمة بيضاء (whitelist)**، ومعالجة غير كافية للترميزات.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/stock.js
export default function handler(req, res) {
  let { stockApi } = req.body;
  
  // خطأ: استخدام blacklist
  const blacklist = ['127.0.0.1', 'localhost', 'admin', 'delete'];
  
  for (const blocked of blacklist) {
    if (stockApi.includes(blocked)) {
      return res.status(403).send('Blocked');
    }
  }
  
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
  let { stockApi } = req.body;
  
  // التصحيح 1: تطبيع الرابط (normalize) أولاً
  // فك جميع الترميزات
  while (stockApi.includes('%')) {
    stockApi = decodeURIComponent(stockApi);
  }
  
  // التصحيح 2: استخدام whitelist بدلاً من blacklist
  const allowedHosts = ['192.168.0.1', 'internal-api.example.com'];
  const url = new URL(stockApi);
  
  if (!allowedHosts.includes(url.hostname)) {
    return res.status(403).json({ error: 'Access denied' });
  }
  
  // التصحيح 3: منع العناوين المحولة ( مثل 127.1 )
  const resolvedHost = url.hostname;
  const internalIPs = ['127.0.0.1', '127.1', 'localhost', '0.0.0.0', '::1'];
  if (internalIPs.includes(resolvedHost)) {
    return res.status(403).json({ error: 'Internal addresses are not allowed' });
  }
  
  // التصحيح 4: التحقق من المسار
  const allowedPaths = ['/stock', '/inventory'];
  if (!allowedPaths.some(path => url.pathname === path)) {
    return res.status(403).json({ error: 'Access denied' });
  }
  
  const response = await fetch(stockApi);
  const data = await response.text();
  res.send(data);
}
```

**الخلاصة:** 
1. استخدم **قائمة بيضاء (whitelist)** بدلاً من القائمة السوداء
2. قم بتطبيع (normalize) المدخلات قبل التحقق (فك جميع الترميزات)
3. تحقق من العناوين المحولة (مثل `127.1`)

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **Whitelist للنطاقات** | حدد النطاقات المسموحة بدلاً من منع نطاقات معينة |
| **تطبيع المدخلات** | قم بفك جميع الترميزات قبل التحقق |
| **منع العناوين المحولة** | تحقق من `127.1`، `0`, `0.0.0.0` وغيرها |
| **التحقق من المسار** | استخدم whitelist للمسارات المسموحة |
| **استخدام URL parser آمن** | تجنب الأخطاء في parsing |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/ssrf/lab-ssrf-with-blacklist-filter)
- [SSRF Cheat Sheet](https://portswigger.net/web-security/ssrf)
