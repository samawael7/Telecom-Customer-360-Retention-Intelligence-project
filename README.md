# 📡 Telecom Customer 360 & Retention Intelligence

نظام Data Warehouse متكامل لشركة تليكوم، بيوحّد 3 أنظمة منفصلة (CRM، Billing، Customer Service) في نجمة بيانات واحدة (Star Schema)، بيحافظ على تاريخ العميل كامل عبر الزمن (SCD Type 2)، وبيطلّع قايمة أولويات احتفاظ بالعملاء (Retention Priority List) مبنية على بيانات حقيقية ومقاييس قابلة للتنفيذ — من غير أي فلترة أو تجميل، مع نظام رفض وتوثيق كامل لأي بيانات معرفناش نربطها.

**الأدوات المستخدمة:** `Informatica PowerCenter` (ETL) · `Oracle Database` · `Power BI` (Reporting)

---

## 📌 نظرة عامة على المشروع

الهدف: بناء "Customer 360" حقيقي — نظام واحد بيجاوب على سؤال **"مين العميل ده فعليًا؟"** عبر 3 أنظمة مختلفة (CRM بيعرفه برقمه القومي، Billing وCustomer Service بيعرفوه برقم تليفونه بس)، مع الحفاظ الكامل على تاريخ تغييراته، عشان في الآخر نقدر نحسب قيمته الحقيقية ودرجة خطورة فقدانه، ونوجّه فريق الـRetention لمين يتصرف معاه أولًا وليه بالتحديد.

---

## 🏗️ الـArchitecture

CRM_CUSTOMER ──┐
BILLING_TRANSACTION ──┤──→ ETL (Informatica) ──→ Oracle Star Schema ──→ Power BI
SERVICE_TICKET ──┘


<img width="1339" height="635" alt="Architecture Diagram" src="https://github.com/user-attachments/assets/3ee352e8-9005-43d4-b765-0e8d99be19e1" />

| النوع | الجداول |
|:---|:---|
| **Dimensions** | `DIM_CUSTOMER` (SCD2), `DIM_DATE`, `DIM_LOCATION`, `DIM_PLAN`, `DIM_PAYMENT_METHOD`, `DIM_SERVICE_CHANNEL` |
| **Facts** | `FACT_BILLING`, `FACT_CUSTOMER_SERVICE` |
| **Aggregation** | `RETENTION_PRIORITY_LIST` |
| **Data Quality** | `ETL_REJECT_RECORD` |

---

## 🔑 القرارات التصميمية الأساسية

### 1) Identity Resolution — إزاي بنحدد "مين العميل ده فعليًا؟"

- **CRM هو مصدر الحقيقة (Master Source)**، وبيولّد `GOLDEN_CUSTOMER_ID` باستخدام **National ID** كمفتاح أساسي — أقوى وأدق معرّف متاح، لأنه ثابت ومتحقق منه رسميًا.
- **Billing وCustomer Service** مالهومش رقم قومي، بس عندهم **Phone**. بيتربطوا بالعميل الصح عن طريق مطابقة رقم التليفون بالنسخة الحالية في `DIM_CUSTOMER`.
- **الإيميل اتلغى عمدًا من المطابقة** — قرار واقعي: الإيميل بيتسجل مرة واحدة وقت التسجيل وغالبًا بيبقى قديم، بينما رقم التليفون مرتبط بالخط النشط الفعلي.

### 2) SCD Type 2 — الحفاظ على تاريخ العميل كامل

كل تغيير في بيانات العميل (رقم تليفون، خطة، حالة اشتراك) بيتسجل كنسخة جديدة، مش استبدال:
- **النسخة القديمة:** `IS_CURRENT='N'`, `EFFECTIVE_END_DATE` = تاريخ التغيير − يوم
- **النسخة الجديدة:** `IS_CURRENT='Y'`, `EFFECTIVE_END_DATE = 31-DEC-9999`
- **الاتنين بياخدوا نفس `GOLDEN_CUSTOMER_ID`** — عشان نعرف إنهم نفس الشخص عبر الزمن.

> فاتورة قديمة اتحملت وقت ما العميل كان بياناته مختلفة، لازم تفضل مرتبطة بنسخته التاريخية الصح — لو فلترنا على `IS_CURRENT='Y'` بدري قبل الأوان، كنا هنفقد ربط الفواتير القديمة بالكامل من حسابات الإيراد.

### 3) منهج Static Lookup بدل Dynamic Lookup Cache

