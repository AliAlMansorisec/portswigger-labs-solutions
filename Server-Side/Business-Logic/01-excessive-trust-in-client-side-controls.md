# Lab: Excessive trust in client-side controls

> **Category:** Business Logic  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - Excessive trust in client-side controls](https://portswigger.net/web-security/logic-flaws/lab-excessive-trust-in-client-side-controls)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة منطق العمل (Business Logic) حيث يثق الخادم بشكل مفرط في المدخلات القادمة من العميل. سنقوم بتعديل سعر المنتج أثناء إضافته إلى السلة لشراء سترة جلدية بسعر أقل من رصيدنا.

**الحساب:** `wiener:peter`

**المطلوب:** شراء "Lightweight l33t leather jacket".

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول وفهم العملية

- سجل الدخول بـ `wiener:peter`
- لاحظ أن رصيدك (store credit) محدود (مثلاً 100 دولار)
- سعر السترة الجلدية أغلى من رصيدك

### الخطوة 2: محاولة الشراء العادية (ستفشل)

- أضف السترة إلى السلة
- حاول إتمام الشراء
- **النتيجة:** يتم رفض الطلب بسبب عدم كفاية الرصيد

### الخطوة 3: اعتراض طلب الإضافة إلى السلة

- في Burp → **Proxy > HTTP history**
- ابحث عن طلب `POST /cart` عند إضافة المنتج إلى السلة:

```http
POST /cart HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded

productId=1&redir=PRODUCT&quantity=1&price=133700
```

**ملاحظة:** طلب الإضافة يحتوي على معامل `price` بقيمة سعر المنتج!

### الخطوة 4: إرسال الطلب إلى Repeater

- زر الماوس الأيمن → **Send to Repeater**

### الخطوة 5: تعديل معامل price

- غيّر قيمة `price` إلى رقم أقل من رصيدك (مثلاً `1`):

```http
POST /cart HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded

productId=1&redir=PRODUCT&quantity=1&price=1
```

### الخطوة 6: إرسال الطلب

- اضغط **Send**

### الخطوة 7: تحديث صفحة السلة

- في المتصفح، اذهب إلى **Cart** (أو قم بتحديث الصفحة)
- **النتيجة:** سعر السترة أصبح `1` دولار! ✅

### الخطوة 8: إتمام الشراء

- اضغط على **Place order** لإتمام عملية الشراء

### الخطوة 9: حل المختبر

بعد شراء السترة، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **أين الثغرة؟** | الخادم يثق في معامل `price` المرسل من العميل عند إضافة منتج إلى السلة |
| **لماذا هذا خطأ؟** | السعر يجب أن يُحدد من قِبل الخادم، وليس من قِبل المستخدم |
| **كيف نستغلها؟** | نغير قيمة `price` إلى أي رقم نريده (حتى `1`) |
| **ما هي ثغرة Business Logic؟** | ثغرة في منطق العمل تسمح بتجاوز القواعد (مثل شراء منتج بسعر أقل) |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يثق في معامل `price` المرسل من العميل لتحديد سعر المنتج، بدلاً من استخدام السعر المخزن في قاعدة البيانات.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/cart.js - إضافة منتج إلى السلة
export default function handler(req, res) {
  const { sessionId } = req.cookies;
  const { productId, quantity, price } = req.body;
  
  // خطأ: يثق في السعر المرسل من العميل
  const product = getProductById(productId);
  
  addToCart(sessionId, {
    productId,
    quantity,
    price: price // السعر من المستخدم!
  });
  
  res.redirect('/cart');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/cart.js - إضافة منتج إلى السلة
export default function handler(req, res) {
  const { sessionId } = req.cookies;
  const { productId, quantity } = req.body;
  
  // التصحيح: استخدام السعر من قاعدة البيانات
  const product = getProductById(productId);
  
  if (!product) {
    return res.status(404).json({ error: 'Product not found' });
  }
  
  addToCart(sessionId, {
    productId,
    quantity: parseInt(quantity),
    price: product.price // السعر من قاعدة البيانات
  });
  
  res.redirect('/cart');
}
```

**الخلاصة:** 
1. لا تثق أبداً في السعر أو أي بيانات حساسة قادمة من العميل
2. استخدم البيانات المخزنة في قاعدة البيانات (backend)
3. تحقق من صحة جميع المدخلات قبل معالجتها

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **لا ترسل الأسعار إلى العميل:** أو على الأقل لا تستخدمها عند العودة.
2. **استخدم السعر من قاعدة البيانات:** دائماً عند حساب المجموع الإجمالي.
3. **تحقق من صحة المدخلات:** تأكد أن الكمية رقمية وليست سالبة.
4. **أعد حساب السعر في الخادم:** لا تعتمد على أي حسابات من العميل.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/logic-flaws/lab-excessive-trust-in-client-side-controls)
- [Business Logic Vulnerabilities](https://portswigger.net/web-security/logic-flaws)
