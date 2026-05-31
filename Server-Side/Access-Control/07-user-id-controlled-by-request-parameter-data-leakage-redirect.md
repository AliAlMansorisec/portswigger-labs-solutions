# Lab: User ID controlled by request parameter with data leakage in redirect

> **Category:** Access Control  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - User ID controlled by request parameter with data leakage in redirect](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-data-leakage-in-redirect)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة **Horizontal Privilege Escalation** في صفحة حساب المستخدم. الخادم يعيد التوجيه (redirect) عند محاولة الوصول إلى مستخدم آخر، لكن **البيانات الحساسة تتسرب في جسم (body) رد التوجيه** قبل الانتقال.

**الحساب:** `wiener:peter`

**المطلوب:** الحصول على API key للمستخدم `carlos` وإرسالها.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول

- سجل الدخول بـ `wiener:peter`
- اذهب إلى **My account**

### الخطوة 2: اعتراض الطلب في Burp

- في Burp → **Proxy > HTTP history**
- اعثر على طلب صفحة الحساب:

```http
GET /my-account?id=wiener HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
```

### الخطوة 3: إرسال الطلب إلى Repeater

- بزر الماوس الأيمن → **Send to Repeater**

### الخطوة 4: تغيير الـ id إلى carlos

- غيّر `id=wiener` إلى `id=carlos`:

```http
GET /my-account?id=carlos HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
```

### الخطوة 5: إرسال الطلب وملاحظة الرد

- اضغط **Send**
- انظر إلى الرد في Repeater

ستلاحظ:

- **Status Code:** `302 Found` (إعادة توجيه)
- **Location header:** `https://YOUR-LAB-ID.web-security-academy.net/` (يعيد إلى الصفحة الرئيسية)
- **Body:** يحتوي على **API key الخاص بـ carlos**!

مثال للرد:

```http
HTTP/1.1 302 Found
Location: https://YOUR-LAB-ID.web-security-academy.net/
Content-Type: text/html; charset=utf-8

<html>
  <body>
    <p>Redirecting...</p>
    <p>API Key for carlos: 2fjsd8f7sdf8s7df</p>
  </body>
</html>
```

### الخطوة 6: استخراج API key

- انسخ الـ API key من الـ Body

### الخطوة 7: إرسال API key

- أرسل الـ API key في حقل الإجابة

### الخطوة 8: حل المختبر

بعد إرسال الـ API key الصحيح، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي الثغرة؟** | الخادم يحاول منع الوصول غير المصرح به (بـ redirect)، لكنه يعرض البيانات الحساسة في الـ Body قبل التوجيه |
| **كيف نستغلها؟** | نغير `id` إلى `carlos` ونقرأ الـ Body في الـ Response |
| **لماذا يظهر API key؟** | الخادم يقوم بجلب بيانات `carlos` ثم يكتشف عدم الصلاحية فيعيد التوجيه، لكن البيانات تكون قد جُلبت بالفعل |
| **الفرق عن اللاب السابق؟** | السابق كان يسمح بالوصول المباشر، وهذا يمنع الوصول لكن يتسرب البيانات في الـ redirect |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يقوم بجلب بيانات المستخدم المطلوب **قبل** التحقق من الصلاحية، ثم إذا كان غير مصرح له يعيد التوجيه لكن البيانات تكون موجودة بالفعل في الـ Response.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/my-account.js
export async function getServerSideProps(context) {
  const { id } = context.query;
  const { sessionId } = context.req.cookies;
  
  // خطأ: جلب البيانات قبل التحقق من الصلاحية
  const requestedUser = getUserById(id);
  
  const currentUser = getUserBySession(sessionId);
  
  // التحقق من الصلاحية بعد جلب البيانات!
  if (!currentUser || currentUser.id !== id) {
    // ولكن البيانات موجودة بالفعل في memory
    return {
      redirect: {
        destination: '/',
        permanent: false,
      },
      // خطأ: البيانات تتسرب في props قبل الـ redirect
      props: { leakedApiKey: requestedUser.apiKey }
    };
  }
  
  return { props: { user: requestedUser } };
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/my-account.js
import { getSession } from 'next-auth/react';

export async function getServerSideProps(context) {
  const { id } = context.query;
  const session = await getSession(context);
  
  // التصحيح: التحقق من الصلاحية أولاً
  if (!session || session.user.id !== id) {
    // إعادة توجيه بدون جلب أي بيانات
    return {
      redirect: {
        destination: '/',
        permanent: false,
      },
      props: {} // لا بيانات حساسة
    };
  }
  
  // فقط بعد التحقق من الصلاحية نقوم بجلب البيانات
  const user = getUserById(id);
  return { props: { user } };
}
```

**الخلاصة:** 
1. تحقق من الصلاحية **قبل** جلب أي بيانات حساسة
2. لا تجلب بيانات المستخدم المستهدف إلا بعد التأكد من أن المستخدم الحالي مصرح له

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **التحقق من الصلاحية أولاً:** لا تجلب البيانات قبل التحقق.
2. **لا تخلط البيانات مع الـ redirect:** إذا كان هناك redirect، لا تضف أي بيانات حساسة في الـ Body أو الـ props.
3. **استخدم `getServerSideProps` بحذر:** تأكد من الترتيب الصحيح للعمليات.
4. **استخدام Row-Level Security:** دع قاعدة البيانات تتحقق من الصلاحية عند جلب البيانات.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-data-leakage-in-redirect)
- [Access Control Cheat Sheet](https://portswigger.net/web-security/access-control)

