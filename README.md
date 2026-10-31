# دليل تشغيل موقع رئيس المرحلة الرابعة — محمد سالم

الموقع كله في **ملف واحد: `index.html`** (5 صفحات داخله: الرئيسية، الأخبار، التصويت، الدردشة، الملاحظات).
قاعدة البيانات **Firebase** والنشر المجاني على **GitHub Pages**.

## صلاحيات الرئيس
| القسم | ما يستطيع الرئيس فعله |
|---|---|
| الأخبار | نشر / تثبيت / حذف |
| التصويت | إنشاء استطلاع، إغلاقه وإعادة فتحه، حذفه |
| الدردشة | **قفل الدردشة، كتم مستخدم، حذف رسالة، مسح كل الرسائل** |
| الملاحظات | قراءة رسائل الطلبة، **الرد عليها وتعديل الرد أو حذفه**، تعليمها كمقروءة، حذفها |

الطالب يرى ردّ الرئيس في قسم **«رسائلي وردود الرئيس»** أسفل نموذج الملاحظات. الرسائل تُحفظ في متصفح الطالب (ولا تتأثر بتبديل الحساب المجهول)، وكل رسالة لها **رمز متابعة** يستطيع الطالب لصقه في خانة «لديك رمز متابعة؟» ليرى رسالته وردها من أي جهاز آخر. إن مسح بيانات المتصفح فعليه استخدام الرمز.

> ⚠️ **مهم جداً — حدّث قواعد Firestore:** انسخ القواعد الموجودة أدناه إلى تبويب **Rules** في Firebase واضغط **Publish**. القاعدة الخاصة بـ`feedback` تغيّرت (صارت `allow get` للجميع و`allow list` للرئيس)، وبدون نشرها لن تظهر الردود للطلبة.
> الرسائل المرسلة قبل هذا التحديث لا تُربط تلقائياً؛ أعد إرسال رسالة تجريبية جديدة للاختبار.

الطلبة يدخلون تلقائياً كمستخدمين مجهولين، والرئيس يدخل من زر **«دخول الرئيس»** في الأعلى.

---

## أولاً: إعداد Firebase

### 1) إنشاء المشروع
1. افتح <https://console.firebase.google.com> وسجّل بحساب Google.
2. **Add project** ← اكتب اسماً (مثل `class4-president`) ← أكمل الخطوات (يمكنك تعطيل Analytics).

### 2) تسجيل تطبيق ويب
1. في صفحة المشروع اضغط أيقونة الويب `</>`.
2. اكتب اسماً ← **Register app**.
3. انسخ الكائن `firebaseConfig` الذي سيظهر لك.
4. افتح `index.html` بمحرر نصوص (Notepad أو VS Code) وابحث عن `FIREBASE_CONFIG` في أول قسم `<script>`، ثم **الصق القيم مكان القيم الافتراضية**.

### 3) تفعيل المصادقة (Authentication)
1. **Build ← Authentication ← Get started**.
2. تبويب **Sign-in method** وفعّل:
   - **Anonymous** (للطلبة)
   - **Email/Password** (للرئيس)
3. تبويب **Users ← Add user**: أنشئ حساب الرئيس (بريد + كلمة مرور قوية).
4. ضع **نفس البريد** في مكانين:
   - `index.html` ← `ADMIN_EMAIL`
   - قواعد Firestore (الخطوة التالية) ← داخل `isAdmin()`

