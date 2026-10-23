# قرارداد Chaincode: `conformity`

کانال: `conformity`. همهٔ توابع `participantId` و `role` فراخواننده را از
ویژگی‌های گواهی (`cid`) می‌خوانند، شرکت‌کنندگان ابطال‌شده را رد می‌کنند و فراخوانی‌هایی را که نقش آن‌ها
فهرست نشده است رد می‌کنند. خطاها از کدهای پایدار استفاده می‌کنند: `ERR_FORBIDDEN`، `ERR_NOT_FOUND`، `ERR_INVALID`،
`ERR_STATE` (گذار غیرمجاز)، `ERR_SOD` (تفکیک وظایف)، `ERR_HASH_MISMATCH`.

ورودی‌ها و خروجی‌ها JSON منطبق با [data-model.md](../data-model.md) هستند.

## توابع نوشتن

| تابع | نقش‌های مجاز | ورودی | اثر / بررسی‌های کلیدی |
|----------|----------------|-------|---------------------|
| `RegisterParticipant` | admin | Participant | ایجاد؛ فقط admin همان MSP |
| `RevokeParticipant` | admin | participantId، reason | status ← revoked |
| `RegisterConsignment` | inspector | Consignment (بدون status) + DocumentRefs | status ← Registered؛ همهٔ فیلدهای الزامی موجود باشند |
| `AttachDocument` | inspector، lab | consignmentId، DocumentRef | فقط افزودنی (append only) |
| `RecordDocumentRemoval` | admin | consignmentId، documentId، legalBasis | `removed=true` |
| `CorrectConsignment` | inspector | consignmentId، field، newValue، reason | رویداد اصلاحیه؛ فیلدهای هویتی (`consignmentId`، `registeredBy`) قابل اصلاح نیستند |
| `SplitConsignment` | inspector | consignmentId، lots[] | محموله‌های فرزند با `parentId`؛ والد ← Split |
| `RecordSample` | inspector | Sample | status از Registered ← Sampling |
| `SubmitResults` | lab | batchId، TestResult[] | همه یا هیچ؛ MSP فراخواننده = assignedLabMsp نمونه؛ status ← ResultsReceived |
| `AssessConsignment` | assessor، service | consignmentId | مجموعه قواعد را بر اساس arrivalDate انتخاب می‌کند، Verdict را می‌نویسد؛ status ← Assessed |
| `RecordManualVerdict` | assessor | consignmentId، perTest[]، justification | Verdict با `manual=true` |
| `DecideConsignment` | assessor | Decision + `documentChecks[] {documentId, sha256}` | اگر فراخواننده هر نتیجه‌ای برای آن ارسال کرده باشد ← `ERR_SOD`؛ بازنگری رأی نیازمند توجیه است؛ هر هش در `documentChecks` که با هش روی زنجیره متفاوت باشد ← `ERR_HASH_MISMATCH`؛ نبود بررسی برای یک سند حذف‌نشده ← `ERR_INVALID` |
| `FileAppeal` | service (از طرف واردکنندهٔ ثبت‌شده) | Appeal | فقط واردکنندهٔ ثبت‌شده؛ status ← UnderAppeal |
| `ResolveAppeal` | assessor (≠ تصمیم‌گیرندهٔ اولیه) | appealId، Decision(kind=appealOutcome) | اعتراض رسیدگی‌شده |
| `ChooseDisposition` | inspector، assessor | Disposition (type) | نیازمند رأی نهایی NonConforming؛ status ← DispositionPending |
| `CompleteDisposition` | inspector | dispositionId، evidenceDocumentIds | Closed، یا Sampling برای rework |
| `DraftRuleSet` | techOfficer | RuleSet (status draft) | شمارهٔ نسخهٔ جدید |
| `ApproveRuleSet` | techOfficer | ruleSetId، version | اگر فراخواننده پیش‌نویس آن را تهیه کرده باشد ← `ERR_SOD`؛ پس از آن تغییرناپذیر |

## توابع خواندن

| تابع | نقش‌های مجاز | خروجی |
|----------|----------------|--------|
| `GetConsignment` | همه به‌جز importer (importer از طریق فیلتر gateway) | Consignment |
| `GetHistory` | inspector، assessor، auditor، admin | ورودی‌های مرتب: txId، timestamp، function، actor، payload |
| `GetRuleSet` | همه | RuleSet (نسخهٔ مشخص یا نسخهٔ نافذ در یک تاریخ) |
| `ReassessDryRun` | auditor، assessor | Verdict بازمحاسبه‌شده از ورودی‌های ثبت‌شده و نسخهٔ ثبت‌شدهٔ مجموعه قواعد؛ بدون نوشتن |
| `VerifyDocument` | auditor، assessor، inspector | `{documentId, status: ok|mismatch|removed}` با دریافت یک sha256 ارائه‌شده |

## قواعد قطعیت (determinism) (اعمال‌شده توسط آزمون‌ها و بررسی ایستا)

- بدون `float32/float64`، `time.Now`، `math/rand`، دسترسی به شبکه یا فایل.
- هنگام ساخت خروجی هرگز روی mapهای Go پیمایش (range) نکنید؛ ابتدا کلیدها را مرتب کنید.
- مُهرهای زمانی فقط از `stub.GetTxTimestamp()` به دست می‌آیند.

## Endorsement

`AND('StdOrgMSP.peer', OutOf(1, <every lab MSP>.peer))`. هنگام پیوستن یا خروج یک آزمایشگاه،
از طریق چرخهٔ حیات chaincode به‌روزرسانی می‌شود.
