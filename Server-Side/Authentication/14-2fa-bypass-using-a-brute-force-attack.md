# Lab: 2FA bypass using a brute-force attack

> **Category:** Authentication  
> **Difficulty:** EXPERT  
> **Lab Link:** [PortSwigger Lab - 2FA bypass using a brute-force attack](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-brute-force)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة في التحقق بخطوتين (2FA) حيث يمكن كسر رمز التحقق المكون من 4 أرقام (0000-9999) بالقوة الغاشمة (Brute-Force). بعد محاولتين خاطئتين، يتم تسجيل الخروج تلقائياً، لذلك نحتاج إلى **ماكرو (Macro)** لإعادة تسجيل الدخول تلقائياً قبل كل محاولة.

**المعطيات:**
- حساب الضحية: `carlos:montoya`

**المطلوب:** العثور على رمز 2FA الصحيح والوصول إلى صفحة حساب `carlos`.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم آلية 2FA

- سجل الدخول بـ `carlos:montoya`
- سيُطلب منك رمز 2FA المكون من 4 أرقام
- **ملاحظة:** بعد محاولتين خاطئتين، سيتم تسجيل الخروج تلقائياً

### الخطوة 2: إعداد Session Handling Rule مع Macro

#### 2.1 فتح إعدادات Session Handling

- في Burp → **Settings** → **Sessions** → **Session Handling Rules** → **Add**

#### 2.2 إعداد النطاق (Scope)

- **Scope** tab
- **URL Scope** → **Include all URLs**

#### 2.3 إضافة Macro

- **Details** tab → **Rule Actions** → **Add** → **Run a macro**
- **Select macro** → **Add**

#### 2.4 اختيار الطلبات في Macro Recorder

اختر الطلبات التالية **بالترتيب**:

```
1. GET /login
2. POST /login (username=carlos, password=montoya)
3. GET /login2
```

#### 2.5 اختبار الـ Macro

- **Test macro**
- تأكد أن الرد الأخير يحتوي على صفحة طلب رمز 2FA

#### 2.6 حفظ الإعدادات

- اضغط **OK** حتى العودة إلى النافذة الرئيسية

### الخطوة 3: إرسال طلب 2FA إلى Intruder

- اعترض طلب `POST /login2` (طلب إدخال رمز 2FA)
- أرسله إلى **Intruder**

```http
POST /login2 HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

mfa-code=1234
```

### الخطوة 4: إعداد Payloads

- ضع علامات `§` حول `mfa-code`:

```
mfa-code=§1234§
```

- **Payload type:** Numbers
- **From:** `0`
- **To:** `9999`
- **Step:** `1`
- **Min integer digits:** `4`
- **Max fraction digits:** `0`

### الخطوة 5: إعداد Resource Pool

- **Resource Pool** tab
- **Maximum concurrent requests:** `1` (هام جداً!)

### الخطوة 6: بدء الهجوم

- اضغط **Start attack**

سيقوم Intruder بتجربة جميع الرموز من 0000 إلى 9999، وسيعيد الـ Macro تسجيل الدخول تلقائياً بعد كل محاولتين.

### الخطوة 7: تحليل النتائج

- ابحث عن طلب برمز **302 Found** (إعادة توجيه)

| رمز 2FA | Status | النتيجة |
|---------|--------|---------|
| 0001 | 200 | ❌ فشل |
| 0002 | 200 | ❌ فشل |
| ... | ... | ... |
| **8472** | **302** | ✅ نجاح |

### الخطوة 8: عرض الرد في المتصفح

- زر الماوس الأيمن على الطلب الناجح → **Show response in browser**
- انسخ الرابط
- الصقه في المتصفح

### الخطوة 9: حل المختبر

