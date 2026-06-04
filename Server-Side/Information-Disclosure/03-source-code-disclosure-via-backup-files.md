# Lab: Source code disclosure via backup files

> **Category:** Information Disclosure  
> **Difficulty:** APPRENTICE  
> **Lab Link:** [PortSwigger Lab - Source code disclosure via backup files](https://portswigger.net/web-security/information-disclosure/lab-source-code-disclosure-via-backup-files)

---

## 🎯 الهدف الرئيسي

استغلال وجود ملفات احتياطية (backup files) في دليل مخفي، والتي تكشف الكود المصدري (source code) للتطبيق. يحتوي الكود المصدري على **كلمة مرور قاعدة البيانات** (database password) مكتوبة بشكل ثابت (hard-coded).

**المطلوب:** استخراج كلمة مرور قاعدة البيانات وإرسالها.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: البحث في robots.txt

- افتح المختبر في المتصفح
- أضف `/robots.txt` إلى الرابط:

```
https://YOUR-LAB-ID.web-security-academy.net/robots.txt
```

**النتيجة:** يظهر المحتوى:

```
User-agent: *
Disallow: /backup
```

هذا يكشف وجود دليل `/backup`!

### الخطوة 2: زيارة دليل /backup

- اذهب إلى:

```
https://YOUR-LAB-ID.web-security-academy.net/backup
```

**النتيجة:** تظهر قائمة الملفات:

```
ProductTemplate.java.bak
```

### الخطوة 3: تحميل ملف النسخة الاحتياطية

- اذهب إلى:

```
https://YOUR-LAB-ID.web-security-academy.net/backup/ProductTemplate.java.bak
```

### الخطوة 4: قراءة الكود المصدري

ستظهر صفحة تحتوي على كود Java:

```java
package data;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class ProductTemplate {
    private static final String DB_HOST = "localhost";
    private static final String DB_NAME = "products";
    private static final String DB_USER = "postgres";
    private static final String DB_PASSWORD = "g7s8df7as8df7as8df7as8df7asdf";

    public static Connection getConnection() throws SQLException {
        String url = "jdbc:postgresql://" + DB_HOST + "/" + DB_NAME;
        return DriverManager.getConnection(url, DB_USER, DB_PASSWORD);
    }
}
```

**المعلومة المطلوبة:** `DB_PASSWORD = "g7s8df7as8df7as8df7as8df7asdf"`

### الخطوة 5: إرسال الإجابة

- ارجع إلى صفحة المختبر
- اضغط **Submit solution**
- الصق قيمة كلمة المرور

### الخطوة 6: حل المختبر

بعد إرسال كلمة المرور الصحيحة، سيتم حل المختبر ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **كيف اكتشفنا دليل /backup؟** | من ملف `robots.txt` الذي يكشف مسارات مخفية |
| **ما هو ملف `.java.bak`؟** | ملف احتياطي للكود المصدري (backup) |
| **ما هي المعلومات المسربة؟** | كلمة مرور قاعدة بيانات PostgreSQL |
| **لماذا هذا خطير؟** | المهاجم يمكنه الاتصال بقاعدة البيانات مباشرة وسرقة أو تعديل البيانات |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

1. وجود ملفات احتياطية (`.bak`, `.old`, `.swp`, `~`) في دليل يمكن الوصول إليه
2. كلمات المرور مكتوبة بشكل ثابت (hard-coded) في الكود المصدري
3. ملف `robots.txt` يكشف مسارات حساسة

---

### ❌ Non-Compliant Code (Java)

```java
// ProductTemplate.java - الكود المصاب
package data;

import java.sql.Connection;
import java.sql.DriverManager;

public class ProductTemplate {
    // خطأ: كلمة المرور مكتوبة بشكل ثابت في الكود
    private static final String DB_PASSWORD = "g7s8df7as8df7as8df7as8df7asdf";
    
    public static Connection getConnection() throws Exception {
        // خطأ: معلومات الاتصال مكشوفة
        String url = "jdbc:postgresql://localhost/products";
        return DriverManager.getConnection(url, "postgres", DB_PASSWORD);
    }
}
```

```nginx
# إعدادات الخادم - لا تمنع الوصول إلى الملفات الاحتياطية
# ملف robots.txt يكشف المسار
```

---

### ✅ Compliant Code (Next.js + Environment Variables)

```javascript
// إصلاح 1: استخدام متغيرات البيئة
// .env.production - لا يتم رفعه إلى Git
DB_HOST=localhost
DB_NAME=products
DB_USER=postgres
DB_PASSWORD=super-secret-password-not-in-code

// lib/db.js
import { Pool } from 'pg';

// إصلاح 2: قراءة كلمة المرور من متغيرات البيئة
const pool = new Pool({
  host: process.env.DB_HOST,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  // إصلاح 3: استخدام SSL وقيود إضافية
  ssl: process.env.NODE_ENV === 'production',
  max: 20,
});

export default pool;
```

```nginx
# إصلاح 4: إعدادات Nginx - منع الوصول إلى الملفات الاحتياطية
location ~* \.(bak|old|swp|~|java|class)$ {
    deny all;
    return 403;
}

# إصلاح 5: إخفاء المسارات الحساسة في robots.txt
# robots.txt - لا تضع مسارات حساسة
User-agent: *
Disallow: /
```

```javascript
// next.config.js - منع الوصول إلى ملفات معينة
module.exports = {
  async headers() {
    return [
      {
        source: '/:path*\\.(bak|old|swp|java)$',
        headers: [
          {
            key: 'X-Robots-Tag',
            value: 'noindex, nofollow',
          },
        ],
      },
    ];
  },
};
```

**الخلاصة:** 
1. استخدم متغيرات البيئة لتخزين كلمات المرور والمفاتيح السرية
2. لا ترفع الملفات الاحتياطية إلى الخادم
3. قم بتكوين الخادم لمنع الوصول إلى ملفات `.bak`, `.old`, `.swp`
4. لا تضع مسارات حساسة في `robots.txt`

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **استخدام متغيرات البيئة (Environment Variables):** لا تضع كلمات المرور في الكود.
2. **إزالة الملفات الاحتياطية:** احذف `.bak`, `.old`, `~` من الخادم.
3. **تكوين الخادم:** امنع الوصول إلى أنواع الملفات الخطيرة.
4. **إخفاء `robots.txt` الحساس:** لا تضع مسارات إدارية أو مخفية فيه.
5. **استخدم `.gitignore`:** لمنع رفع الملفات الحساسة إلى المستودع.

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/information-disclosure/lab-source-code-disclosure-via-backup-files)
- [Information Disclosure Cheat Sheet](https://portswigger.net/web-security/information-disclosure)
- [Environment Variables Best Practices](https://12factor.net/config)
