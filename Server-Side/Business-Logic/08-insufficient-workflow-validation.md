# Lab: Insufficient workflow validation

> **Category:** Business Logic  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Insufficient workflow validation](https://portswigger.net/web-security/logic-flaws/lab-insufficient-workflow-validation)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة منطق العمل (Business Logic) في **عملية الشراء المتعددة الخطوات**. الخادم يفترض أن الطلبات تأتي بالترتيب الصحيح، لكن يمكننا **تخطي خطوة الدفع** والانتقال مباشرة إلى خطوة تأكيد الطلب، مما يسمح لنا بالحصول على المنتج دون خصم الرصيد.

**الحساب:** `wiener:peter`

**المطلوب:** شراء "Lightweight l33t leather jacket".

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول وفهم عملية الشراء

- سجل الدخول بـ `wiener:peter`
- لاحظ أن رصيدك محدود (مثلاً `$100`)
- سعر السترة الجلدية أغلى من رصيدك (`$1337`)

### الخطوة 2: شراء منتج رخيص (لفهم التسلسل)

- أضف منتجاً رخيصاً (يمكنك شراؤه برصيدك) إلى السلة
- قم بعملية الشراء الكاملة
- في Burp → **Proxy > HTTP history**، ادرس الطلبات

**تسلسل الطلبات الطبيعي:**

```
1. POST /cart → إضافة منتج إلى السلة
2. POST /cart/checkout → بدء عملية الدفع
3. GET /cart/order-confirmation?order-confirmation=true → تأكيد الطلب
4. GET /cart/order-complete → اكتمال الطلب
```

### الخطوة 3: تحديد طلب التأكيد

- ابحث عن طلب `GET /cart/order-confirmation?order-confirmation=true`
- أرسله إلى **Repeater**

### الخطوة 4: إضافة السترة الجلدية إلى السلة

- في المتصفح، أضف "Lightweight l33t leather jacket" إلى السلة
- **لا تقم بالدفع** (لا ترسل طلب `/cart/checkout`)

### الخطوة 5: إعادة إرسال طلب التأكيد

- في Repeater، الطلب الذي حفظته من الخطوة 3:

```http
GET /cart/order-confirmation?order-confirmation=true HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
```

- اضغط **Send**

### الخطوة 6: ملاحظة النتيجة

**النتيجة:** تم تأكيد الطلب واكتمل! ✅

لقد تم شراء السترة الجلدية **بدون خصم الرصيد**!

### الخطوة 7: حل المختبر

بعد إرسال الطلب، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **أين الثغرة؟** | الخادم لا يتحقق من أن طلب التأكيد يأتي بعد خطوة الدفع (`/cart/checkout`) |
| **لماذا هذا خطأ؟** | يجب أن تكون العملية متكاملة: لا يمكن تأكيد طلب لم يتم الدفع له |
| **كيف نستغلها؟** | نتخطى خطوة الدفع ونرسل طلب التأكيد مباشرة |
| **ما هو insufficient workflow validation؟** | التحقق غير الكافي من تسلسل الخطوات في العملية |

---

## 📊 تسلسل العملية الطبيعي مقابل الهجوم

| الخطوة | العملية الطبيعية | الهجوم |
|--------|------------------|--------|
| 1 | إضافة منتج إلى السلة | ✅ إضافة السترة إلى السلة |
| 2 | بدء الدفع (`/cart/checkout`) | ❌ **نتخطى هذه الخطوة** |
| 3 | تأكيد الطلب | ✅ نرسل طلب التأكيد مباشرة |
| 4 | اكتمال الطلب | ✅ يتم الشراء بدون دفع |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم لا يتحقق من أن طلب التأكيد (`/cart/order-confirmation`) يأتي بعد طلب دفع صالح (`/cart/checkout`). يفترض فقط أن الطلبات تأتي بالترتيب الصحيح.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/cart/order-confirmation.js
const orderState = new Map(); // sessionId -> { cartId, confirmed }

export default function handler(req, res) {
  const { sessionId } = req.cookies;
  const { orderConfirmation } = req.query;
  
  if (orderConfirmation !== 'true') {
    return res.status(400).json({ error: 'Invalid confirmation' });
  }
  
  // خطأ: لا يتحقق من وجود عملية دفع سابقة
  const cart = getCart(sessionId);
  
  if (cart.items.length === 0) {
    return res.status(400).json({ error: 'Cart is empty' });
  }
  
  // إنشاء الطلب مباشرة بدون التحقق من الدفع
  createOrder(sessionId, cart);
  clearCart(sessionId);
  
  res.redirect('/cart/order-complete');
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/cart/checkout.js - بدء الدفع
const checkoutState = new Map(); // sessionId -> { cartId, timestamp }

export default function checkoutHandler(req, res) {
  const { sessionId } = req.cookies;
  const cart = getCart(sessionId);
  
  if (cart.items.length === 0) {
    return res.status(400).json({ error: 'Cart is empty' });
  }
  
  // التحقق من كفاية الرصيد
  if (cart.totalPrice > getUserCredit(sessionId)) {
    return res.status(400).json({ error: 'Insufficient credit' });
  }
  
  // تخزين حالة الدفع قبل التأكيد
  const checkoutId = generateCheckoutId();
  checkoutState.set(checkoutId, {
    sessionId,
    cartId: cart.id,
    timestamp: Date.now(),
    totalPrice: cart.totalPrice
  });
  
  res.redirect(`/cart/order-confirmation?checkoutId=${checkoutId}`);
}

// pages/api/cart/order-confirmation.js - تأكيد الطلب
export default function confirmationHandler(req, res) {
  const { checkoutId } = req.query;
  
  // التصحيح: التحقق من وجود عملية دفع صالحة
  const checkout = checkoutState.get(checkoutId);
  
  if (!checkout) {
    return res.status(400).json({ error: 'Invalid or expired checkout' });
  }
  
  // التحقق من انتهاء الوقت (مثلاً 10 دقائق)
  if (Date.now() - checkout.timestamp > 10 * 60 * 1000) {
    checkoutState.delete(checkoutId);
    return res.status(400).json({ error: 'Checkout expired' });
  }
  
  // خصم الرصيد
  const user = getUserBySession(checkout.sessionId);
  deductCredit(user.id, checkout.totalPrice);
  
  // إنشاء الطلب
  createOrder(checkout.sessionId, checkout.cartId);
  clearCart(checkout.sessionId);
  checkoutState.delete(checkoutId);
  
  res.redirect('/cart/order-complete');
}
```

**الخلاصة:** 
1. استخدم **حالة (state)** لربط خطوات العملية ببعضها
2. لا تقبل طلب التأكيد بدون وجود عملية دفع سابقة
3. استخدم معرفات مؤقتة (مثل `checkoutId`) بدلاً من الاعتماد على الترتيب فقط

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **استخدام State Tokens:** أنشئ توكن فريد لكل عملية دفع وتحقق منه في خطوة التأكيد.
2. **التحقق من التسلسل:** تأكد من أن الطلب الحالي يأتي بعد خطوة صالحة سابقة.
3. **صلاحية محدودة:** اجعل حالة الدفع تنتهي بعد فترة زمنية قصيرة.
4. **لا تعتمد على الترتيب فقط:** استخدم آليات تتبع الحالة (state tracking).

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/logic-flaws/lab-insufficient-workflow-validation)
- [Business Logic Vulnerabilities](https://portswigger.net/web-security/logic-flaws)
