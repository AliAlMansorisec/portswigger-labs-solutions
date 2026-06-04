 Lab: Information disclosure in error messages

> **Category:** Information Disclosure  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - Information disclosure in error messages](https://portswigger.net/web-security/information-disclosure/lab-information-disclosure-in-error-messages)

---

## 🎯 الهدف الرئيسي

استغلال رسائل الخطأ المفصلة (verbose error messages) التي يكشفها الخادم عند إرسال بيانات غير متوقعة. هذه الرسائل تكشف معلومات حساسة، مثل **رقم إصدار الإطار المستخدم (framework version)**.

**المطلوب:** الحصول على رقم إصدار Apache Struts وإرساله.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: فتح صفحة منتج

- افتح المختبر في المتصفح
- اضغط على أي منتج لعرض صفحته

### الخطوة 2: اعتراض الطلب في Burp

- في Burp → **Proxy > HTTP history**
- ابحث عن طلب `GET /product`:

```http
GET /product?productId=1 HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

### الخطوة 3: إرسال الطلب إلى Repeater

- زر الماوس الأيمن → **Send to Repeater**

### الخطوة 4: تغيير productId إلى قيمة غير متوقعة

- غيّر `productId=1` إلى قيمة نصية (string) بدلاً من رقم:

```http
GET /product?productId="example" HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

### الخطوة 5: إرسال الطلب

- اضغط **Send**

### الخطوة 6: ملاحظة رسالة الخطأ

ستظهر استجابة تحتوي على **stack trace** كامل:

```java
java.lang.NumberFormatException: For input string: "example"
    at java.lang.NumberFormatException.forInputString(NumberFormatException.java:65)
    ...
    at org.apache.struts2.dispatcher.Dispatcher.serviceAction(Dispatcher.java:532)
    ...
    at org.apache.struts2 2.3.31...
```

**المعلومة المطلوبة:** `2 2.3.31` (أو `2.3.31`)

### الخطوة 7: إرسال الإجابة

- ارجع إلى صفحة المختبر
- اضغط **Submit solution**
- أدخل رقم الإصدار: `2 2.3.31`

### الخطوة 8: حل المختبر

بعد إرسال الإجابة الصحيحة، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هي المعلومات المسربة؟** | رقم إصدار Apache Struts (مثال: 2.3.31) |
| **كيف حدث التسريب؟** | إرسال نوع بيانات غير متوقع (string بدلاً من int) تسبب في استثناء (exception) وكشف stack trace |
| **لماذا هذا خطير؟** | الإصدار القديم قد يحتوي على ثغرات معروفة يمكن استغلالها |
| **ما هي verbose error messages؟** | رسائل خطأ مفصلة تعرض معلومات داخلية عن النظام |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يعرض رسائل خطأ مفصلة (stack traces) تحتوي على معلومات حساسة عن الإطار المستخدم وإصداره.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/product.js
export default function productHandler(req, res) {
  const { productId } = req.query;
  
  try {
    const id = parseInt(productId);
    const product = getProductById(id);
    res.json(product);
  } catch (error) {
    // خطأ: إرسال stack trace كاملاً إلى العميل
    res.status(500).send(error.stack);
  }
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/product.js
export default function productHandler(req, res) {
  const { productId } = req.query;
  
  try {
    const id = parseInt(productId);
    
    if (isNaN(id)) {
      // التصحيح: رسالة خطأ عامة بدون تفاصيل
      return res.status(400).json({ error: 'Invalid product ID' });
    }
    
    const product = getProductById(id);
    
    if (!product) {
      return res.status(404).json({ error: 'Product not found' });
    }
    
    res.json(product);
  } catch (error) {
    // التصحيح: تسجيل الخطأ في الخادم فقط
    console.error('Internal error:', error);
    
    // إرسال رسالة عامة للعميل
    res.status(500).json({ error: 'Internal server error' });
  }
}
```

**الخلاصة:** 
1. لا ترسل stack traces أو معلومات داخلية إلى العميل
2. استخدم رسائل خطأ عامة للمستخدمين
3. سجل التفاصيل في الخادم فقط (logs)

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **إخفاء رسائل الخطأ التفصيلية:** في بيئة الإنتاج (production)، لا تعرض stack traces.
2. **استخدام middleware عالمي:** للقبض على الأخطاء وإرسال رسائل عامة.
3. **تسجيل الأخطاء في الخادم:** استخدم logging لحفظ التفاصيل للمطورين.
4. **التحقق من صحة المدخلات (Input validation):** ارفض البيانات غير المتوقعة برسائل عامة.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/information-disclosure/lab-information-disclosure-in-error-messages)
- [Information Disclosure Cheat Sheet](https://portswigger.net/web-security/information-disclosure)