### 4) قاعدة البيانات (Firestore)
1. **Build ← Firestore Database ← Create database**.
2. اختر **Production mode** وأقرب منطقة.
3. تبويب **Rules** ← احذف كل شيء ← الصق القواعد التالية ← غيّر البريد ← **Publish**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function signedIn() {
      return request.auth != null;
    }

    // CHANGE THE EMAIL BELOW to the president email (same as ADMIN_EMAIL in index.html)
    function isAdmin() {
      return request.auth != null && request.auth.token.email == "president@example.com";
    }

    // ---------- news ----------
    match /news/{id} {
      allow read: if true;
      allow write: if isAdmin();
    }

    // ---------- polls ----------
    match /polls/{pid} {
      allow read: if true;
      allow write: if isAdmin();

      match /votes/{uid} {
        allow read: if true;
        allow create: if signedIn()
          && request.auth.uid == uid
          && get(/databases/$(database)/documents/polls/$(pid)).data.closed == false
          && request.resource.data.keys().hasOnly(['option', 'at'])
          && request.resource.data.option is int
          && request.resource.data.option >= 0
          && request.resource.data.option < get(/databases/$(database)/documents/polls/$(pid)).data.options.size();
        allow delete: if isAdmin();
      }
    }

    // ---------- feedback ----------
    match /feedback/{id} {
      allow create: if signedIn()
        && request.resource.data.keys().hasOnly(['name', 'type', 'message', 'uid', 'read', 'createdAt'])
        && request.resource.data.uid == request.auth.uid
        && request.resource.data.message is string
        && request.resource.data.message.size() >= 5
        && request.resource.data.message.size() <= 1000
        && request.resource.data.name is string
        && request.resource.data.name.size() <= 40
        && request.resource.data.read == false
        && request.resource.data.createdAt == request.time;
      allow get: if signedIn();
      allow list: if isAdmin();
      allow update: if isAdmin();
      allow delete: if isAdmin();
    }

    // ---------- chat ----------
    function chatLocked() {
      return exists(/databases/$(database)/documents/settings/chat)
        && get(/databases/$(database)/documents/settings/chat).data.locked == true;
    }

    function isMuted() {
      return exists(/databases/$(database)/documents/muted/$(request.auth.uid));
    }

    match /messages/{id} {
      allow read: if signedIn();
      allow create: if signedIn()
        && request.resource.data.keys().hasOnly(['text', 'name', 'uid', 'isPresident', 'createdAt'])
        && request.resource.data.uid == request.auth.uid
        && request.resource.data.isPresident == isAdmin()
        && request.resource.data.text is string
        && request.resource.data.text.size() > 0
        && request.resource.data.text.size() <= 500
        && request.resource.data.name is string
        && request.resource.data.name.size() > 0
        && request.resource.data.name.size() <= 60
        && request.resource.data.createdAt == request.time
        && (isAdmin() || (!chatLocked() && !isMuted()));
      allow delete: if isAdmin();
    }

    match /muted/{uid} {
      allow read: if signedIn();
      allow write: if isAdmin();
    }

    match /settings/{id} {
      allow read: if true;
      allow write: if isAdmin();
    }
  }
}
```

> المجموعات (news, polls, messages…) تُنشأ تلقائياً عند أول استخدام، لا تحتاج لإنشائها يدوياً.

---

## ثانياً: التجربة على جهازك (اختياري)
الصفحة تستخدم JavaScript modules فلا تعمل بالنقر المزدوج على الملف. استخدم:
- **VS Code + Live Server**: كليك يمين على `index.html` ← *Open with Live Server*.
- أو من الطرفية داخل المجلد: `python -m http.server 8000` ثم افتح `http://localhost:8000`.

---

## ثالثاً: النشر على GitHub Pages

### من المتصفح (بدون أوامر)
1. أنشئ حساباً في <https://github.com>.
2. **+ ← New repository** ← سمِّه `president-site` ← **Public** ← **Create repository**.
3. اضغط **uploading an existing file** ← اسحب `index.html` ← **Commit changes**.
4. **Settings ← Pages**.
5. **Source: Deploy from a branch** ← الفرع `main` والمجلد `/ (root)` ← **Save**.
6. بعد دقيقة أو دقيقتين يظهر الرابط: `https://USERNAME.github.io/president-site/`

### بالأوامر (Git)
```bash
git init
git add index.html
git commit -m "الإصدار الأول"
git branch -M main
git remote add origin https://github.com/USERNAME/president-site.git
git push -u origin main
```
وللتحديث لاحقاً: `git add . && git commit -m "تحديث" && git push`

