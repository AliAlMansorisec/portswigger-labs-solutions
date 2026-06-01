# Lab: Flawed enforcement of business rules

> **Category:** Business Logic  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - Flawed enforcement of business rules](https://portswigger.net/web-security/logic-flaws/lab-flawed-enforcement-of-business-rules)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة منطق العمل (Business Logic) في نظام الخصم (coupon codes). الخادم يتحقق من أن نفس الكوبون لا يُطبق مرتين متتاليتين، ولكن يمكن **التناوب بين كوبونين** لتطبيقهما عدة مرات والحصول على خصم غير محدود.

**الحساب:** `wiener:peter`

**المطلوب:** شراء "Lightweight l33t leather jacket" باستخدام الخصومات المتكررة حتى يصبح السعر أقل من رصيدك.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول والحصول على الكوبونات

- سجل الدخول بـ `wiener:peter`
- لاحظ وجود كوبون: `NEWCUST5` (خصم 5% للمستخدمين الجدد)

### الخطوة 2: الاشتراك في النشرة البريدية

- اذهب إلى أسفل الصفحة
- اشترك في النشرة البريدية (Sign up to newsletter)
- ستتلقى كوبون آخر: `SIGNUP30` (خصم 30%)

### الخطوة 3: إضافة السترة الجلدية إلى السلة

- أضف "Lightweight l33t leather jacket" إلى السلة
- سعرها مثلاً `$1337`

### الخطوة 4: الذهاب إلى صفحة الدفع

- اذهب إلى **Checkout** (أو **Cart** ثم **Checkout**)

### الخطوة 5: تطبيق الكوبونات مرة واحدة

- جرب تطبيق `NEWCUST5` → ينجح ✅ (الخصم 5%)
- جرب تطبيق `SIGNUP30` → ينجح ✅ (الخصم 30% على السعر المخفض)

### الخطوة 6: اكتشاف الثغرة

- جرب تطبيق `NEWCUST5` مرة أخرى → يتم رفضه ❌ (Coupon already applied)
- جرب تطبيق `SIGNUP30` مرة أخرى → يتم رفضه ❌

**لكن:** إذا **تناوبت** بين الكوبونين:

```
تطبيق NEWCUST5 → ينجح ✅
تطبيق SIGNUP30 → ينجح ✅
تطبيق NEWCUST5 → ينجح ✅ (لم يعد مرفوضاً!)
تطبيق SIGNUP30 → ينجح ✅
تطبيق NEWCUST5 → ينجح ✅
... وهكذا
```

الخادم يتذكر آخر كوبون تم تطبيقه فقط، وليس جميع الكوبونات المستخدمة!

### الخطوة 7: تكرار الكوبونات لتقليل السعر

- استمر في التناوب بين `NEWCUST5` و `SIGNUP30`
- كل مرة ينخفض السعر الإجمالي

مثال (تقريبي):
```
السعر الأصلي: $1337
بعد NEWCUST5 (5%): $1270.15
بعد SIGNUP30 (30%): $889.10
بعد NEWCUST5 (5%): $844.64
بعد SIGNUP30 (30%): $591.25
بعد NEWCUST5 (5%): $561.69
بعد SIGNUP30 (30%): $393.18
... وهكذا
```

### الخطوة 8: الوصول إلى سعر أقل من رصيدك

- استمر في التناوب حتى يصبح السعر أقل من رصيدك (مثلاً $100 أو أقل)

### الخطوة 9: إتمام الشراء

- اضغط على **Place order** (أو تأكيد الشراء)

### الخطوة 10: حل المختبر

بعد شراء السترة، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **أين الثغرة؟** | الخادم يتذكر فقط **آخر كوبون** تم تطبيقه، وليس سجل جميع الكوبونات المستخدمة |
| **لماذا هذا خطأ؟** | يجب أن يمنع الخادم استخدام أي كوبون أكثر من مرة، بغض النظر عن الترتيب |
| **كيف نستغلها؟** | بالتناوب بين كوبونين لتطبيقهما عدة مرات |
| **ما هو flawed enforcement؟** | تطبيق غير صحيح لقاعدة العمل (يجب أن يكون الكوبون مرة واحدة فقط) |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

الخادم يتتبع فقط **آخر كوبون** تم تطبيقه لمنع التكرار، وليس سجل كامل لجميع الكوبونات المستخدمة.

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/apply-coupon.js
const appliedCoupons = new Map(); // sessionId -> lastCouponCode

export default function handler(req, res) {
  const { sessionId } = req.cookies;
  const { couponCode } = req.body;
  
  // خطأ: يتحقق فقط من أن الكوبون ليس هو آخر كوبون تم تطبيقه
  const lastCoupon = appliedCoupons.get(sessionId);
  
  if (lastCoupon === couponCode) {
    return res.status(400).json({ error: 'Coupon already applied' });
  }
  
  // تطبيق الخصم
  const cart = getCart(sessionId);
  const discount = getCouponDiscount(couponCode);
  cart.applyDiscount(discount);
  
  // تخزين آخر كوبون فقط
  appliedCoupons.set(sessionId, couponCode);
  
  res.json({ success: true, newTotal: cart.total });
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/apply-coupon.js
const appliedCoupons = new Map(); // sessionId -> Set of coupon codes

export default function handler(req, res) {
  const { sessionId } = req.cookies;
  const { couponCode } = req.body;
  
  // التصحيح: تتبع جميع الكوبونات المستخدمة
  const usedCoupons = appliedCoupons.get(sessionId) || new Set();
  
  if (usedCoupons.has(couponCode)) {
    return res.status(400).json({ error: 'Coupon already used' });
  }
  
  // التحقق من أن الكوبون صالح (مرة واحدة فقط لكل مستخدم)
  if (isSingleUseCoupon(couponCode)) {
    usedCoupons.add(couponCode);
    appliedCoupons.set(sessionId, usedCoupons);
  }
  
  // تطبيق الخصم
  const cart = getCart(sessionId);
  const discount = getCouponDiscount(couponCode);
  cart.applyDiscount(discount);
  
  res.json({ success: true, newTotal: cart.total });
}
```

**الخلاصة:** 
1. تتبع جميع الكوبونات المستخدمة، وليس آخر واحد فقط
2. كل كوبون يجب أن يُطبق مرة واحدة فقط لكل مستخدم
3. استخدم Set أو قائمة لتخزين الكوبونات المستخدمة

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **تتبع جميع الكوبونات المستخدمة:** استخدم قائمة (Set) وليس متغيراً واحداً.
2. **التحقق من الصلاحية على الخادم:** لا تعتمد على التحقق من جانب العميل.
3. **تحديد صلاحية الكوبون:** مرة واحدة لكل مستخدم، أو لفترة زمنية محددة.
4. **التحقق من الحد الأقصى للخصم:** لا تسمح بخصم يتجاوز سعر المنتج.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/logic-flaws/lab-flawed-enforcement-of-business-rules)
- [Business Logic Vulnerabilities](https://portswigger.net/web-security/logic-flaws)
