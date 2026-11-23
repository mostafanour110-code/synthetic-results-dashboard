# SyntheticResultsDashboard — لوحة نتائج تجريبية

**المشروع مستقل ومحلي وتعليمي ببيانات مصطنعة فقط.**

Portfolio demo for Mostafa Ahmed. AI-assisted demo, not client work. No employer code, employer data, or real personal records are included.

![Desktop results](screenshots/results-desktop.png)

[Mobile results screenshot](screenshots/results-mobile.png)
 ليس تنفيذًا لنظام عميل، ولا يصلح للنشر الإنتاجي أو لاستيراد بيانات حقيقية. لا حسابات مستخدمين ولا Authentication. **لا تعرضه على الشبكة.**

## 1) المتطلبات والتقنيات والتراخيص

- .NET 8 SDK، وليس مجرد ASP.NET Runtime.
- ASP.NET Core Razor Pages، HTML/CSS محلي بدون CDN، صفحات عربية `lang=ar` و`dir=rtl`.
- EF Core 8.0.32 + Microsoft.EntityFrameworkCore.Sqlite 8.0.32: **MIT**.
- ClosedXML 0.105.1: **MIT**، لقراءة ملفات XLSX وكتابتها فقط. لا تدعم هذه النسخة XLS/XLSM/CSV.
- xUnit 2.9.3 وxunit.runner.visualstudio 3.1.4: **Apache-2.0**؛ Microsoft.NET.Test.Sdk 17.13.0 وMicrosoft.AspNetCore.Mvc.Testing 8.0.32: **MIT**. راجع تراخيص الحزم الانتقالية عند إعادة التوزيع.
- SQLite قاعدة محلية تُنشأ تلقائيًا داخل مجلد `src/SyntheticResultsDashboard.Web/App_Data/synthetic.db` (قد يختلف الجذر مع `ContentRootPath`).

الإصدارات مثبتة في `*.csproj`، وجميع المكتبات تُسترجع من NuGet أثناء `dotnet restore`. لا توجد مكتبات JavaScript خارجية ولا اتصالات شبكية مطلوبة أثناء استخدام النموذج؛ **استعادة الحزم للمرة الأولى تتطلب وصولًا إلى NuGet أو cache محليًا**.

## 2) التشغيل من مجلد الحل (PowerShell)

```powershell
dotnet --info
dotnet restore .\SyntheticResultsDashboard.sln
dotnet test .\SyntheticResultsDashboard.sln --no-restore
dotnet run --project .\src\SyntheticResultsDashboard.Web\SyntheticResultsDashboard.Web.csproj --no-restore
```

الرابط المحلي فقط: **http://127.0.0.1:5087**. التطبيق يضبط Kestrel على loopback صراحة، ولا يوجد تكوين للنشر على IIS أو استضافة عامة.

تُطبّق Migration ابتدائية واحدة تلقائيًا بـ `Database.Migrate()` عند التشغيل. **لا تستخدم `EnsureCreated`.** لتطبيق Migration يدويًا (إذا كان `dotnet-ef` 8 متاحًا):

```powershell
dotnet ef database update --project .\src\SyntheticResultsDashboard.Infrastructure --startup-project .\src\SyntheticResultsDashboard.Web
```

**لا تنفذ `migrations add InitialCreate` ثانية**: ملف Migration موجود بالفعل ضمن الشفرة. يمكن حذف قاعدة تجريبية محلية مغلقة وبدء قاعدة جديدة عند الحاجة؛ لا تحذف قاعدة لها بيانات تريد الاحتفاظ بها.

## 3) هيكل المشروع واتجاه الاعتماد

```text
SyntheticResultsDashboard/
  SyntheticResultsDashboard.sln
  src/
    SyntheticResultsDashboard.Domain/         # Entities, enum, pure validation
    SyntheticResultsDashboard.Application/    # Interfaces, orchestrated use cases
    SyntheticResultsDashboard.Infrastructure/ # EF, migration, ZIP/OOXML, file store, Excel
    SyntheticResultsDashboard.Web/            # Razor Pages, CSS, composition root
  tests/
    SyntheticResultsDashboard.Tests.Unit/
    SyntheticResultsDashboard.Tests.Integration/
  samples/synthetic-demo-template.xlsx
  TEST_RESULTS.md
```

`Domain ← Application ← Infrastructure`؛ طبقة `Web` تعتمد على `Application/Infrastructure` لربط الاعتماديات فقط. لا توجد REST API أو Blazor أو React. لا استعلامات EF ولا قواعد مطابقة داخل ملفات cshtml.

