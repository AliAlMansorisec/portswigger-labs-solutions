# Lab: Low-level logic flaw

> **Category:** Business Logic  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Low-level logic flaw](https://portswigger.net/web-security/logic-flaws/lab-low-level-logic-flaw)

---

## 🎯 الهدف الرئيسي

استغلال ثغرة منطق العمل (Business Logic) على مستوى منخفض (Low-level) ناتجة عن **تجاوز الحد الأقصى للقيمة الصحيحة (Integer Overflow)** في لغة البرمجة الخلفية. بإضافة عدد كبير جداً من المنتجات إلى السلة، سيتجاوز السعر الإجمالي الحد الأقصى (`2,147,483,647`) ويلتف إلى قيمة سالبة كبيرة، ثم نضبط الكمية لتصبح بين `$0` و `$100`.

**الحساب:** `wiener:peter`

**المطلوب:** شراء "Lightweight l33t leather jacket".

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: تسجيل الدخول وفهم العملية

- سجل الدخول بـ `wiener:peter`
- لاحظ أن رصيدك محدود (مثلاً `$100`)
- سعر السترة الجلدية: `$1337` (أكثر من رصيدك)

### الخطوة 2: فحص طلب الإضافة إلى السلة

- أضف السترة إلى السلة
- اعترض طلب `POST /cart` في Burp:

```http
POST /cart HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded

productId=1&redir=PRODUCT&quantity=1&price=133700
```

**ملاحظة:** السعر بـ **cents** (133700 سنت = $1337).

### الخطوة 3: فهم قيود الكمية

- جرب تغيير `quantity` إلى رقم مكون من 3 أرقام (مثل `100`)
- **النتيجة:** يتم رفض الطلب أو تقطيعه إلى رقمين فقط!
- الحد الأقصى للكمية في الطلب الواحد هو `99`

### الخطوة 4: إعداد هجوم Intruder (إضافة 99 قطعة لكل طلب)

- أرسل الطلب إلى **Intruder** (Ctrl+I أو زر الماوس الأيمن → Send to Intruder)
- عيّن موضع الهجوم (payload position) على قيمة `quantity`
- اترك القيمة الافتراضية `1` أو أي رقم مؤقت

**إعدادات Intruder:**

| الإعداد | القيمة |
|---------|--------|
| Payload type | Null payloads |
| Payload configuration | Continue indefinitely |
| عدد الطلبات المتزامنة (Resource pool) | 1 (Maximum concurrent requests = 1) |

### الخطوة 5: بدء الهجوم ومراقبة السعر

- ابدأ الهجوم (Start attack)
- في المتصفح، اذهب إلى **Cart** وقم بتحديث الصفحة كل بضع ثوانٍ
- راقب السعر الإجمالي

**ماذا سيحدث؟**

السعر يبدأ في الزيادة:
```
$1,337 → $2,674 → $4,011 → ... → $2,147,483,647 (الحد الأقصى)
```

عند تجاوز الحد الأقصى، **يحدث Integer Overflow**:

```
$2,147,483,647 + $1,337 = -$2,147,483,648
```

ثم يبدأ السعر في **الزيادة من الرقم السالب إلى الصفر**:
```
-$2,147,483,648 → -$2,147,482,311 → ... → $0 → $1,337
```

### الخطوة 6: حساب عدد الطلبات المطلوبة

سعر السترة بـ **cents**: `133700`

الحد الأقصى للـ integer: `2,147,483,647`

نحتاج إلى الوصول إلى سعر إجمالي يقع بين `$0` و `$100` (أي بين `0` و `10000` سنت).

**الهدف:** الوصول إلى سعر سالب صغير (قريب من الصفر) ثم إضافة منتج آخر لتعديله.

### الخطوة 7: إعداد هجوم بعدد محدد من الطلبات

- في Intruder، بدلاً من "Continue indefinitely"، اختر **Exactly** مع عدد محدد من البايلودات
- جرب `323` طلباً (كل طلب يضيف 99 قطعة)

### الخطوة 8: إضافة الكمية النهائية

بعد انتهاء الهجوم، ستجد أن السعر الإجمالي أصبح قيمة سالبة أو صغيرة.

**مثال:** قد يصل السعر إلى `-$1,221.96` (أي `-122196` سنت).

### الخطوة 9: تعديل السعر باستخدام منتج آخر

- أرسل طلب `POST /cart` عادي إلى Repeater
- أضف كمية من منتج آخر (مثل منتج رخيص) لتعديل السعر الإجمالي

الهدف: جعل السعر الإجمالي بين `$0` و `$100`.

**مثال:**
```
السعر الحالي: -$1,221.96
أضف 13 منتجاً بسعر $100 (إجمالي $1,300)
السعر الجديد: $78.04
```

### الخطوة 10: إتمام الشراء

- عندما يصبح السعر الإجمالي أقل من رصيدك (بين `$0` و `$100`)
- اضغط على **Place order**

### الخطوة 11: حل المختبر

بعد شراء السترة، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو Integer Overflow؟** | عندما تتجاوز القيمة الحد الأقصى للرقم الصحيح (`2,147,483,647`)، تلف إلى الرقم السالب الأصغر (`-2,147,483,648`) |
| **لماذا هذا خطأ؟** | الخادم يستخدم نوع بيانات غير مناسب (32-bit integer) للسعر، ولا يتحقق من التجاوز |
| **كيف نستغلها؟** | نضيف كمية كبيرة جداً حتى يتجاوز السعر الحد الأقصى ويصبح سالباً |
| **لماذا 99 قطعة لكل طلب؟** | الخادم يقبل فقط كميات مكونة من رقمين (max 99) في الطلب الواحد |

---

## 📊 شرح Integer Overflow

```
حدود الـ 32-bit signed integer:
الحد الأقصى:  2,147,483,647
الحد الأدنى: -2,147,483,648

عند إضافة 1 إلى الحد الأقصى:
2,147,483,647 + 1 = -2,147,483,648

عند إضافة 1,337 (سعر السترة بـ cents):
2,147,483,647 + 1,337 = -2,147,482,311
```

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

استخدام `int` (32-bit) لتخزين السعر الإجمالي، وعدم التحقق من تجاوز الحد الأقصى (integer overflow).

---

### ❌ Non-Compliant Code (Next.js)

```javascript
// pages/api/cart.js - حساب السعر الإجمالي
export default function handler(req, res) {
  const { sessionId } = req.cookies;
  const cart = getCart(sessionId);
  
  // خطأ: استخدام 32-bit integer (JavaScript يستخدم Number لكن الخادم الخلفي قد يكون بلغة أخرى)
  // في هذه الحالة، اللغة الخلفية تستخدم 32-bit int
  let totalPrice = 0;
  
  for (const item of cart.items) {
    totalPrice += item.price * item.quantity; // يمكن أن يتجاوز الحد الأقصى
  }
  
  if (totalPrice > user.storeCredit) {
    return res.status(400).json({ error: 'Insufficient credit' });
  }
  
  // معالجة الطلب
}
```

---

### ✅ Compliant Code (Next.js)

```javascript
// pages/api/cart.js - حساب السعر الإجمالي
export default function handler(req, res) {
  const { sessionId } = req.cookies;
  const cart = getCart(sessionId);
  
  // التصحيح 1: استخدام BigInt أو 64-bit integer
  let totalPrice = 0n; // BigInt في JavaScript
  
  for (const item of cart.items) {
    const itemTotal = BigInt(item.price) * BigInt(item.quantity);
    totalPrice += itemTotal;
  }
  
  // التصحيح 2: التحقق من أن السعر موجب
  if (totalPrice < 0n) {
    return res.status(400).json({ error: 'Price overflow detected' });
  }
  
  // التصحيح 3: التحقق من الحد الأقصى المعقول
  const MAX_REASONABLE_PRICE = 1000000n; // $10,000
  if (totalPrice > MAX_REASONABLE_PRICE) {
    return res.status(400).json({ error: 'Price exceeds maximum' });
  }
  
  if (totalPrice > BigInt(user.storeCredit)) {
    return res.status(400).json({ error: 'Insufficient credit' });
  }
  
  // معالجة الطلب
}
```

**الخلاصة:** 
1. استخدم نوع بيانات مناسب (64-bit أو BigInt)
2. تحقق من تجاوز الحدود (overflow detection)
3. حدد حداً أقصى معقولاً للسعر الإجمالي

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **استخدم 64-bit integer:** بدلاً من 32-bit لتجنب التجاوز السريع.
2. **تحقق من التجاوز (overflow check):** قبل وبعد العمليات الحسابية.
3. **حدد حداً أقصى للكمية:** لا تسمح بإضافة كميات غير معقولة.
4. **استخدم BigInt:** في اللغات التي تدعم الأعداد الكبيرة.
5. **تحقق من صحة السعر:** تأكد أن السعر الإجمالي بين 0 والحد الأقصى المعقول.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/logic-flaws/lab-low-level-logic-flaw)
- [Integer Overflow Explained](https://portswigger.net/web-security/logic-flaws/examples#integer-overflow)
- [Business Logic Vulnerabilities](https://portswigger.net/web-security/logic-flaws)