جُرّب في البداية Dynamic Lookup Cache للـidentity resolution، لكن اتكشف فيه خطر حقيقي: خاصية "Insert Else Update" ممكن تكتب فوق `GOLDEN_CUSTOMER_ID` الصح بقيمة غلط لو القيم الجاية مختلفة عن الكاش.
**الحل النهائي:** Static Lookups (قراءة بس)، مع تقنية "ذاكرة بين الصفوف" (variable ports) داخل الـExpression نفسها — أضمن وأبسط في الـdebugging.

### 4) حماية كاملة من التكرار عند إعادة التشغيل (Idempotency)

كل مابينج فيها Lookup إضافي بيتأكد إن السجل مش موجود بالفعل قبل أي insert، عشان تشغيل الـWorkflow أكتر من مرة **ميكررش أي بيانات ولا يفقد أي حاجة**. اتّختبر عمليًا: تشغيل الـWorkflow مرتين متتاليتين من غير بيانات جديدة أدّى لنفس الأرقام بالظبط في كل الجداول، وإضافة عميل واحد جديد زوّدت كل جدول متأثر بواحد بالظبط.

---

## 📊 حساب RETENTION_PRIORITY_LIST — المعادلات كاملة

### VALUE_SCORE (0–100)

مبني على **Percentile Rank** لإجمالي الإيراد لكل عميل مقارنة بباقي العملاء (مش رقم ثابت):

VALUE_SCORE = (1 − (ترتيب العميل حسب الإيراد − 1) / (عدد العملاء − 1)) × 100


### RISK_SCORE (0–100)

RISK_SCORE = MIN(100,
(المبلغ المتأخر ÷ إجمالي الإيراد) × 40

(عدد التذاكر عالية الأولوية) × 10
(لو متوسط الرضا < 3 من 5 → +20)
)

### RETENTION_PRIORITY (التصنيف النهائي)

| الشرط | التصنيف |
|:---|:---|
| قيمة عالية + خطورة عالية جدًا (تذاكر متصعّدة/مفتوحة كتير) | 🔴 **CRITICAL** |
| قيمة عالية + خطورة عالية (رضا منخفض أو Risk Score مرتفع) | 🟠 **HIGH_VALUE_AT_RISK** |
| خطورة متوسطة/عالية بدون قيمة عالية، أو شكاوى متكررة | 🟡 **MONITOR** |
| قيمة عالية + خطورة منخفضة + رضا عالي | 🟢 **HIGH_VALUE_STABLE** |
| غير كده | ⚪ **LOW_PRIORITY** |

كل عميل معاه كمان `TOP_ISSUE_TYPE` (أكتر نوع مشكلة متكررة معاه) و`RECOMMENDED_ACTION` نص ديناميكي بيذكر نوع المشكلة نفسها — القايمة قابلة للتنفيذ المباشر، مش بس تصنيف.

> **قرار بيزنس:** أي عميل مالوش أي فاتورة خالص مُستبعد من القايمة — مفيش معنى نحسب "قيمة" لعميل معندوش إيراد أصلاً.

---

## ⚠️ Data Quality & Reject Handling

أي سجل معرفناش نربطه بعميل معروف (رقم تليفون مش متطابق، رقم قومي غلط الفورمات، هوية مكررة) **مش بيتجاهل** — بيتسجل في `ETL_REJECT_RECORD` مع السبب الدقيق. ده مش error handling بس، ده **KPI بيزنس حقيقي**: نسبة البيانات اللي مش متزامنة بين الأنظمة (~6–8% في هذا المشروع، رقم واقعي لأي نظام حقيقي).

---

## 📈 Power BI Dashboards

### 1️⃣ Overview Dashboard
نظرة سريعة وشاملة للإدارة على الإيرادات وحجم المخاطر.

**KPI Cards**
| Card | بيجاوب على |
|:---|:---|
| **Total Revenue** | إجمالي الإيرادات المحققة من كل العملاء |
| **Customers At Risk** | عدد العملاء المعرّضين لخطر ترك الخدمة |
| **Active Customers** | حجم القاعدة النشطة من العملاء حاليًا |
| **Revenue at Risk** | قيمة الإيرادات المهددة بالضياع بسبب العملاء المعرّضين للخطر |

**Visuals**
- **Avg Revenue per Customer by Priority** (Bar Chart) — هل العملاء الأهم فعلاً الأكثر إيرادًا؟
- **Retention Priority Distribution** (Donut Chart) — توزيع العملاء بحسب فئات الخطورة
- **Revenue at Risk by Customer Tenure** (Stacked Bar Chart) — الإيرادات المعرّضة للخطر موزّعة حسب مدة بقاء العميل معنا

<img width="1256" height="694" alt="Overview Dashboard" src="https://github.com/user-attachments/assets/3310e05a-9b25-47e6-abd4-8b8778beebe6" />

---

### 2️⃣ Retention Priority Dashboard
تحليل عميق للعملاء الأكثر خطورة وأسباب المشاكل، لتحديد التدخلات السريعة.