## 4) طريقة التجربة

1. افتح `/`؛ جميع الإحصاءات تبدأ صفرًا دون أي seed آلي.
2. انتقل إلى `/Demo` واضغط **إضافة بيانات مرجعية تجريبية**. تضاف سجلات DEMO-001 إلى DEMO-004 فقط، على نحو idempotent، مع رفض التعارض.
3. نزّل القالب من `/Demo/Template` (يوجد أيضًا في `samples/`).
4. ارفع XLSX من `/Imports/New`. تُفحص سلامة بنية المصنف وأسماء أوراقه عند الرفع، لكن لا تبدأ مطابقة الصفوف في هذه المرحلة؛ تحفظ نسخة مؤقتة باسم عشوائي **خارج wwwroot**.
5. اختر الورقة المرئية `DemoResults` في `/Imports/SelectSheet/{id}` ثم اضغط **فحص ومطابقة**.
6. شاهد `/Imports/Details/{id}` بفلتر للحالة وترقيم صفحات ثابت 25، ثم صدّر **جميع** الصفوف من `/Imports/Export/{id}`.

النتيجة المرجعية للقالب بعد seed: `TotalRows=6`، `Matched=1`، `NotFound=1`، `DepartmentMismatch=1`، `Invalid=3`. بطاقة غير مطابق = 2. تكرار الكود يلغي كل صفوفه، بما فيها أول ظهور.

## 5) عقد بيانات XLSX

- الحد المضغوط `2 MiB`، والحد الإجمالي للأجزاء المفكوكة `20 MiB`، حتى `200` جزء ZIP، حتى `10` أوراق، مع عرض الأوراق المرئية فقط.
- الورقة: الصف الأول `ReferenceCode, DisplayName, Department, Score` بنفس الترتيب وبعد Trim للعناوين فقط؛ حتى 1000 صف بيانات غير فارغ، آخر صف مستخدم <= 2001، آخر عمود <= D.
- `ReferenceCode`: نص يبدأ بحرف لاتيني كبير أو رقم، ثم أحرف `A-Z` أو أرقام أو `-`، طول 1–30. الخلايا الرقمية مرفوضة؛ الحروف الصغيرة لا تتحول تلقائيًا.
- `DisplayName`: نص مصطنع 1–100، للعرض فقط.
- `Department`: نص يساوي `SCI` أو `ART` تمامًا.
- `Score`: رقم Excel حقيقي من 0 إلى 100 وبحد أقصى منزلتين عشريتين. لا تقبل الصيغ/نصوص الأرقام/التواريخ/القيم المنطقية.
- يرفض المصنف إذا كان غير سليم أو يتجاوز حدود ZIP/OOXML أو يحتوي VBA أو علاقات خارجية/مكونات غير مدعومة. يرفض تحليل الورقة عند الرؤوس الخاطئة أو الصيغ أو الدمج المؤثر؛ أما عيوب حقول الصف فتظهر في `ErrorCodes` دون إلغاء بقية الصفوف.

## 6) قواعد المطابقة

التصنيف **حصري وبالأولوية**: `Invalid` ثم `NotFound` ثم `DepartmentMismatch` ثم `Matched`. المطابقة بالـ ReferenceCode والقسم فقط، لا بالأسماء، ولا تستخدم الدرجة في أي حكم على النجاح أو الرسوب. يتحقق دائمًا:

`TotalRows = InvalidCount + NotFoundCount + DepartmentMismatchCount + MatchedCount`.

يتم الاستعلام عن الرموز الصحيحة المميزة في استعلام تجميعي، وليس تحميل جدول المرجع لكل صف. التخزين لا يحتفظ إلا بنصوص مقيدة بالطول، والدرجات من النوع `decimal`، والملف الأصلي يُحذف بعد نجاح التحليل.

## 7) الأمن ودورة حياة الملف

