# Lab: Authentication bypass via flawed state machine

> **Category:** Business Logic  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Authentication bypass via flawed state machine](https://portswigger.net/web-security/logic-flaws/lab-authentication-bypass-via-flawed-state-machine)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة منطق العمل (Business Logic) في **آلة الحالة (state machine)** لعملية تسجيل الدخول. بعد تسجيل الدخول، يجب على المستخدم اختيار دوره (role)، لكن إذا **أسقطنا (drop)** طلب اختيار الدور، سينتقل النظام تلقائياً إلى الصفحة الرئيسية بدور **admin** افتراضي.

**الحساب:** `wiener:peter`

**المطلوب:** الوصول إلى `/admin` وحذف المستخدم `carlos`.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فهم عملية تسجيل الدخول

- سجل الدخول بـ `wiener:peter`
- لاحظ أنه بعد إدخال اسم المستخدم وكلمة المرور، تظهر صفحة **اختيار الدور (role selector)** قبل الانتقال إلى الصفحة الرئيسية

**تسلسل الطلبات العادي:**

```
1. POST /login (إرسال بيانات الدخول)
2. GET /role-selector (اختيار الدور)
3. POST /role-selector (تأكيد الدور)
4. GET / (الصفحة الرئيسية)
```

### الخطوة 2: اكتشاف مسار /admin

- استخدم **Content Discovery** في Burp
- الطريقة: **Target** > **Site map** > زر الماوس الأيمن على النطاق > **Engagement tools** > **Discover content**
- ستجد أن الأداة اكتشفت المسار `/admin`

### الخطوة 3: محاولة الوصول إلى /admin مباشرة (ستفشل)

- من صفحة اختيار الدور، جرب زيارة `/admin` مباشرة
- **النتيجة:** يتم حظر الوصول (غير مصرح)

### الخطوة 4: تسجيل الخروج

- قم بتسجيل الخروج (Log out)

### الخطوة 5: اعتراض الطلبات أثناء تسجيل الدخول

- في Burp، شغّل **Intercept** (اعتراض الطلبات)
- سجل الدخول بـ `wiener:peter` مرة أخرى

### الخطوة 6: تسلسل الطلبات

بعد إرسال بيانات الدخول، سترى الطلبات التالية:

```
1. POST /login (أرسله Forward)
2. GET /role-selector (هذا هو الطلب الذي نريد إسقاطه)
```

### الخطوة 7: إسقاط (Drop) طلب role-selector

- عندما يظهر طلب `GET /role-selector` في Burp:
  - **لا ترسله (Do NOT forward)**
  - اضغط **Drop** لإسقاطه

### الخطوة 8: الوصول إلى الصفحة الرئيسية

- بعد إسقاط طلب `role-selector`، اذهب إلى الصفحة الرئيسية مباشرة:

```
https://YOUR-LAB-ID.web-security-academy.net/
```

### الخطوة 9: ملاحظة النتيجة

- **النتيجة:** تم نقلك إلى الصفحة الرئيسية **بدور admin** افتراضي! ✅
- لاحظ أنك الآن تمتلك صلاحيات المدير

### الخطوة 10: الوصول إلى /admin وحذف carlos

- اذهب إلى `/admin`:

```
https://YOUR-LAB-ID.web-security-academy.net/admin
```

- ستظهر لوحة التحكم
- اضغط على زر **Delete** بجانب `carlos`

### الخطوة 11: حل المختبر

بعد حذف `carlos`، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي آلة الحالة (State Machine)؟** | سلسلة من الحالات التي يمر بها المستخدم أثناء عملية تسجيل الدخول (login → role selection → home) |
| **أين الثغرة؟** | الخادم يفترض أن المستخدم سيكمل عملية اختيار الدور، وإذا لم يفعل، ينتقل إلى حالة افتراضية غير آمنة (admin) |
| **كيف نستغلها؟** | نُسقط (drop) طلب اختيار الدور، فينتقل النظام إلى الصفحة الرئيسية بدور admin |
| **لماذا هذا خطأ؟** | الحالة الافتراضية بعد التخطي يجب أن تكون مستخدم عادي، وليس مدير |

---

## 📊 تسلسل العملية الطبيعية مقابل الهجوم

| الخطوة | العملية الطبيعية | الهجوم |
|--------|------------------|--------|
| 1 | POST /login (صحيح) | ✅ POST /login |
| 2 | GET /role-selector | ❌ **نُسقط الطلب** |
| 3 | POST /role-selector (اختيار role) | ❌ **لا يحدث** |
| 4 | GET / (الصفحة الرئيسية) | ✅ GET / (يدخل كـ admin!) |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

آلة الحالة (state machine) في عملية تسجيل الدخول تفترض أن المستخدم سيكمل جميع الخطوات. إذا لم يكمل، تنتقل إلى حالة **افتراضية غير آمنة** (admin) بدلاً من حالة افتراضية آمنة (user).

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// صفحات التالي (Next.js) - عملية تسجيل الدخول
const sessionState = new Map(); // sessionId -> { step, role }

// بعد تسجيل الدخول الناجح
export default function loginHandler(req, res) {
  const { username, password } = req.body;
  const user = authenticate(username, password);
  
  if (!user) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  const sessionId = generateSessionId();
  // خطأ: الحالة الأولية هي "role-selection" بدون دور محدد
  sessionState.set(sessionId, { step: 'role-selection', userId: user.id });
  
  res.setHeader('Set-Cookie', `session=${sessionId}`);
  res.redirect('/role-selector');
}

// صفحة role-selector
export default function roleSelectorHandler(req, res) {
  const { sessionId } = req.cookies;
  const state = sessionState.get(sessionId);
  
  if (!state || state.step !== 'role-selection') {
    return res.redirect('/');
  }
  
  // عرض صفحة اختيار الدور
  res.send(roleSelectorPage);
}

// بعد اختيار الدور
export default function setRoleHandler(req, res) {
  const { sessionId } = req.cookies;
  const { role } = req.body;
  const state = sessionState.get(sessionId);
  
  if (state && state.step === 'role-selection') {
    state.step = 'complete';
    state.role = role; // 'user' أو 'admin'
    sessionState.set(sessionId, state);
  }
  
  res.redirect('/');
}

// الصفحة الرئيسية - التحقق من الصلاحية
export default function homeHandler(req, res) {
  const { sessionId } = req.cookies;
  const state = sessionState.get(sessionId);
  
  // خطأ: إذا لم تكن الحالة "complete"، يمنح دور admin افتراضي!
  if (!state || state.step !== 'complete') {
    // خطأ فادح: الحالة الافتراضية هي admin
    const user = getUserById(state?.userId);
    if (user?.username === 'wiener') {
      // يعطي صلاحيات admin للمستخدم wiener!
    }
  }
  
  // عرض الصفحة الرئيسية
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// صفحات التالي (Next.js) - عملية تسجيل الدخول الآمنة
const sessionState = new Map(); // sessionId -> { step, role, userId }

// بعد تسجيل الدخول الناجح
export default function loginHandler(req, res) {
  const { username, password } = req.body;
  const user = authenticate(username, password);
  
  if (!user) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  const sessionId = generateSessionId();
  // التصحيح: الحالة الأولية مع دور افتراضي آمن (user)
  sessionState.set(sessionId, { 
    step: 'role-selection', 
    userId: user.id,
    role: 'user' // دور افتراضي آمن
  });
  
  res.setHeader('Set-Cookie', `session=${sessionId}`);
  res.redirect('/role-selector');
}

// الصفحة الرئيسية - التحقق من الصلاحية
export default function homeHandler(req, res) {
  const { sessionId } = req.cookies;
  const state = sessionState.get(sessionId);
  
  // التصحيح: التحقق من وجود حالة صالحة ودور محدد
  if (!state || state.step !== 'complete') {
    // إذا لم تكتمل العملية، اعرض خطأ أو سجل خروج
    return res.redirect('/login');
  }
  
  // استخدام الدور المخزن في الحالة
  const user = getUserById(state.userId);
  const isAdmin = (state.role === 'admin') && (user.role === 'admin');
  
  // عرض الصفحة الرئيسية مع الصلاحيات المناسبة
  res.send(homePage(isAdmin));
}

// صلاحية زمنية للحالة (timeout)
setTimeout(() => {
  for (const [sessionId, state] of sessionState) {
    if (state.step !== 'complete' && Date.now() - state.createdAt > 5 * 60 * 1000) {
      sessionState.delete(sessionId);
    }
  }
}, 60000);
```

**الخلاصة:** 
1. الحالة الافتراضية (fallback) يجب أن تكون **user** وليس **admin**
2. إذا لم تكتمل العملية، أعد التوجيه إلى login وليس إلى home
3. لا تمنح صلاحيات إدارية بدون تحقق صريح

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **حالة افتراضية آمنة:** دائماً افترض أقل الصلاحيات (default-deny).
2. **تحقق من اكتمال العملية:** لا تسمح بالوصول إلى الصفحات المحمية إلا بعد اكتمال جميع الخطوات.
3. **صلاحية زمنية للحالة:** اجعل حالة الجلسة تنتهي بعد فترة قصيرة.
4. **لا تعتمد على الترتيب فقط:** استخدم توكنات أو حالة مخزنة على الخادم.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/logic-flaws/lab-authentication-bypass-via-flawed-state-machine)
- [Business Logic Vulnerabilities](https://portswigger.net/web-security/logic-flaws)
- [State Machine Flaws](https://portswigger.net/web-security/logic-flaws/examples#flawed-state-machines)
