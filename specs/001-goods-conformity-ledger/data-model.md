# مدل داده: دفتر کل انطباق کالاهای وارداتی

دو لایه:

- **درون‌زنجیره** (world state (وضعیت جهانی) در Fabric + تاریخچهٔ تغییرناپذیر): شناسه‌ها، هش‌ها، رأی‌ها، تصمیم‌ها.
  بدون داده شخصی (منشور، اصل III، FR-006).
- **برون‌زنجیره** (PostgreSQL): داده شخصی، بایت‌های اسناد، مواد کلید کاربران، مدل خواندنی
  برای گزارش‌ها. مدل خواندنی قابل بازسازی است؛ منبع حقیقت، دفتر کل است.

قراردادها: شناسه‌ها ULIDهایی به‌صورت رشته هستند. مهرهای زمانی، مهر زمانی تراکنش هستند
(`stub.GetTxTimestamp`، UTC، RFC 3339) — هرگز ساعت سیستم درون chaincode استفاده نمی‌شود. مقادیر اعشاری
رشته هستند (مثلاً `"0.75"`). هر نوشتن درون‌زنجیره، رویدادی هم‌نام با
تابع نیز منتشر می‌کند.

## موجودیت‌های درون‌زنجیره

کلیدهای world state از کلیدهای مرکب `<type>~<id>` استفاده می‌کنند. تاریخچهٔ هر کلید از
دفتر کل در دسترس است (`GetHistoryForKey`) که ردّ کامل و فقط‌افزودنی را ارائه می‌دهد.

### Participant

| فیلد | نوع | قواعد |
|-------|------|-------|
| participantId | string | یکتا؛ با ویژگی گواهی `participantId` منطبق است |
| orgMsp | string | `StdOrgMSP` یا MSP یک آزمایشگاه |
| roles | []enum | `inspector`، `lab`، `assessor`، `techOfficer`، `importer`، `auditor`، `admin`، `service` (هویت gateway سازمان StdOrg برای ارزیابی خودکار و اعتراض‌های واردکنندگان) |
| status | enum | `active` / `revoked` |
| revokedAt | timestamp? | هنگام ابطال تنظیم می‌شود؛ اقدامات پس از آن رد می‌شوند (FR-020) |

واردکنندگان مشارکت‌کنندگانی بدون گواهی هستند؛ `participantId` به رکورد برون‌زنجیرهٔ
واردکننده پیوند دارد.

### Consignment

| فیلد | نوع | قواعد |
|-------|------|-------|
| consignmentId | string | یکتا (FR-002) |
| parentId | string? | زمانی تنظیم می‌شود که این محموله بخشی تفکیک‌شده از محمولهٔ دیگری باشد |
| importerId | string | مشارکت‌کننده با نقش `importer` |
| productCategory | string | کدی از فهرست دسته‌بندی‌های سازمان |
| productDescription | string | حداکثر 500 نویسه؛ بدون داده شخصی |
| standardRef | object | `{number, edition}`؛ الزامی |
| quantity | decimal string + unit | ‎> 0 |
| origin | string | ISO 3166-1 alpha-2 |
| arrivalDate | date | ≤ زمان ثبت |
| registeredBy | participantId | نقش `inspector` |
| documents | []DocumentRef | به ادامه مراجعه شود |
| status | enum | به ماشین حالت مراجعه شود |
| currentVerdictId | string? | آخرین رأی |
| currentDecisionId | string? | آخرین تصمیم |

### DocumentRef (تعبیه‌شده، درون‌زنجیره)

| فیلد | نوع | قواعد |
|-------|------|-------|
| documentId | string | با رکورد برون‌زنجیره منطبق است |
| kind | enum | `invoice`، `packingList`، `photo`، `testReport`، `dispositionEvidence`، `other` |
| sha256 | hex string | 64 نویسه (FR-005) |
| removed | bool | پس از حذف قانونی true می‌شود؛ هرگز به false بازنمی‌گردد |

### Correction (فقط رویداد)

