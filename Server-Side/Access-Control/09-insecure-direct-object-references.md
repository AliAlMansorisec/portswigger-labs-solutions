# Lab: Insecure direct object references

> **Category:** Access Control  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - Insecure direct object references](https://portswigger.net/web-security/access-control/lab-insecure-direct-object-references)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة **IDOR (Insecure Direct Object References)** في خاصية تحميل محادثات الدردشة. الملفات مخزنة على الخادم بأرقام متسلسلة، ويمكننا الوصول إلى ملفات المستخدمين الآخرين بمجرد تغيير رقم الملف.

**المطلوب:** العثور على كلمة مرور المستخدم `carlos` من ملفات المحادثة، ثم تسجيل الدخول بحسابه.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فتح خاصية Live Chat

- في الصفحة الرئيسية للمختبر، اضغط على **Live chat** (أسفل الصفحة)

### الخطوة 2: إرسال رسالة

- اكتب أي رسالة (مثل `hello`) وأرسلها

### الخطوة 3: تحميل نص المحادثة (View transcript)

- بعد إرسال الرسالة، اضغط على **View transcript**

### الخطوة 4: ملاحظة الرابط

ستظهر صفحة جديدة ورابطها مشابه لهذا:

```
https://YOUR-LAB-ID.web-security-academy.net/download-transcript/2.txt
```

**ملاحظة:** الرقم `2` يزداد مع كل محادثة جديدة.

### الخطوة 5: تغيير رقم الملف

- غيّر الرقم في الرابط إلى `1.txt`:

```
https://YOUR-LAB-ID.web-security-academy.net/download-transcript/1.txt
```

### الخطوة 6: قراءة المحادثة

ستظهر محادثة سابقة تحتوي على:

```
... carlos: My password is 3fjsd8f7sdf8s7df ...
```

### الخطوة 7: استخراج كلمة المرور

- انسخ كلمة المرور الخاصة بـ `carlos`

### الخطوة 8: تسجيل الدخول بحساب carlos

- ارجع إلى الصفحة الرئيسية
- اضغط **My account**
- سجل الدخول باستخدام:
  - **Username:** `carlos`
  - **Password:** [كلمة المرور التي حصلت عليها]

### الخطوة 9: حل المختبر

بعد تسجيل الدخول بحساب `carlos`، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي IDOR؟** | الوصول المباشر إلى كائن (ملف، سجل، إلخ) بتغيير معرفه في الرابط |
| **أين الثغرة؟** | الخادم لا يتحقق من أن المستخدم الحالي يملك صلاحية الوصول إلى ملف المحادثة |
| **كيف نستغلها؟** | نغير رقم الملف من `2.txt` إلى `1.txt` (أو أي رقم آخر) |
| **لماذا نجحت؟** | الملفات مخزنة بأرقام متسلسلة يمكن تخمينها |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يستخدم معرفات مباشرة (Direct Object References) بدون التحقق من أن المستخدم الحالي يملك صلاحية الوصول إلى المورد المطلوب.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/download-transcript/[id].js
export default function TranscriptPage({ content }) {
  return <pre>{content}</pre>;
}

export async function getServerSideProps(context) {
  const { id } = context.params;
  const { sessionId } = context.req.cookies;
  
  // خطأ: لا يتحقق من أن المستخدم يملك صلاحية الوصول إلى هذا الملف
  const filePath = path.join(process.cwd(), 'transcripts', `${id}.txt`);
  
  try {
    const content = fs.readFileSync(filePath, 'utf8');
    return { props: { content } };
  } catch (error) {
    return { notFound: true };
  }
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/download-transcript/[id].js
import { getSession } from 'next-auth/react';

export default function TranscriptPage({ content }) {
  return <pre>{content}</pre>;
}

export async function getServerSideProps(context) {
  const { id } = context.params;
  const session = await getSession(context);
  
  // التصحيح 1: التحقق من تسجيل الدخول
  if (!session) {
    return { redirect: { destination: '/login', permanent: false } };
  }
  
  // التصحيح 2: التحقق من أن المستخدم يملك صلاحية الوصول إلى هذا الملف
  const transcriptOwner = getTranscriptOwnerById(id);
  
  if (transcriptOwner !== session.user.id) {
    return { redirect: { destination: '/', permanent: false } };
  }
  
  const filePath = path.join(process.cwd(), 'transcripts', `${id}.txt`);
  
  try {
    const content = fs.readFileSync(filePath, 'utf8');
    return { props: { content } };
  } catch (error) {
    return { notFound: true };
  }
}
```

**الخلاصة:** 
1. لا تستخدم معرفات مباشرة يمكن تخمينها (مثل الأرقام المتسلسلة)
2. تحقق من صلاحية المستخدم قبل عرض أي مورد
3. استخدم معرفات عشوائية غير قابلة للتخمين (UUIDs)

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **التحقق من الصلاحية:** تأكد من أن المستخدم يملك حق الوصول إلى كل مورد.
2. **استخدام معرفات غير قابلة للتخمين:** مثل UUIDs بدلاً من الأرقام المتسلسلة.
3. **لا تعتمد على إخفاء المعرفات:** المعرفات العشوائية وحدها ليست حماية كافية.
4. **استخدام Row-Level Security:** دع قاعدة البيانات تتحقق من الصلاحية.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/access-control/lab-insecure-direct-object-references)
- [IDOR Explained](https://portswigger.net/web-security/access-control#idor)
- [Access Control Cheat Sheet](https://portswigger.net/web-security/access-control)
