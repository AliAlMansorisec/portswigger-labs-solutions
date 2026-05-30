# Lab: Unprotected admin functionality with unpredictable URL

> **Category:** Access Control  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - Unprotected admin functionality with unpredictable URL](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality-with-unpredictable-url)

---

## 🎯 الهدف الرئيسي

الوصول إلى لوحة تحكم المدير (admin panel) الموجودة في رابط غير متوقع (unpredictable URL)، ثم حذف المستخدم `carlos`. الرابط مكشوف في مكان ما داخل التطبيق.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فتح الصفحة الرئيسية

- افتح المختبر في المتصفح
- اضغط `Ctrl+U` (أو `Cmd+Option+U` في Mac) لعرض **مصدر الصفحة (Page Source)**

### الخطوة 2: البحث عن رابط الإدارة

- ابحث في مصدر الصفحة عن كلمات مثل:
  - `admin`
  - `panel`
  - `dashboard`
  - `/admin`

### الخطوة 3: العثور على الرابط

ستجد كود مشابه لهذا في المصدر:

```html
<script>
    var isAdmin = false;
    // Admin panel is located at /admin-xxxxx
</script>
```

أو:

```html
<a href="/admin-8f7g3h2j1k">Admin Panel</a>
```

**ملاحظة:** الرابط الحقيقي سيكون مختلفاً وعشوائياً (مثل `/admin-8f7g3h2j1k`).

### الخطوة 4: زيارة رابط الإدارة

- أضف الرابط الذي وجدته إلى نهاية رابط المختبر:

```
https://YOUR-LAB-ID.web-security-academy.net/admin-xxxxx
```

### الخطوة 5: حذف المستخدم carlos

- في صفحة الإدارة، ابحث عن زر **Delete** بجانب `carlos`
- اضغط عليه

### الخطوة 6: حل المختبر

بعد حذف `carlos`، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **لماذا الرابط غير متوقع؟** | لمنع الوصول العشوائي، لكنه مكشوف في مصدر الصفحة |
| **أين نبحث عن الرابط؟** | في مصدر HTML، ملفات JavaScript، أو التعليقات |
| **كيف نفتح مصدر الصفحة؟** | `Ctrl+U` أو من قائمة المتصفح → View Page Source |
| **ماذا لو لم نجده؟** | ابحث في ملفات JS المرفقة أو استخدم Burp لفحص الردود |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

لوحة التحكم الإدارية في رابط عشوائي (security through obscurity) لكن الرابط مكشوف في مصدر الصفحة أو JavaScript.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/admin-8f7g3h2j1k.js - رابط عشوائي لكن غير محمي
export default function AdminPanel() {
  // خطأ: أي شخص يعرف الرابط يمكنه الوصول
  return (
    <div>
      <h1>Admin Panel</h1>
      <button onClick={deleteUser}>Delete carlos</button>
    </div>
  );
}

// components/Header.js - الرابط مكشوف في الكود
<Link href="/admin-8f7g3h2j1k">Admin</Link>
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/admin.js - رابط ثابت مع حماية المصادقة
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

// في الكود: لا تظهر الرابط للمستخدمين العاديين
{user?.role === 'admin' && <Link href="/admin">Admin</Link>}
```

**الخلاصة:** 
1. لا تعتمد على إخفاء الرابط (security through obscurity)
2. استخدم مصادقة قوية مع التحقق من الصلاحيات
3. لا تظهر الروابط الإدارية للمستخدمين غير المصرح لهم

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **مصادقة إلزامية:** كل الصفحات الإدارية تتطلب تسجيل دخول.
2. **التحقق من الصلاحيات:** تأكد أن المستخدم لديه role `admin`.
3. **لا تكشف الروابط الإدارية:** اظهرها فقط للمستخدمين المصرح لهم.
4. **استخدم أدوار (Roles):** ميز بين المستخدم العادي والمدير.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality-with-unpredictable-url)
- [Access Control Cheat Sheet](https://portswigger.net/web-security/access-control)