اصلاحات تاریخچه را بازنویسی نمی‌کنند: فیلد اصلاح‌شده در world state به‌روزرسانی می‌شود و
تراکنش `{targetId, field, oldValue, newValue, reason, by}` را ذخیره می‌کند؛ هر دو مقدار در
تاریخچهٔ کلید باقی می‌مانند (FR-004). `reason` الزامی است، حداکثر 500 نویسه.

### Sample

| فیلد | نوع | قواعد |
|-------|------|-------|
| sampleId | string | یکتا |
| consignmentId | string | محمولهٔ موجود |
| takenBy | participantId | نقش `inspector` |
| takenAt | date | ≥ arrivalDate |
| assignedLabMsp | string | آزمایشگاهی که آن را آزمون خواهد کرد |

### TestResult

| فیلد | نوع | قواعد |
|-------|------|-------|
| resultId | string | یکتا |
| sampleId | string | نمونهٔ موجود |
| testCode | string | باید در مجموعه قواعد قابل‌اعمال وجود داشته باشد |
| value | decimal string | برای آزمون‌های عددی الزامی است |
| passFail | enum? | برای آزمون‌های کیفی (مثلاً بررسی برچسب‌گذاری): `pass`/`fail` |
| unit | string | باید واحد قاعده باشد یا با جدول ضریب مجموعه قواعد قابل تبدیل باشد |
| method | string | ارجاع به روش آزمون |
| reportDocumentId | string | DocumentRef از نوع `testReport` |
| submittedBy | participantId | نقش `lab`، از آزمایشگاه تخصیص‌یافته به نمونه |
| batchId | string | نتایجی را که با هم از یک فایل بارگذاری شده‌اند گروه‌بندی می‌کند |

### RuleSet

| فیلد | نوع | قواعد |
|-------|------|-------|
| ruleSetId | string | برای هر ویرایش استاندارد ثابت است |
| version | int | از 1 شروع می‌شود، به ازای هر نسخهٔ منتشرشده +1 |
| standardRef | object | `{number, edition}` |
| productCategories | []string | دسته‌هایی که این مجموعه قواعد بر آن‌ها اعمال می‌شود |
| tests | []RuleTest | حداقل یکی |
| unitFactors | map unit→{baseUnit, factor} | ضرایب اعشاری؛ برای قطعیت (determinism) به‌صورت فهرست مرتب ذخیره می‌شود |
| effectiveFrom | date | باید ≥ تاریخ تأیید باشد |
| draftedBy | participantId | نقش `techOfficer` |
| approvedBy | participantId? | نقش `techOfficer`، ‏≠ draftedBy (FR-019) |
| status | enum | `draft` → `approved` (پس از تأیید تغییرناپذیر) |

**RuleTest**: `testCode`، `name`، `clause` (بند استاندارد، الزامی)، `kind`
(`numeric` / `qualitative`)، `unit`، `min?`، `max?`، `minInclusive`، `maxInclusive`،
`required` (bool)، `samplesRequired` (int ≥ 1).

### Verdict

| فیلد | نوع | قواعد |
|-------|------|-------|
| verdictId | string | یکتا |
| consignmentId | string | |
| ruleSetId + version | string + int | نسخهٔ معتبر در arrivalDate (FR-009/010) |
| inputsHash | sha256 | هش شناسه‌ها و مقادیر مرتب‌شدهٔ نتایج ورودی |
| perTest | []{testCode, outcome, measured, limit, reason} | outcome: `pass`/`fail`/`missing`/`conflict`/`unitError` |
| overall | enum | `Conforming` / `NonConforming` / `RequiresReview` |
| manual | bool | برای مسیر دستی true است (FR-017) |

قاعدهٔ کلی: هر `fail` ← `NonConforming`؛ در غیر این صورت هر `missing`/`conflict`/`unitError` یا
نبود مجموعه قواعد فعال ← `RequiresReview`؛ در غیر این صورت `Conforming`. یک `fail` تنها زمانی محسوب می‌شود که هیچ
نتیجهٔ دیگری برای همان آزمون و نمونه با آن در تعارض نباشد؛ نتایج متعارض ← `conflict`.

