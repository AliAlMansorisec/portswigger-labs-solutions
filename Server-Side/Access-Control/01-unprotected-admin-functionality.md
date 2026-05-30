# Lab: Unprotected admin functionality

> **Category:** Access Control  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - Unprotected admin functionality](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة Access Control بالوصول إلى لوحة تحكم المدير (admin panel) غير المحمية، ثم حذف المستخدم `carlos`.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: البحث عن ملف robots.txt

- افتح المختبر في المتصفح
- أضف `/robots.txt` إلى نهاية الرابط:

```
https://YOUR-LAB-ID.web-security-academy.net/robots.txt
```

### الخطوة 2: قراءة محتوى robots.txt

ستظهر صفحة مثل هذه:

```
User-agent: *
Disallow: /administrator-panel
```

**ماذا يعني هذا؟**  
المسار `/administrator-panel` مخفي عن محركات البحث، لكنه **غير محمي** ويمكن لأي شخص زيارته.

### الخطوة 3: زيارة لوحة التحكم

- استبدل `/robots.txt` في الرابط بـ `/administrator-panel`:

```
https://YOUR-LAB-ID.web-security-academy.net/administrator-panel
```

### الخطوة 4: حذف المستخدم carlos

- ستظهر صفحة تعرض المستخدمين
- ابحث عن زر **Delete** بجانب `carlos`
- اضغط عليه

### الخطوة 5: حل المختبر

بعد حذف `carlos`، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو robots.txt؟** | ملف يوجه محركات البحث إلى الصفحات التي لا يجب فهرستها |
| **لماذا هذا خطأ؟** | إخفاء المسار في robots.txt لا يمنع الوصول إليه، بل قد يجذب المهاجمين |
| **كيف نكتشفه؟** | تجربة `/robots.txt` أو `/robots` أو تخمين المسارات الشائعة |
| **الحل الصحيح؟** | حماية اللوحة الإدارية بمصادقة قوية وليس بإخفاء المسار |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

لوحة التحكم الإدارية غير محمية بأي مصادقة (authentication)، والمسار موجود في `robots.txt`.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/administrator-panel.js - بدون أي حماية
export default function AdminPanel() {
  // خطأ: أي شخص يمكنه الوصول إلى هذه الصفحة
  return (
    <div>
      <h1>Admin Panel</h1>
      <button onClick={deleteUser}>Delete carlos</button>
    </div>
  );
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/administrator-panel.js - مع حماية المصادقة
import { getSession } from 'next-auth/react';

export default function AdminPanel({ user }) {
  if (!user || user.role !== 'admin') {
    return <div>Access denied. Admin only.</div>;
  }
  
  return (
    <div>
      <h1>Admin Panel</h1>
      <button onClick={deleteUser}>Delete carlos</button>
    </div>
  );
}

export async function getServerSideProps(context) {
  const session = await getSession(context);
  
  if (!session || session.user.role !== 'admin') {
    return { props: { user: null } };
  }
  
  return { props: { user: session.user } };
}
```

**الخلاصة:** 
1. لا تعتمد على إخفاء المسار كحماية
2. استخدم مصادقة قوية مع التحقق من الصلاحيات (admin role)
3. لا تضع مسارات حساسة في `robots.txt`

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **مصادقة إلزامية:** كل الصفحات الإدارية تتطلب تسجيل دخول.
2. **التحقق من الصلاحيات:** تأكد أن المستخدم لديه role `admin`.
3. **لا تعتمد على robots.txt:** هو إرشاد وليس حماية.
4. **استخدام `getServerSideProps`:** للتحقق من الجلسة قبل عرض الصفحة.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality)
- [Access Control Cheat Sheet](https://portswigger.net/web-security/access-control)
