# Lab: User ID controlled by request parameter with password disclosure

> **Category:** Access Control  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - User ID controlled by request parameter with password disclosure](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-password-disclosure)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة **Horizontal Privilege Escalation** في صفحة حساب المستخدم. صفحة الحساب تحتوي على حقل كلمة المرور **معبأ مسبقاً** (prefilled) بشكل masked، لكن يمكننا تغيير `id` إلى `administrator` لرؤية كلمة مرور المدير.

**الحساب:** `wiener:peter`

**المطلوب:** الحصول على كلمة مرور `administrator`، ثم تسجيل الدخول بها وحذف `carlos`.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول

- سجل الدخول بـ `wiener:peter`
- اذهب إلى **My account**

### الخطوة 2: ملاحظة الرابط

لاحظ أن رابط صفحة الحساب يحتوي على `id`:

```
https://YOUR-LAB-ID.web-security-academy.net/my-account?id=wiener
```

### الخطوة 3: تغيير الـ id إلى administrator

- غيّر `id=wiener` إلى `id=administrator`:

```
https://YOUR-LAB-ID.web-security-academy.net/my-account?id=administrator
```

### الخطوة 4: عرض مصدر الصفحة أو اعتراض الرد في Burp

- **الطريقة 1:** اضغط `Ctrl+U` لعرض مصدر الصفحة
- **الطريقة 2:** اعترض الرد في Burp

ستجد في الصفحة حقل كلمة المرور معبأ مسبقاً:

```html
<input type="password" name="password" value="superSecretPassword123">
```

### الخطوة 5: استخراج كلمة المرور

- انسخ قيمة `value` (هذه هي كلمة مرور `administrator`)

### الخطوة 6: تسجيل الدخول بحساب administrator

- سجل الدخول باستخدام:
  - **Username:** `administrator`
  - **Password:** [كلمة المرور التي حصلت عليها]

### الخطوة 7: حذف المستخدم carlos

- بعد تسجيل الدخول كـ `administrator`
- اذهب إلى لوحة التحكم `/admin`
- اضغط على زر **Delete** بجانب `carlos`

### الخطوة 8: حل المختبر

بعد حذف `carlos`، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **أين الثغرة؟** | صفحة الحساب تعرض كلمة المرور معبأة مسبقاً، ويمكن الوصول إليها لأي مستخدم بتغيير `id` |
| **لماذا هذا خطأ؟** | كلمة المرور لا يجب أن تُرسل إلى العميل (client) أبداً، حتى masked |
| **كيف نستغلها؟** | نغير `id` إلى `administrator` ونقرأ قيمة حقل password |
| **ما هو masked input؟** | حقل من نوع `password` يظهر نقاطاً بدلاً من الأحرف، لكن القيمة موجودة في الـ HTML |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

1. صفحة الحساب تعرض كلمة المرور الحالية للمستخدم في الـ HTML (حتى لو masked)
2. لا يوجد تحقق من الصلاحية عند عرض صفحة مستخدم آخر

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/my-account.js
export default function AccountPage({ user }) {
  return (
    <div>
      <h1>My Account</h1>
      <form>
        <input type="text" name="username" defaultValue={user.username} />
        {/* خطأ: إرسال كلمة المرور الحالية إلى العميل */}
        <input type="password" name="password" defaultValue={user.password} />
        <button type="submit">Update</button>
      </form>
    </div>
  );
}

export async function getServerSideProps(context) {
  const { id } = context.query;
  // خطأ: يسمح بعرض أي مستخدم بدون تحقق
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
      <form>
        <input type="text" name="username" defaultValue={user.username} />
        {/* التصحيح: لا نرسل كلمة المرور الحالية أبداً */}
        <input type="password" name="newPassword" placeholder="New password (leave empty to keep current)" />
        <input type="password" name="confirmPassword" placeholder="Confirm new password" />
        <button type="submit">Update Password</button>
      </form>
    </div>
  );
}

export async function getServerSideProps(context) {
  const { id } = context.query;
  const session = await getSession(context);
  
  // التصحيح: التحقق من الصلاحية
  if (!session || session.user.id !== id) {
    return { redirect: { destination: '/login', permanent: false } };
  }
  
  // لا نرسل كلمة المرور أبداً
  const user = getUserById(id);
  return { props: { user: { username: user.username } } }; // بدون password
}
```

**الخلاصة:** 
1. لا ترسل كلمة المرور الحالية إلى العميل أبداً
2. تحقق من الصلاحية قبل عرض أي بيانات حساسة

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **لا ترسل كلمة المرور إلى العميل:** حتى في input masked.
2. **استخدم نموذج تغيير كلمة المرور منفصل:** يطلب كلمة المرور الجديدة فقط.
3. **تحقق من الصلاحية على كل endpoint:** لا تسمح بعرض بيانات مستخدم آخر.
4. **استخدم session لتحديد المستخدم:** لا تعتمد على معامل `id` في الرابط.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-password-disclosure)
- [Access Control Cheat Sheet](https://portswigger.net/web-security/access-control)