- تخزين تحت `App_Data/StagedWorkbooks` خارج `wwwroot`، ومعرف عشوائي داخلي دون نشره في الصفحة أو URL. افحص ACL على Windows، واقصر أذونات هذا المجلد على حساب تشغيل التطبيق. لا تضف ملفات `App_Data` إلى Git.
- ترفض الملفات الضخمة أولًا، وتُراجع بنية ZIP وأحجام مكوناته وروابط OOXML قبل القراءة عبر ClosedXML؛ لا تنفيذ للصيغ أو الماكرو.
- `POST` محمي بـ antiforgery الخاص بـ Razor Pages؛ الاستيراد لا يعدّل المرجع، والواجهة بلا حذف جماعي.
- تخزن حالة الدفعة والعدادات والصفوف في **Transaction واحدة**، وتُرفض إعادة التحليل بعد `Completed`.
- عند أول طلب لاحق، يفحص النموذج دفعات `Uploaded` التي تجاوزت ساعة أو فقدت ملفها ويحولها إلى `Expired` ويحذف المؤقت.
- مزامنة `SemaphoreSlim` وحيدة داخل العملية بين التحليل والـ cleanup وseed والرفع. **لا يوجد دعم Multi-instance أو distributed locking.**
- رسالة الخطأ العامة لا تتضمن Stack Trace؛ يُعرض Trace ID. تُكتب بيانات الحدث والفئة والمعرّف ونوع الاستثناء فقط في `App_Data/logs/events.log` بلا قيم صفوف ولا رسائل استثناء؛ لا تتضمن سجلات واجهة الاستخدام تفاصيل الطلبات.
- نصوص التصدير تكتب كـ **نص وليس كصيغ**، وتسبق المدخلات الخطرة بحرف zero-width لتعطيل Excel formula injection ظاهريًا مع الحفاظ على القراءة البصرية.

## 8) اختبارات القبول

| المطلب | الاختبار أو وسيلة التحقق |
|---|---|
| البداية بصفر، بلا مرجع تلقائي | `HttpSecurityTests.FreshHomeShowsZeros_...` + `ImportWorkflowTests.SqliteMigration_...` |
| seed مرتان، لا تكرار | `SqliteMigration_AndServiceEndToEnd_...` |
| لا نتائج قبل التحليل | `SqliteMigration_AndServiceEndToEnd_...` |
| الورقة المخفية / اسم ملفق | `DemoTemplate_IsXlsxWithVisibleAndHiddenSheets_...` و`SqliteMigration_...` |
| ZIP تالف / حد الحجم | `InvalidFile_AndOversizedArchive_...` |
| رؤوس خاطئة / صيغة | `InvalidHeaderAndFormula_...` |
| ملف علاقات خارجية / ماكرو | `ExternalRelationshipAndMacroArchive_AreRejected` |
| فشل الإدراج وrollback | `FailedSqliteInsert_RollsBackRowsAndCompletion` |
| نص يبدأ بعلامة صيغة في التصدير | `ExportDangerousText_StoresTextRatherThanFormula` |
| تكرارات بعد Trim، كود رقمي، lowercase | `WorkbookRulesTests` |
| Score على الحدود وخارجها وأكثر من منزلتين | `WorkbookRulesTests` |
| الحالات الست ومعادلة الأعداد | `KnownSixRows_RespectPriorityAndCountInvariant` |
| تغيير الاسم لا يغير المطابقة | `DifferentDisplayNameDoesNotBlockMatched` |
| إعادة تحليل Completed مرفوضة | `SqliteMigration_AndServiceEndToEnd_...` |
| استمرار SQLite بعد إعادة فتح Context | `SqliteMigration_AndServiceEndToEnd_...` |
| الفلتر وتقرير كامل بلا صيغ | `SqliteMigration_AndServiceEndToEnd_...` |
| انتهاء المؤقت | `MissingTempFile_BecomesExpiredOnCleanup` |
| غياب antiforgery / معرف غير موجود | `HttpSecurityTests` |

**مهم:** بعض معايير القبول المذكورة في المواصفات لا يغطيها اختبار آلي منفصل بعد، ومنها: اختبار تزامن عدة طلبات مع cleanup، اختبارات كل أنواع الخلايا Excel وتفاوت النطاقات، وفحص بصري حقيقي 360px ولوحة المفاتيح. هذه عناصر يجب اختبارها قبل اعتبار العمل مكتملًا، ولا يُفهم من وجود الكود أنها اجتازت الاختبارات.

## 9) حدود النسخة والتسليم

هذه **نسخة مصدر تعليمية** وليست إثباتًا لتسليم منتج عميل أو نظام إنتاج. لا تحتوي بيانات عميل أو bin/obj أو أسرار أو SQLite جاهزة، ولا تتعامل مع ملفات XLS/XLSM/CSV أو مستندات محمية أو ملفات كبيرة أو أجهزة أو شبكة أو صلاحيات أو مستخدمين متزامنين. راجع `TEST_RESULTS.md` لنتائج التحقق الفعلي بتاريخ 23 نوفمبر 2026.