### Decision

| فیلد | نوع | قواعد |
|-------|------|-------|
| decisionId | string | |
| consignmentId | string | |
| verdictId | string | رأیی که تأیید/بازنگری رأی می‌شود |
| kind | enum | `confirm` / `override` / `appealOutcome` / `manual` |
| outcome | enum | `Conforming` / `NonConforming` |
| justification | string | برای `override`، `appealOutcome`، `manual` الزامی است؛ حداکثر 2000 نویسه |
| certificateRef | string? | هنگامی که outcome برابر `Conforming` باشد الزامی است |
| rejectionReasons | []string? | هنگامی که outcome برابر `NonConforming` باشد الزامی است |
| decidedBy | participantId | نقش `assessor`؛ **نباید** همان MSP/فردی باشد که نتایج را ثبت کرده است (FR-019) |
| previousDecisionId | string? | برای نتایج اعتراض تنظیم می‌شود |

### Appeal

`appealId`، `consignmentId`، `decisionId`، `filedBy` (مشارکت‌کنندهٔ واردکننده)، `grounds`
(حداکثر 2000 نویسه)، `status` (`open` / `resolved`)، `resolutionDecisionId?`.

### Disposition (Clarification Q5)

| فیلد | نوع | قواعد |
|-------|------|-------|
| dispositionId | string | |
| consignmentId | string | تصمیم نهایی NonConforming |
| type | enum | `reExport` / `destruction` / `rework` |
| chosenBy | participantId | نقش `inspector` یا `assessor` |
| completedBy | participantId? | نقش `inspector` |
| completedAt | date? | |
| evidenceDocumentIds | []string | ‏≥ 1 هنگام تکمیل |
| reassessmentConsignmentLink | string? | برای rework: چرخهٔ رأی جدید از همان consignmentId استفاده می‌کند |

## ماشین حالت محموله

```text
Registered ──sample──▶ Sampling ──results──▶ ResultsReceived ──assess──▶ Assessed
Assessed ──confirm/override──▶ Conforming (final)
Assessed ──confirm/override──▶ NonConforming ──choose──▶ DispositionPending
DispositionPending ──complete(reExport|destruction)──▶ Closed
DispositionPending ──complete(rework)──▶ Sampling   (new samples, new verdict)
Conforming | NonConforming ──appeal──▶ UnderAppeal ──appealOutcome──▶ Conforming | NonConforming
Any non-final ──split──▶ Split (children get their own lifecycle)
```

گذارهایی که فهرست نشده‌اند توسط chaincode رد می‌شوند. رأی‌های `manual` در حالت `Assessed` وارد می‌شوند.

## موجودیت‌های برون‌زنجیره (PostgreSQL)

| جدول | فیلدهای کلیدی | یادداشت |
|-------|-----------|-------|
| importer | id, legal_name, national_id, contact_email, phone, locale | داده شخصی (هرگز درون‌زنجیره نیست) |
| app_user | id, participant_id, org, oidc_subject, locale, calendar (`jalali`/`gregorian`) | |
| user_key | user_id, encrypted_private_key, cert_pem | با کلید HSM/فایل پوشانده می‌شود |
| document | id, consignment_id, kind, filename, mime, size (≤ 25 MB), sha256, bytes, removed_at | بایت‌ها هنگام حذف قانونی null می‌شوند |
| results_upload | id, lab_msp, filename, sha256, status, errors_json | خطاهای هر ردیف هنگام رد |
| notification | id, recipient_user_id, kind, payload_json, sent_at, read_at | |
| rm_consignment, rm_verdict, rm_decision, rm_disposition | بازتاب فیلدهای زنجیره + شمارهٔ بلوک | مدل خواندنی، بازسازی‌شده از دفتر کل |
| projection_checkpoint | channel, last_block | نقطهٔ ازسرگیری برای شنوندهٔ رویداد |