**KPI Cards**
| Card | بيجاوب على |
|:---|:---|
| **Critical Customers** | عدد العملاء في الحالة الحرجة القصوى |
| **High Value At Risk** | عدد العملاء ذوي القيمة العالية المعرّضين للخطر |
| **Avg Satisfaction – At Risk** | متوسط رضا العملاء المعرّضين للخطر |
| **Avg Customer Value – At Risk** | متوسط القيمة المالية لهؤلاء العملاء |

**Visuals**
- **Risk vs. Satisfaction** (Scatter Plot) — العلاقة بين درجة الخطورة ومستوى الرضا، وأين يتجمّع عملاء الخطر
- **Issues Driving Retention Risk** (Bar Chart) — أبرز الأسباب الجذرية (Billing, Internet, Network, Plan...) اللي بتدفع العملاء نحو الخطر
- **Customer Table** (Detailed Grid) — تفاصيل العملاء كاملة (الأرقام، درجات الخطورة، القيمة، الإيرادات، عدد الشكاوى) لاتخاذ إجراء مباشر

<img width="1248" height="697" alt="Retention Priority Dashboard" src="https://github.com/user-attachments/assets/6421eeff-b987-4b9a-b292-84c33089ae0b" />

---

### 3️⃣ Customer Service Dashboard
متابعة أداء الدعم الفني، حالة التذاكر، وسرعة الاستجابة.

**KPI Cards**
| Card | بيجاوب على |
|:---|:---|
| **Open Tickets** | عدد التذاكر المفتوحة اللي لسه محتاجة متابعة |
| **Complaint Count** | إجمالي عدد التذاكر المسجلة |
| **Avg Resolution Time** | متوسط الوقت المستغرق لحل وإغلاق التذاكر |

**Visuals**
- **Ticket Status by Customer Count** (Pie Chart) — توزيع حالة التذاكر (Resolved, Open, In Progress, Escalated)
- **Resolution Time vs Customer Satisfaction by Issue** (Combo Chart) — هل نوع المشكلة بيأثر على سرعة الحل، وهل ده بينعكس على رضا العملاء؟
- **Satisfaction Trend** (Line Chart) — تطور مستوى رضا العملاء عبر الوقت

<img width="1256" height="697" alt="Customer Service Dashboard" src="https://github.com/user-attachments/assets/9fac0baa-a9b6-4d0b-a516-3e1151f77a53" />

---

## 📁 هيكل المشروع

telecom-customer-retention-etl/
├── informatica/ # XML exports للـmappings والـworkflow
├── sql/ # سكريبتات إنشاء الجداول، توليد بيانات إضافية، استعلامات التحقق
├── powerbi/ # ملف الـ.pbix
├── docs/ # خطة المشروع الكاملة وخطة الـPower BI
└── README.md


---

## 🛠️ أبرز التحديات التقنية اللي اتحلّت

| المشكلة | الأثر | الحل |
|:---|:---|:---|
| **NULL matching في Static Lookup Cache** | رقم تليفون فاضي اتربط غلط بأول عميل NULL في الكاش (عميل واحد ابتلع 210 فاتورة بالغلط) | شرط صريح `NOT ISNULL()` قبل تصديق أي نتيجة Lookup |
| **Column truncation صامت** | عمود `CHANNEL_NAME` قصّ قيمة "Call Center" بصمت، فوّت 1595 صف من المطابقة | توسيع العمود + مراجعة كل أعمدة الأبعاد |
| **Case-sensitivity mismatch** | قيم `PAYMENT_METHOD` بحالة أحرف مختلفة بين المصدر والـDimension (`CASH` مقابل `Cash`) | توحيد الحالة قبل أي مقارنة |
| **Full-history reprocessing على target غير فاضي** | إعادة تشغيل SCD2 على مصدر كامل بعد ما الـtarget بقى فيه بيانات، سبب تعارض في القيود | منطق يفرّق بين بيانات "اتحمّلت قبل كده" و"تحديث حقيقي جديد" |

---

## ✅ التحقق من الجودة (Verified)

- ✔️ **Idempotency test** — تشغيل الـWorkflow مرتين من غير بيانات جديدة → صفر تغيير في كل الجداول
- ✔️ **End-to-end test** — إضافة عميل جديد كامل (CRM + فاتورة + تذكرة) → كل جدول زاد بالظبط بواحد
- ✔️ **Orphan record check** — صفر سجلات "يتيمة" في كل الـFacts وRETENTION_PRIORITY_LIST
- ✔️ **Data reconciliation** — كل صفوف المصدر = صفوف ناجحة + صفوف مرفوضة موثّقة، بالظبط