- اضغط **My account** للوصول إلى صفحة حساب `carlos`
- سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **لماذا نحتاج Macro؟** | بعد محاولتين خاطئتين، يتم تسجيل الخروج، والـ Macro يعيد تسجيل الدخول تلقائياً |
| **ما هي الطلبات في Macro؟** | `GET /login` → `POST /login` → `GET /login2` |
| **لماذا `concurrent requests = 1`؟** | لضمان ترتيب الطلبات الصحيح ومنع اختلاط الجلسات |
| **كم عدد المحاولات؟** | 10000 محاولة (0000 إلى 9999) |
| **لماذا قد نكرر الهجوم؟** | إذا تغير الرمز أثناء الهجوم، نحتاج إلى إعادة تشغيله |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

1. رمز 2FA مكون من 4 أرقام فقط (10000 احتمال) يمكن كسره بالقوة الغاشمة
2. عدم وجود حماية لقفل الحساب بعد محاولات فاشلة (باستثناء تسجيل الخروج)
3. يمكن أتمتة عملية إعادة تسجيل الدخول

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/login2.js (POST - التحقق من رمز 2FA)
const twoFACodes = new Map(); // sessionId -> { code, attempts }

export default function handler(req, res) {
  const sessionId = req.cookies.session;
  const { 'mfa-code': code } = req.body;
  
  const record = twoFACodes.get(sessionId);
  
  if (!record) {
    return res.redirect('/login');
  }
  
  // خطأ: رمز 2FA مكون من 4 أرقام فقط
  // خطأ: بعد محاولتين خاطئتين يتم تسجيل الخروج فقط (بدون حظر)
  if (record.attempts >= 2) {
    twoFACodes.delete(sessionId);
    return res.redirect('/login');
  }
  
  if (code !== record.code) {
    record.attempts++;
    twoFACodes.set(sessionId, record);
    return res.status(200).json({ error: 'Invalid code' });
  }
  
  // تسجيل الدخول الناجح
  twoFACodes.delete(sessionId);
  req.session.is2FAVerified = true;
  res.redirect('/my-account');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/login2.js (POST - التحقق من رمز 2FA)
const twoFACodes = new Map(); // sessionId -> { code, attempts, lockUntil }

export default function handler(req, res) {
  const sessionId = req.cookies.session;
  const { 'mfa-code': code } = req.body;
  
  const record = twoFACodes.get(sessionId);
  
  if (!record) {
    return res.redirect('/login');
  }
  
  // التصحيح 1: قفل الحساب بعد 3 محاولات فاشلة
  if (record.lockUntil && record.lockUntil > Date.now()) {
    return res.status(200).json({ error: 'Too many attempts. Try again later.' });
  }
  
  if (code !== record.code) {
    record.attempts++;
    
    // التصحيح 2: قفل لمدة 15 دقيقة بعد 3 محاولات
    if (record.attempts >= 3) {
      record.lockUntil = Date.now() + 15 * 60 * 1000;
    }
    
    twoFACodes.set(sessionId, record);
    return res.status(200).json({ error: 'Invalid code' });
  }
  
  // التصحيح 3: رمز 2FA مكون من 6 أرقام (1,000,000 احتمال)
  // أو 8 أرقام (100,000,000 احتمال)
  
  twoFACodes.delete(sessionId);
  req.session.is2FAVerified = true;
  res.redirect('/my-account');
}
```

```javascript
// pages/api/login2.js (GET - إنشاء رمز 2FA)
import crypto from 'crypto';

export default function handler(req, res) {
  const sessionId = req.cookies.session;
  const session = getSession(sessionId);
  
  if (!session || !session.pending2FA) {
    return res.redirect('/login');
  }
  
  // التصحيح 4: رمز 2FA مكون من 6 أرقام (أكثر أماناً)
  const code = crypto.randomInt(100000, 999999).toString();
  const expiresAt = Date.now() + 5 * 60 * 1000; // 5 دقائق
  
  twoFACodes.set(sessionId, {
    code,
    attempts: 0,
    lockUntil: 0,
    expiresAt
  });
  
  sendEmail(session.user.email, `Your 2FA code is: ${code}`);
  res.render('2fa-page');
}
```

**الخلاصة:** 
1. استخدم رمز 2FA مكون من 6 أرقام على الأقل (1,000,000 احتمال)
2. قفل الحساب بعد 3 محاولات فاشلة (لمدة 15 دقيقة)
3. صلاحية زمنية قصيرة للرمز (5 دقائق)
4. استخدام `crypto.randomInt()` لتوليد رموز آمنة

---

## 🛡️ كيفية الوقاية (How to Prevent)

| الإجراء | الوصف |
|---------|-------|
| **زيادة طول الرمز** | استخدم 6-8 أرقام بدلاً من 4 (1,000,000 - 100,000,000 احتمال) |
| **قفل الحساب (Account Lockout)** | قفل الحساب بعد 3-5 محاولات فاشلة |
| **صلاحية زمنية قصيرة** | اجعل الرمز صالحاً لمدة 5 دقائق فقط |
| **معدل محدود للمحاولات (Rate Limiting)** | حدد عدد الطلبات لكل جلسة |
| **إعادة تعيين الجلسة بعد الفشل** | لا تسمح باستمرار الجلسة بعد المحاولات الفاشلة |

---
# 📊 ملخص لابات Authentication

| # | اللاب | الصعوبة | نوع الثغرة | طريقة الاستغلال |
|---|-------|---------|-----------|----------------|
| 01 | Username enumeration via different responses | APPRENTICE | رسائل خطأ مختلفة | استخدام اختلاف طول الرد (Length) لتعداد الأسماء |
| 02 | 2FA simple bypass | APPRENTICE | تخطي 2FA | الانتقال مباشرة إلى `/my-account` بعد تسجيل الدخول |
| 03 | Password reset broken logic | APPRENTICE | منطق إعادة تعيين كلمة المرور | تجاهل رمز إعادة التعيين وتغيير كلمة مرور أي مستخدم |
| 04 | Username enumeration via subtly different responses | PRACTITIONER | اختلاف طفيف في الرسائل | استخدام Grep-Extract لالتقاط مسافة/نقطة مختلفة |
| 05 | Username enumeration via response timing | PRACTITIONER | توقيت الاستجابة | استخدام كلمة مرور طويلة + هيدر `X-Forwarded-For` |
| 06 | Broken brute-force protection, IP block | PRACTITIONER | حماية IP معطوبة | إعادة تعيين العداد بتسجيل دخول ناجح من حساب آخر |
| 07 | Username enumeration via account lock | PRACTITIONER | قفل الحساب | إرسال 5 محاولات فاشلة لكل اسم لمعرفة المقفل |
| 08 | 2FA broken logic | PRACTITIONER | منطق 2FA معطوب | إنشاء رمز 2FA لـ carlos عبر `GET /login2?verify=carlos` |
| 09 | Brute-forcing a stay-logged-in cookie | PRACTITIONER | كوكي قابل للتخمين | استخدام `base64(username:MD5(password))` مع Intruder |
| 10 | Offline password cracking | PRACTITIONER | XSS + MD5 | سرقة كوكي `stay-logged-in` وكسر MD5 |
| 11 | Password reset poisoning via middleware | PRACTITIONER | تسميم إعادة التعيين | استخدام `X-Forwarded-Host` لتوجيه الرابط إلى خادم المهاجم |
| 12 | Password brute-force via password change | PRACTITIONER | تغيير كلمة المرور | استخدام `new-password-1 != new-password-2` لتجنب القفل |
| 13 | Broken brute-force protection, multiple credentials per request | EXPERT | مصفوفة كلمات المرور | إرسال مصفوفة JSON تحتوي على جميع كلمات المرور في طلب واحد |
| 14 | 2FA bypass using a brute-force attack | EXPERT | كسر رمز 2FA | Macro + Intruder لاختبار 10000 رمز (0000-9999) |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-brute-force)
- [2FA Brute-Force](https://portswigger.net/web-security/authentication/multi-factor#2fa-brute-forcing)
- [Burp Macros](https://portswigger.net/burp/documentation/desktop/settings/sessions/macros)