---

## رابعاً: خطوة لا تنسها بعد النشر
أضف نطاق موقعك لـ Firebase وإلا لن يعمل تسجيل الدخول:

**Authentication ← Settings ← Authorized domains ← Add domain** ثم اكتب: `USERNAME.github.io`

---

---

## خامساً: نسخة التطبيق (APK)

الموقع صار **تطبيقاً قابلاً للتثبيت (PWA)**: فيه `manifest.json` و`sw.js` وأيقونات. يجب رفع **كل الملفات** إلى GitHub (وليس `index.html` وحده):
`index.html` و`manifest.json` و`sw.js` و`icon-192.png` و`icon-512.png` و`icon-maskable-192.png` و`icon-maskable-512.png`

### الطريقة 1 — تثبيت فوري بدون APK (الأسرع)
1. افتح رابط الموقع في **Chrome على الأندرويد**.
2. القائمة ⋮ ← **تثبيت التطبيق** (Install app) أو **إضافة إلى الشاشة الرئيسية**.
3. يظهر بأيقونته على الشاشة ويفتح بملء الشاشة كأي تطبيق.

### الطريقة 2 — ملف APK حقيقي (مجاني عبر PWABuilder)
1. أكمل نشر الموقع على GitHub Pages وتأكد أنه يعمل على الرابط `https://USERNAME.github.io/president-site/`.
2. افتح <https://www.pwabuilder.com> والصق الرابط ← **Start**.
3. انتظر الفحص (يجب أن يجد المانيفست والـService Worker).
4. اضغط **Package for stores** ← **Android** ← **Generate Package**.
5. اتركه على الإعدادات الافتراضية، ويُفضّل تغيير **Package ID** إلى شيء مثل `com.mohammedsalim.class4`، واترك **Signing key** على **New** ليُنشئ مفتاحاً.
6. حمّل الملف المضغوط: بداخله **APK** (للتثبيت المباشر) و**AAB** (لمتجر Google Play) و**مفتاح التوقيع**.
   - ⚠️ **احتفظ بملف المفتاح وكلمته**؛ بدونه لا يمكنك إصدار تحديثات لنفس التطبيق لاحقاً.
7. انقل ملف الـAPK إلى هاتف أندرويد وافتحه، واسمح بالتثبيت من مصدر غير معروف.

**شريط العنوان:** لإخفاء شريط الرابط داخل التطبيق يطلب منك PWABuilder إضافة ملف `assetlinks.json` (يعطيك نصّه في الحزمة) داخل مجلد `.well-known` في **جذر نطاقك**. على `github.io/president-site/` لا يمكن وضعه في الجذر، فسيعمل التطبيق لكن قد يظهر شريط رابط صغير في الأعلى. الحل الكامل: نطاق خاص بك أو مستودع باسم `USERNAME.github.io`.

> ملاحظة: الـAPK يفتح موقعك المباشر، لذا **أي تحديث على الموقع يظهر في التطبيق تلقائياً** دون إعادة بنائه. وعلى آيفون: Safari ← مشاركة ← إضافة إلى الشاشة الرئيسية (لا يوجد APK لآيفون).

