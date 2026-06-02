# Lab: Infinite money logic flaw

> **Category:** Business Logic  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Infinite money logic flaw](https://portswigger.net/web-security/logic-flaws/lab-infinite-money-logic-flaw)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة منطق العمل (Business Logic) في نظام شراء واسترداد بطاقات الهدايا (gift cards). يمكننا شراء بطاقة هدية بسعر مخفض (بعد خصم 30%) واستردادها بقيمتها الكاملة ($10)، مما يولد ربحاً قدره $3 في كل دورة. بتكرار العملية، نحصل على رصيد غير محدود لشراء السترة الجلدية.

**الحساب:** `wiener:peter`

**المطلوب:** شراء "Lightweight l33t leather jacket".

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول والحصول على الكوبون

- سجل الدخول بـ `wiener:peter`
- اشترك في النشرة البريدية (Sign up for newsletter)
- ستحصل على كوبون خصم: `SIGNUP30` (خصم 30%)

### الخطوة 2: فهم الثغرة

**المنطق الطبيعي:**
1. شراء بطاقة هدية بـ $10
2. تطبيق كوبون 30% → تدفع $7
3. استرداد البطاقة → تحصل على $10
4. **الربح: $3 لكل دورة** ✅

يمكن تكرار هذه العملية للحصول على رصيد غير محدود.

### الخطوة 3: إعداد Session Handling Rule في Burp

#### 3.1 فتح الإعدادات
- **Burp** → **Settings** (أو **Project options** → **Sessions**)
- في **Session handling rules** → **Add**

#### 3.2 إعداد نطاق القاعدة (Scope)
- **Scope** tab
- **URL scope** → **Include all URLs**

#### 3.3 إضافة Macro
- **Details** tab → **Rule actions** → **Add** → **Run a macro**
- **Select macro** → **Add** (لإنشاء ماكرو جديد)

#### 3.4 اختيار الطلبات في Macro Recorder

اختر هذه الطلبات بالترتيب (من Proxy history):

```
1. POST /cart (إضافة بطاقة هدية إلى السلة)
2. POST /cart/coupon (تطبيق كوبون SIGNUP30)
3. POST /cart/checkout (بدء الدفع)
4. GET /cart/order-confirmation?order-confirmation=true (تأكيد الطلب)
5. POST /gift-card (استرداد بطاقة الهدية)
```

#### 3.5 تكوين استخراج رمز البطاقة

- في Macro Editor، اختر الطلب رقم 4 (`GET /cart/order-confirmation...`)
- **Configure item** → **Add** (custom parameter)
  - **Parameter name:** `gift-card`
  - حدد رمز البطاقة من الـ Response (أسفل الصفحة)
- **OK**

#### 3.6 ربط المعامل بالطلب الأخير

- اختر الطلب رقم 5 (`POST /gift-card`)
- **Configure item** → **Parameter handling**
  - معامل `gift-card` → Derived from prior response (response 4)
- **OK**

#### 3.7 اختبار الماكرو

- **Test macro**
- تأكد أن رمز البطاقة ظهر في الـ Response رقم 4
- وتأكد أن `POST /gift-card` يستخدم نفس الرمز
- **OK** حتى العودة إلى الإعدادات الرئيسية

### الخطوة 4: إعداد Burp Intruder

#### 4.1 إرسال طلب إلى Intruder
- اذهب إلى `GET /my-account` (أو أي طلب)
- زر الماوس الأيمن → **Send to Intruder**

#### 4.2 إعدادات Intruder
- **Attack type:** Sniper
- **Payloads** tab:
  - **Payload type:** Null payloads
  - **Generate:** 412 payloads (أو عدد كافٍ للحصول على رصيد $1337)
- **Resource pool** tab:
  - **Maximum concurrent requests:** 1 (هام جداً!)

### الخطوة 5: بدء الهجوم

- **Start attack**
- سيقوم Intruder بتنفيذ الماكرو 412 مرة
- كل دورة تضيف $3 إلى رصيدك

### الخطوة 6: حساب عدد الدورات المطلوبة

سعر السترة: $1337
الرصيد الحالي: $100

المبلغ المطلوب: `1337 - 100 = 1237` دولار

كل دورة تربح: $3

عدد الدورات المطلوبة: `1237 / 3 ≈ 413` دورة

### الخطوة 7: شراء السترة

- بعد اكتمال الهجوم، ارجع إلى المتصفح
- أضف "Lightweight l33t leather jacket" إلى السلة
- أكمل عملية الشراء (الرصيد الآن كافٍ)

### الخطوة 8: حل المختبر

بعد شراء السترة، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **أين الثغرة؟** | يمكن استرداد بطاقة الهدية بقيمتها الكاملة ($10) بعد شرائها بسعر مخفض ($7) |
| **لماذا هذا خطأ؟** | يجب أن يكون سعر الشراء والاسترداد متطابقين، أو منع استرداد البطاقات المشتراة بخصم |
| **كيف نستغلها؟** | بأتمتة عملية الشراء والاسترداد باستخدام Burp Intruder |
| **لماذا 412 دورة؟** | $1337 سعر السترة، $100 الرصيد الحالي، $4 ربح كل دورة ≈ 413 دورة |

---

## 📊 شرح الربح في كل دورة

```
شراء بطاقة هدية: -$10
تطبيق خصم 30%: +$3 (تدفع فقط $7)
استرداد البطاقة: +$10
─────────────────
الربح الصافي: +$3
```

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

نظام استرداد بطاقات الهدايا لا يتحقق من أن البطاقة تم شراؤها بالسعر الكامل، مما يسمح بتحقيق ربح من البطاقات المشتراة بخصم.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// صفحات API - شراء بطاقة هدية
export default function purchaseGiftCard(req, res) {
  const { sessionId } = req.cookies;
  const { couponCode } = req.body;
  
  let price = 1000; // $10 بالcents
  
  if (couponCode === 'SIGNUP30') {
    price = price * 0.7; // $7 بعد الخصم
  }
  
  // خصم المبلغ من رصيد المستخدم
  deductCredit(sessionId, price);
  
  // إنشاء رمز بطاقة عشوائي
  const giftCardCode = generateGiftCardCode();
  
  // تخزين البطاقة في قاعدة البيانات
  storeGiftCard(sessionId, giftCardCode, 1000); // القيمة الاسمية $10
  
  res.json({ giftCardCode });
}

// استرداد بطاقة هدية
export default function redeemGiftCard(req, res) {
  const { sessionId } = req.cookies;
  const { giftCardCode } = req.body;
  
  // خطأ: لا يتحقق من سعر شراء البطاقة
  const giftCard = getGiftCard(giftCardCode);
  
  if (!giftCard || giftCard.redeemed) {
    return res.status(400).json({ error: 'Invalid gift card' });
  }
  
  // إضافة القيمة الاسمية الكاملة ($10) إلى الرصيد
  addCredit(sessionId, giftCard.value); // $10
  
  giftCard.redeemed = true;
  res.json({ success: true });
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// صفحات API - شراء بطاقة هدية
export default function purchaseGiftCard(req, res) {
  const { sessionId } = req.cookies;
  const { couponCode } = req.body;
  
  let price = 1000; // $10 بالcents
  let paidAmount = price;
  
  if (couponCode === 'SIGNUP30') {
    paidAmount = price * 0.7; // $7 بعد الخصم
  }
  
  // خصم المبلغ من رصيد المستخدم
  deductCredit(sessionId, paidAmount);
  
  // إنشاء رمز بطاقة عشوائي
  const giftCardCode = generateGiftCardCode();
  
  // التصحيح: تخزين سعر الشراء الفعلي
  storeGiftCard(sessionId, giftCardCode, {
    nominalValue: price,     // $10
    purchasePrice: paidAmount  // $7
  });
  
  res.json({ giftCardCode });
}

// استرداد بطاقة هدية
export default function redeemGiftCard(req, res) {
  const { sessionId } = req.cookies;
  const { giftCardCode } = req.body;
  
  const giftCard = getGiftCard(giftCardCode);
  
  if (!giftCard || giftCard.redeemed) {
    return res.status(400).json({ error: 'Invalid gift card' });
  }
  
  // التصحيح: التحقق من أن المستخدم هو من اشترى البطاقة
  if (giftCard.purchasedBy !== sessionId) {
    return res.status(403).json({ error: 'Not your gift card' });
  }
  
  // التصحيح: استرداد سعر الشراء فقط، وليس القيمة الاسمية
  const refundAmount = giftCard.purchasePrice; // $7 فقط، وليس $10
  
  addCredit(sessionId, refundAmount);
  
  giftCard.redeemed = true;
  res.json({ success: true });
}
```

**الخلاصة:** 
1. سجل سعر الشراء الفعلي (وليس القيمة الاسمية) عند شراء بطاقة هدية
2. عند الاسترداد، أعد فقط المبلغ الذي دفعه المستخدم
3. تأكد من أن المستخدم المسترد هو من اشترى البطاقة

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **تتبع سعر الشراء الفعلي:** لا تستخدم القيمة الاسمية للاسترداد.
2. **منع استرداد البطاقات المشتراة بخصم:** أو رد فقط المبلغ المدفوع.
3. **ربط البطاقة بالمستخدم:** تأكد من أن المستخدم المسترد هو من اشتراها.
4. **حد أقصى لعدد البطاقات:** لا تسمح بشراء عدد غير محدود من البطاقات.
5. **مراجعة العملية يدوياً:** تطبيق الحد الأقصى للخصم أو الربح المسموح.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/logic-flaws/lab-infinite-money-logic-flaw)
- [Business Logic Vulnerabilities](https://portswigger.net/web-security/logic-flaws)
