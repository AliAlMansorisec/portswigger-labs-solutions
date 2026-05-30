# Lab: User ID controlled by request parameter

> **Category:** Access Control  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - User ID controlled by request parameter](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة **Horizontal Privilege Escalation** (تسجيل الصلاحيات الأفقي) في صفحة حساب المستخدم. يمكننا الوصول إلى بيانات مستخدم آخر (carlos) بمجرد تغيير معامل `id` في الرابط.

**الحساب:** `wiener:peter`

**المطلوب:** الحصول على API key للمستخدم `carlos` وإرسالها.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول

- سجل الدخول بـ `wiener:peter`
- اذهب إلى **My account**

### الخطوة 2: ملاحظة الرابط

لاحظ أن رابط صفحة الحساب يحتوي على معامل `id`:

```
https://YOUR-LAB-ID.web-security-academy.net/my-account?id=wiener
```

### الخطوة 3: اختبار تغيير الـ id

- غيّر قيمة `id` في الرابط من `wiener` إلى `carlos`:

```
https://YOUR-LAB-ID.web-security-academy.net/my-account?id=carlos
```

### الخطوة 4: عرض بيانات carlos

ستظهر صفحة حساب المستخدم `carlos` تحتوي على:

```
API Key: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### الخطوة 5: إرسال API key

- انسخ الـ API key
- اذهب إلى صفحة المختبر
- الصق الـ API key في حقل الإجابة (Submit solution)

### الخطوة 6: حل المختبر

بعد إرسال الـ API key الصحيح، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي Horizontal Privilege Escalation؟** | الوصول إلى بيانات مستخدم آخر بنفس مستوى الصلاحية |
| **أين الثغرة؟** | الخادم لا يتحقق من أن المستخدم الحالي هو صاحب البيانات |
| **كيف نستغلها؟** | نغير `id` من `wiener` إلى `carlos` |
| **الحل الصحيح؟** | التحقق من أن `id` المطلوب يخص المستخدم الحالي |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يثق بمعامل `id` من المستخدم ويعرض بيانات أي مستخدم آخر بدون التحقق من الصلاحية.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/my-account.js
export default function AccountPage({ user }) {
  return (
    <div>
      <h1>My Account</h1>
      <p>API Key: {user.apiKey}</p>
    </div>
  );
}

export async function getServerSideProps(context) {
  const { id } = context.query;
  const { sessionId } = context.req.cookies;
  
  // خطأ: لا يتحقق من أن id يخص المستخدم الحالي
  const user = getUserById(id);
  
  return { props: { user } };
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/my-account.js
import { getSession } from 'next-auth/react';

export default function AccountPage({ user }) {
  return (
    <div>
      <h1>My Account</h1>
      <p>API Key: {user.apiKey}</p>
    </div>
  );
}

export async function getServerSideProps(context) {
  const { id } = context.query;
  const session = await getSession(context);
  
  // التصحيح: التحقق من أن id المطلوب يخص المستخدم الحالي
  if (!session || session.user.id !== id) {
    return { redirect: { destination: '/login', permanent: false } };
  }
  
  const user = getUserById(id);
  return { props: { user } };
}
```

**الخلاصة:** 
1. لا تثق بمعامل `id` من المستخدم
2. تحقق من أن المستخدم الحالي هو صاحب البيانات المطلوبة
3. استخدم الجلسة لتحديد هوية المستخدم، وليس الـ `id` من الرابط

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **استخدام الجلسة لتحديد المستخدم:** لا تعتمد على معامل `id` في الرابط.
2. **التحقق من الصلاحية:** تأكد أن المستخدم الحالي يملك صلاحية الوصول إلى البيانات المطلوبة.
3. **استخدام UUIDs غير قابلة للتخمين:** بدلاً من الأسماء أو الأرقام المتسلسلة.
4. **لا تعرض بيانات حساسة في الرابط:** استخدم POST أو session بدلاً من GET.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter)
- [Access Control Cheat Sheet](https://portswigger.net/web-security/access-control)
- [Horizontal Privilege Escalation](https://portswigger.net/web-security/access-control#horizontal-privilege-escalation)