## تخصيص الموقع
كل الإعدادات في أول قسم `<script>` داخل `index.html`:
- **الاسم، العنوان، الوصف:** داخل `SITE`.
- **صورة الرئيس (الطريقة الأسهل — من الموقع نفسه):** سجّل دخول الرئيس، سيظهر زر **«📷 تغيير الصورة»** تحت الصورة في الصفحة الرئيسية. اختر صورتك من جهازك (JPG أو PNG)، فتُقصّ تلقائياً بشكل مربع وتُحفظ في قاعدة البيانات وتظهر للجميع في البطاقة وفي الشعار العلوي، **بدون رفع ملفات على GitHub**. لا تحتاج تعديل القواعد (تستخدم `settings` الموجودة أصلاً). هذه الصورة لها الأولوية على `photoUrl`.
- **صورة الرئيس (الطريقة اليدوية):** ارفع الصورة إلى نفس مستودع GitHub بجانب `index.html` (مثلاً باسم `president.jpg`) واكتب اسمها في `photoUrl`، أو ضع رابط صورة كاملاً. وإن تُرك الحقل فارغاً يظهر الحرفان الأولان من الاسم.
- **الشعار في الزاوية العلوية:** يظهر داخل دائرة بجانب الاسم. افتراضياً يستخدم **صورة الرئيس** (`photoUrl`). إن أردت شعاراً مختلفاً ارفعه إلى GitHub واكتب اسمه في `logoUrl` (مثل `"logo.png"`). وإن لم توجد صورة يظهر الحرفان الأولان من الاسم.
- **أيقونات التواصل (إنستغرام، واتساب، تيليجرام):** تظهر كأزرار دائرية ذهبية **تحت صورة الرئيس** في الصفحة الرئيسية. املأ الروابط داخل `SITE`:

```js
photoUrl: "president.jpg",
telegram: "https://t.me/username",
whatsapp: "https://wa.me/9647XXXXXXXXX",
instagram: "https://instagram.com/username",
```

  - كل رابط تتركه فارغاً (`""`) لا تظهر أيقونته.
  - **واتساب:** الرقم بالصيغة الدولية بدون `+` ولا أصفار في البداية، مثل `9647701234567`.
  - **تيليجرام:** اسم المستخدم بدون `@` داخل الرابط.
- **الألوان:** المتغيرات في أعلى الـ`<style>` تحت `:root`.

---

## الأمان
- مفاتيح `firebaseConfig` **ليست سرية** وهي مصممة لتظهر في المتصفح؛ الحماية الحقيقية هي **قواعد Firestore**، فلا تنسَ نشرها.
- صلاحيات الرئيس تُفرض من الخادم عبر البريد، فلا يمكن انتحالها بتعديل الكود.
- لا تشارك كلمة مرور الرئيس، وفعّل التحقق بخطوتين على حساب Google الخاص بالمشروع.
- **حدود الكتم:** الطلبة بحسابات مجهولة، فإن مسح الطالب بيانات المتصفح قد يحصل على هوية جديدة ويتجاوز الكتم. لمنع أقوى، اجعل الدخول بحسابات Google.
- الخطة المجانية (Spark) تكفي لمرحلة دراسية كاملة، وراقب الاستخدام من **Firestore ← Usage**.

## حل المشاكل
| المشكلة | الحل |
|---|---|
| «تعذر الاتصال بـ Firebase» | تأكد من لصق `firebaseConfig` الصحيح وتفعيل **Anonymous** |
| لا يعمل دخول الرئيس | البريد يجب أن يتطابق في `ADMIN_EMAIL` والقواعد، وأضف `github.io` للنطاقات المسموحة |
| `Missing or insufficient permissions` | لم تُنشر القواعد بعد، أو البريد داخلها خطأ |
| صفحة بيضاء محلياً | استخدم Live Server ولا تفتح الملف مباشرة |
| صورة الرئيس لا تظهر (تظهر الحروف فقط) | اسم الملف في `photoUrl` يجب أن يطابق الملف المرفوع **حرفاً بحرف** (الأحرف الكبيرة والصغيرة مهمة، مثل `President.JPG` ≠ `president.jpg`)، وأن تكون الصورة في نفس مجلد `index.html` على GitHub. جرّب فتح `https://USERNAME.github.io/president-site/اسم-الصورة` مباشرة: إن لم تظهر هناك فالمشكلة في الرفع أو الاسم. يُفضّل تسمية الصورة بحروف إنجليزية بدون مسافات |
| التعديلات لا تظهر بعد الرفع | انتظر دقيقتين ثم `Ctrl + Shift + R` |
