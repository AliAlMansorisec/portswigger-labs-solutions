# Lab: User ID controlled by request parameter, with unpredictable user IDs

> **Category:** Access Control  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - User ID controlled by request parameter, with unpredictable user IDs](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-unpredictable-user-ids)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة **Horizontal Privilege Escalation** في صفحة حساب المستخدم، ولكن هذه المرة المعرفات (IDs) غير قابلة للتخمين (GUIDs). سنبحث عن معرف `carlos` في مكان آخر في التطبيق، ثم نستخدمه للوصول إلى بياناته.

**الحساب:** `wiener:peter`

**المطلوب:** الحصول على API key للمستخدم `carlos` وإرسالها.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: البحث عن مقال لـ carlos

- تصفح الموقع وابحث عن مقال (blog post) كتبه `carlos`
- ستجد مقالات عادية في الصفحة الرئيسية

### الخطوة 2: النقر على اسم carlos

- اضغط على اسم `carlos` في المقال
- لاحظ الرابط:

```
https://YOUR-LAB-ID.web-security-academy.net/blogs?userId=2a3b4c5d-6e7f-8g9h-0i1j-2k3l4m5n6o7p
```

**هذا هو GUID الخاص بـ carlos!** ✅

### الخطوة 3: تسجيل الدخول

- سجل الدخول بـ `wiener:peter`
- اذهب إلى **My account**

### الخطوة 4: ملاحظة الرابط الخاص بحساب wiener

رابط صفحة حساب wiener:

```
https://YOUR-LAB-ID.web-security-academy.net/my-account?id=7x8y9z0a-1b2c-3d4e-5f6g-7h8i9j0k1l2m
```

### الخطوة 5: استبدال الـ id بـ GUID الخاص بـ carlos

- غيّر قيمة `id` في الرابط من GUID الـ wiener إلى GUID الـ carlos:

```
https://YOUR-LAB-ID.web-security-academy.net/my-account?id=2a3b4c5d-6e7f-8g9h-0i1j-2k3l4m5n6o7p
```

### الخطوة 6: عرض بيانات carlos

ستظهر صفحة حساب المستخدم `carlos` تحتوي على:

```
API Key: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### الخطوة 7: إرسال API key

- انسخ الـ API key
- أرسلها في حقل الإجابة

### الخطوة 8: حل المختبر

بعد إرسال الـ API key الصحيح، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو GUID؟** | معرف فريد عالمياً غير قابل للتخمين (مثل `2a3b4c5d-6e7f-8g9h-0i1j-2k3l4m5n6o7p`) |
| **أين وجدنا GUID كارلوس؟** | في رابط صفحة مقالاته (`/blogs?userId=...`) |
| **ما هي الثغرة؟** | الخادم لا يتحقق من أن المستخدم الحالي يملك صلاحية لعرض بيانات مستخدم آخر |
| **كيف نستغلها؟** | نجد GUID المستخدم المستهدف من مكان آخر ونستخدمه في رابط `/my-account` |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يستخدم GUIDs غير قابلة للتخمين، لكنه يفشل في التحقق من أن المستخدم الحالي هو صاحب البيانات المطلوبة (Missing access control on the endpoint).

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
  
  // خطأ: يثق بـ id ويعرض أي مستخدم
  const user = getUserById(id);
  
  if (!user) {
    return { notFound: true };
  }
  
  return { props: { user } };
}

// pages/blogs.js - يعرض GUID في الرابط
export async function getServerSideProps(context) {
  const { userId } = context.query;
  const blogs = getBlogsByUserId(userId);
  return { props: { blogs, userId } };
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
  
  // التصحيح: التحقق من أن المستخدم الحالي يملك صلاحية الوصول
  if (!session) {
    return { redirect: { destination: '/login', permanent: false } };
  }
  
  // المستخدم يمكنه فقط عرض بيانات نفسه
  if (session.user.id !== id) {
    return { redirect: { destination: '/my-account', permanent: false } };
  }
  
  const user = getUserById(id);
  return { props: { user } };
}

// pages/blogs.js - لا تعرض GUID في الرابط (أو امنع الوصول غير المصرح به)
export async function getServerSideProps(context) {
  const { userId } = context.query;
  const session = await getSession(context);
  
  // التحقق: المستخدم يمكنه فقط رؤية مقالاته
  if (session?.user?.id !== userId) {
    return { props: { blogs: [], userId: null } };
  }
  
  const blogs = getBlogsByUserId(userId);
  return { props: { blogs, userId } };
}
```

**الخلاصة:** 
1. استخدام GUIDs غير قابل للتخمين **ليس حماية كافية**
2. يجب إضافة **Access Control** يتحقق من أن المستخدم الحالي يملك صلاحية الوصول
3. كل endpoint يعرض بيانات حساسة يجب أن يتحقق من الصلاحية

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **التحقق من الصلاحية على كل endpoint:** لا تعتمد على صعوبة تخمين المعرفات.
2. **استخدام الجلسة لتحديد المستخدم:** لا تعتمد على معامل `id` من الرابط.
3. **إخفاء GUIDs الحساسة:** لا تعرض GUIDs المستخدمين في روابط عامة (مثل `/blogs?userId=...`).
4. **استخدام Row-Level Security:** تأكد أن المستخدم يمكنه فقط قراءة بياناته.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-unpredictable-user-ids)
- [Access Control Cheat Sheet](https://portswigger.net/web-security/access-control)
- [GUID vs ID](https://portswigger.net/web-security/access-control#platform-configuration-errors)
