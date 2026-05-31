# Lab: Multi-step process with no access control on one step

> **Category:** Access Control  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Multi-step process with no access control on one step](https://portswigger.net/web-security/access-control/lab-multi-step-process-with-no-access-control-on-one-step)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة التحكم في الوصول (Access Control) في عملية **متعددة الخطوات**. الخطوة الأولى قد تكون محمية، لكن الخطوة الثانية (مثل تأكيد الترقية) **غير محمية** ويمكن تنفيذها مباشرة.

**المعطيات:**
- حساب المدير: `administrator:admin` (لمعرفة كيف تعمل العملية)
- حسابنا: `wiener:peter`

**المطلوب:** ترقية حساب `wiener` ليصبح مديراً (administrator).

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول بحساب المدير (للفهم)

- سجل الدخول بـ `administrator:admin`
- اذهب إلى لوحة التحكم `/admin`
- لاحظ أن ترقية مستخدم تتطلب **خطوتين**:
  1. اختيار المستخدم ورفع الصلاحية (POST /admin-roles)
  2. تأكيد العملية (GET /admin-roles?confirm=true)

### الخطوة 2: اعتراض طلب التأكيد في Burp

- قم بترقية مستخدم (مثل `carlos`)
- عندما تظهر صفحة التأكيد، اعترض طلب التأكيد:

```http
GET /admin-roles?username=carlos&action=upgrade&confirm=true HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=ADMIN_SESSION_COOKIE
```

- أرسل هذا الطلب إلى **Repeater**

### الخطوة 3: تسجيل الدخول بحساب wiener

- افتح نافذة خاصة (Incognito/Private)
- سجل الدخول بـ `wiener:peter`

### الخطوة 4: استبدال كوكي الجلسة

- في Repeater، استبدل كوكي الجلسة (session) بـ **كوكي wiener**
- غيّر `username=carlos` إلى `username=wiener`

يصبح الطلب:

```http
GET /admin-roles?username=wiener&action=upgrade&confirm=true HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=WIENER_SESSION_COOKIE
```

### الخطوة 5: إرسال الطلب

- اضغط **Send**

**النتيجة:** تم ترقية `wiener` إلى مدير ✅

### الخطوة 6: التحقق من الصلاحية

- في المتصفح (حساب wiener)، اذهب إلى `/admin`
- ستظهر لوحة التحكم ✅

### الخطوة 7: حل المختبر

بمجرد الوصول إلى لوحة التحكم (أو ترقية نفسك)، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **أين الثغرة؟** | الخطوة الأولى (اختيار المستخدم) محمية، لكن الخطوة الثانية (التأكيد) غير محمية |
| **كيف نستغلها؟** | نتخطى الخطوة الأولى ونرسل طلب التأكيد مباشرة مع اسم مستخدمنا |
| **لماذا نجحت؟** | الخادم يفترض أن طلب التأكيد يأتي بعد خطوة الترقية، لكنه لا يتحقق من ذلك |
| **ما هو Multi-step process؟** | عملية تتطلب عدة خطوات متتالية (مثل اختيار → تأكيد) |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

العملية متعددة الخطوات تفترض أن الخطوات تأتي بالترتيب الصحيح، ولكن لا يوجد **تحقق من حالة الجلسة** لكل خطوة على حدة.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/admin-roles.js
const upgradeState = new Map(); // sessionId -> { username, step }

// الخطوة الأولى: اختيار المستخدم
export default function handler(req, res) {
  const { sessionId } = req.cookies;
  const currentUser = getUserBySession(sessionId);
  
  if (!currentUser || currentUser.role !== 'admin') {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  if (req.method === 'POST') {
    const { username, action } = req.body;
    
    if (action === 'upgrade') {
      // تخزين أن هذه الجلسة في خطوة الترقية
      upgradeState.set(sessionId, { username, step: 'awaiting-confirm' });
      return res.redirect(`/admin-roles?username=${username}&action=upgrade&confirm=true`);
    }
  }
  
  // خطأ: خطوة التأكيد لا تتحقق من الصلاحية!
  if (req.method === 'GET' && req.query.confirm === 'true') {
    const { username, action } = req.query;
    
    // يفترض أن المستخدم هو مدير، لكن لا يتحقق!
    if (action === 'upgrade') {
      upgradeUser(username); // أي شخص يمكنه الوصول هنا!
      return res.json({ success: true });
    }
  }
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/admin-roles.js
const upgradeState = new Map(); // sessionId -> { username, step, timestamp }

export default function handler(req, res) {
  const { sessionId } = req.cookies;
  const currentUser = getUserBySession(sessionId);
  
  // التصحيح 1: التحقق من الصلاحية أولاً
  if (!currentUser || currentUser.role !== 'admin') {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  if (req.method === 'POST') {
    const { username, action } = req.body;
    
    if (action === 'upgrade') {
      // تخزين حالة الترقية مع وقت انتهاء
      upgradeState.set(sessionId, { 
        username, 
        step: 'awaiting-confirm',
        timestamp: Date.now()
      });
      return res.redirect(`/admin-roles?username=${username}&action=upgrade&confirm=true`);
    }
  }
  
  // التصحيح 2: التأكد من وجود حالة صالحة قبل التأكيد
  if (req.method === 'GET' && req.query.confirm === 'true') {
    const { username, action } = req.query;
    
    // التحقق من وجود حالة ترقية صالحة لهذه الجلسة
    const pendingUpgrade = upgradeState.get(sessionId);
    
    if (!pendingUpgrade || 
        pendingUpgrade.username !== username || 
        pendingUpgrade.step !== 'awaiting-confirm') {
      return res.status(403).json({ error: 'Invalid upgrade flow' });
    }
    
    // التحقق من انتهاء الوقت (مثلاً 5 دقائق)
    if (Date.now() - pendingUpgrade.timestamp > 5 * 60 * 1000) {
      upgradeState.delete(sessionId);
      return res.status(403).json({ error: 'Upgrade expired' });
    }
    
    if (action === 'upgrade') {
      upgradeUser(username);
      upgradeState.delete(sessionId);
      return res.json({ success: true });
    }
  }
}
```

**الخلاصة:** 
1. كل خطوة في العملية متعددة الخطوات يجب أن تتحقق من الصلاحية
2. استخدم حالة جلسة (session state) لربط الخطوات ببعضها
3. لا تفترض أن الخطوات تأتي بالترتيب الصحيح

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **التحقق من الصلاحية في كل خطوة:** لا تفترض أن المستخدم اجتاز الخطوات السابقة.
2. **استخدم state tracking:** خزّن حالة العملية في الجلسة على الخادم.
3. **اربط الخطوات ببعضها:** تأكد من أن الطلب الحالي يأتي بعد خطوة صالحة سابقة.
4. **استخدم CSRF tokens:** في كل خطوة من العملية.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/access-control/lab-multi-step-process-with-no-access-control-on-one-step)
- [Access Control Cheat Sheet](https://portswigger.net/web-security/access-control)
