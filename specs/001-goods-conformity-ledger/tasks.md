---

description: "فهرست وظایف دفتر کل انطباق کالاهای ورودی"
---

# وظایف: دفتر کل انطباق کالاهای ورودی

**ورودی**: اسناد طراحی از `/specs/001-goods-conformity-ledger/`

**پیش‌نیازها**: plan.md، spec.md، research.md، data-model.md، contracts/، quickstart.md

**آزمون‌ها**: گنجانده شده‌اند. اصل V منشور، chaincode آزمون‌محور (ابتدا آزمون) را غیرقابل‌مذاکره می‌داند و
دروازه گردش کار، آزمون‌های یکپارچگی روی شبکه‌ای چندنودی را الزامی می‌کند. آزمون‌های chaincode **باید**
پیش از وظیفه پیاده‌سازی متناظر نوشته شوند و شکست آن‌ها مشاهده شود.

**سازمان‌دهی**: وظایف بر اساس داستان کاربری گروه‌بندی شده‌اند تا هر داستان به‌تنهایی ساخته و
آزموده شود.

## قالب: `[ID] [P?] [Story] Description`

- **[P]**: قابل اجرای موازی (فایل‌های متفاوت، بدون وابستگی)
- **[Story]**: داستان کاربری‌ای که این وظیفه به آن تعلق دارد (US1–US5)

## قراردادهای مسیر

- Chaincode: `chaincode/conformity/`
- Gateway: `gateway/cmd/`، `gateway/internal/`، `gateway/migrations/`
- وب: `web/src/`، `web/tests/`
- شبکه: `network/compose/`، `network/config/`، `network/scripts/`
- آزمون‌های بین‌مؤلفه‌ای: `tests/integration/`؛ داده‌های آزمون: `testdata/`

---

## فاز 1: راه‌اندازی (زیرساخت مشترک)

**هدف**: ساختار مخزن، زنجیره‌ابزارها، شبکه محلی سه‌سازمانی

- [ ] T001 ایجاد ساختار پوشه‌ها مطابق plan.md (`chaincode/conformity/`، `gateway/cmd/gateway/`، `gateway/internal/{api,fabric,store,projector,resultsfile,notify,keys,config}/`، `gateway/migrations/`، `web/`، `network/{compose,config,scripts}/`، `tests/integration/`، `testdata/`، `tools/detcheck/`) و یک `Makefile` ریشه با هدف‌های `network-up`، `network-down`، `chaincode-deploy`، `stack-up`، `seed`، `test-chaincode`، `test-integration`، `test-web`
- [ ] T002 مقداردهی اولیه ماژول Go در `chaincode/conformity/go.mod` (Go 1.23) با `github.com/hyperledger/fabric-contract-api-go/v2`، `github.com/hyperledger/fabric-chaincode-go/v2`، `github.com/shopspring/decimal`
- [ ] T003 [P] مقداردهی اولیه ماژول Go در `gateway/go.mod` (Go 1.23) با `github.com/hyperledger/fabric-gateway`، `github.com/jackc/pgx/v5`، `github.com/coreos/go-oidc/v3`، `github.com/prometheus/client_golang`؛ ثبت رخدادها از `log/slog` کتابخانه استاندارد استفاده می‌کند
- [ ] T004 [P] ایجاد اسکلت برنامه وب در `web/` با Vite + React 18 + TypeScript 5، و افزودن `react-i18next`، `i18next`، `oidc-client-ts`، `vitest`، `@testing-library/react`، `@playwright/test`
- [ ] T005 [P] افزودن پیکربندی lint برای Go در `.golangci.yml` (govet، staticcheck، errcheck) و پیکربندی lint وب در `web/eslint.config.js`
- [ ] T006 [P] پیاده‌سازی تحلیل‌گر قطعیت در `tools/detcheck/main.go` با استفاده از `golang.org/x/tools/go/analysis` که در این موارد شکست می‌خورد: هرگونه استفاده از `float32`/`float64`، `time.Now`، `math/rand`، import کردن `net`، `net/http`، `os`، و `range` روی نوع map؛ اتصال آن به `make test-chaincode` روی `chaincode/conformity/...`
- [ ] T007 نوشتن پیکربندی کانال Fabric در `network/config/configtx.yaml` با سازمان‌های `StdOrgMSP`، `Lab1MSP`، `Lab2MSP`، سه orderer از نوع Raft متعلق به StdOrg، کانال `conformity`، و پیکربندی‌های سرور Fabric CA در `network/config/ca/{stdorg,lab1,lab2}/fabric-ca-server-config.yaml`
- [ ] T008 نوشتن `network/compose/docker-compose.fabric.yaml` با سه CA، `orderer{0,1,2}.stdorg`، `peer0.stdorg`، `peer1.stdorg`، `peer0.lab1`، `peer0.lab2` (وضعیت LevelDB، معیارهای Prometheus فعال)
- [ ] T009 نوشتن `network/scripts/network-up.sh` (ثبت‌نام مدیران سازمان‌ها، ایجاد کانال `conformity`، پیوستن همه peerها) و `network/scripts/deploy-chaincode.sh` (بسته‌بندی، نصب روی همه peerها، تأیید توسط هر سازمان، commit با سیاست تأیید `AND('StdOrgMSP.peer', OutOf(1, 'Lab1MSP.peer', 'Lab2MSP.peer'))`)
- [ ] T010 نوشتن `network/compose/docker-compose.stack.yaml` با PostgreSQL 16 برای StdOrg و Lab1، Keycloak 26 با import کردن `network/config/keycloak/stdorg-realm.json` و `network/config/keycloak/lab1-realm.json` (کاربران اولیه `inspector1`، `assessor1`، `assessor2`، `techofficer1`، `techofficer2`، `auditor1`، `admin1`، `importer1` در StdOrg، `lab1user` در Lab1؛ گذرواژه `Passw0rd!`؛ TOTP در محیط توسعه غیرفعال)، gateway سازمان StdOrg روی `:8080` و gateway سازمان Lab1 روی `:8081`
- [ ] T011 [P] افزودن گردش کار CI در `.github/workflows/ci.yml` که `make test-chaincode`، `make network-up chaincode-deploy stack-up test-integration`، `make test-web` را اجرا می‌کند؛ هر شکستی مانع ادغام می‌شود

---

## فاز 2: زیربنایی (پیش‌نیازهای مسدودکننده)

**هدف**: هسته مشترک chaincode، هویت/دسترسی، ذخیره‌سازی مجموعه قواعد، اسکلت gateway، پوسته وب

**⚠️ بحرانی**: تا تکمیل این فاز، هیچ کاری روی داستان‌های کاربری نمی‌تواند آغاز شود

### هسته chaincode

- [ ] T012 تعریف همه ساختارها و enumهای روی زنجیره از data-model.md در `chaincode/conformity/types.go` (Participant، Consignment، DocumentRef، Correction، Sample، TestResult، RuleSet، RuleTest، Verdict، Decision، Appeal، Disposition)؛ فیلدهای اعشاری از نوع `string` هستند؛ enum نقش‌ها `inspector`، `lab`، `assessor`، `techOfficer`، `importer`، `auditor`، `admin`، `service`
- [ ] T013 [P] تعریف کدهای خطای `ERR_FORBIDDEN`، `ERR_NOT_FOUND`، `ERR_INVALID`، `ERR_STATE`، `ERR_SOD`، `ERR_HASH_MISMATCH` و سازنده‌ای که `CODE: message` برمی‌گرداند در `chaincode/conformity/errors.go`
- [ ] T014 [P] ساخت stub جعلی درون‌حافظه‌ای برای آزمون‌ها در `chaincode/conformity/fakestub_test.go`: وضعیت جهانی، تاریخچه به ازای هر کلید، مهر زمانی ثابت تراکنش، شناسه تراکنش، و MSP فراخوان و ویژگی‌های گواهی `role`، `participantId` قابل پیکربندی
- [ ] T015 نوشتن توابع کمکی وضعیت در `chaincode/conformity/state.go`: کلیدهای ترکیبی `<type>~<id>`، `getJSON`/`putJSON`، `emit(eventName, payload)` که به نام تابع فراخوان نام‌گذاری می‌شود، `txTime()` فقط از `stub.GetTxTimestamp()`، و تابع کمکی `sortedKeys` (هرگز برای خروجی روی mapها range نکنید)
- [ ] T016 نوشتن آزمون‌های شکست‌خورده در `chaincode/conformity/access_test.go`: شرکت‌کننده ناشناخته → `ERR_FORBIDDEN`؛ شرکت‌کننده ابطال‌شده (اقدام پس از `revokedAt`) → `ERR_FORBIDDEN`؛ نقش نادرست → `ERR_FORBIDDEN`؛ نقش درست پذیرفته می‌شود
- [ ] T017 پیاده‌سازی هویت فراخوان و `requireRole(roles...)` در `chaincode/conformity/access.go` با خواندن ویژگی‌های گواهی `role` و `participantId` از طریق `cid`، و بررسی اینکه `status` شرکت‌کننده `active` باشد
- [ ] T018 نوشتن آزمون‌های شکست‌خورده در `chaincode/conformity/participant_test.go`: `RegisterParticipant` فقط توسط `admin` همان `orgMsp`؛ `RevokeParticipant` مقدار `status` را `revoked` و `revokedAt` را تنظیم می‌کند؛ مدخل‌های گذشته امضاشده توسط شرکت‌کننده ابطال‌شده قابل خواندن باقی می‌مانند (FR-020)
- [ ] T019 پیاده‌سازی `RegisterParticipant` و `RevokeParticipant` در `chaincode/conformity/participant.go`، به‌علاوه یک `InitLedger` یک‌باره که نخستین مدیر StdOrg را از هویت مستقرکننده ثبت می‌کند
- [ ] T020 نوشتن آزمون‌های شکست‌خورده در `chaincode/conformity/ruleset_test.go`: `DraftRuleSet` توسط `techOfficer` نسخه‌ای با `version` = قبلی + 1 و `status` برابر `draft` ایجاد می‌کند؛ هر RuleTest به `clause` نیاز دارد؛ `ApproveRuleSet` توسط تهیه‌کننده پیش‌نویس → `ERR_SOD`؛ مجموعه قواعد تأییدشده تغییرناپذیر است؛ `effectiveFrom` **باید** ≥ تاریخ تأیید باشد؛ `GetRuleSet(id, date)` بالاترین نسخه تأییدشده با `effectiveFrom` ≤ تاریخ را برمی‌گرداند؛ `unitFactors` به‌صورت فهرستی مرتب‌شده بر اساس واحد ذخیره می‌شود
- [ ] T021 پیاده‌سازی `DraftRuleSet`، `ApproveRuleSet`، `GetRuleSet` (بر اساس نسخه یا نسخه نافذ در یک تاریخ) در `chaincode/conformity/ruleset.go`
- [ ] T022 ثبت قرارداد و مسیریابی همه توابع در `chaincode/conformity/contract.go` و `chaincode/conformity/main.go` (نقطه ورود chaincode-as-a-service)
- [ ] T023 نوشتن `network/scripts/seed.sh` که کاربران اولیه را با ویژگی‌های Fabric CA ثبت‌نام می‌کند، شرکت‌کنندگان را ثبت می‌کند، و `testdata/rulesets/iec60335-1-ed6-v1.json` را از طریق `peer chaincode invoke` پیش‌نویس (`techofficer1`) و تأیید (`techofficer2`) می‌کند؛ ایجاد آن داده آزمون مجموعه قواعد با دست‌کم یک آزمون عددی با حد شامل (`IR_MIN`، `MOhm`، حداقل `"2"`)، یکی با حداکثر غیرشامل و ضریب واحد (`LEAK_MAX`، `mA`، `A -> mA = 1000`)، و یک آزمون کیفی (`LABEL`)

### هسته gateway

- [ ] T024 پیاده‌سازی بارگذاری پیکربندی از متغیرهای محیطی در `gateway/internal/config/config.go`: MSP سازمان، فهرست مرتب endpointهای peer (ابتدا peer خودی، peer سازمان StdOrg به‌عنوان جایگزین برای gatewayهای آزمایشگاه)، URL پایگاه داده، صادرکننده/کلاینت OIDC، رله SMTP، مسیر فایل کلید پوشش‌دهنده کلید، نشانی شنود
- [ ] T025 نوشتن طرح‌واره برون‌زنجیره‌ای `gateway/migrations/001_init.sql` برای جدول‌های `importer`، `app_user` (`calendar` در `jalali`/`gregorian`، `locale` در `fa`/`en`)، `user_key`، `document` (CHECK `size <= 26214400`)، `results_upload`، `notification`، `rm_consignment`، `rm_verdict`، `rm_decision`، `rm_disposition`، `projection_checkpoint`؛ اعمال مهاجرت‌های تعبیه‌شده به‌ترتیب هنگام راه‌اندازی از `gateway/internal/store/migrate.go`
- [ ] T026 [P] پیاده‌سازی pool مربوط به pgx و پرس‌وجوهای کاربر/کلید در `gateway/internal/store/store.go` و `gateway/internal/store/users.go`
- [ ] T027 [P] پیاده‌سازی پوشش/بازگشایی AES-256-GCM کلیدهای خصوصی کاربران با فایل کلید پوشش‌دهنده در `gateway/internal/keys/keys.go` همراه با آزمون رفت‌وبرگشت `gateway/internal/keys/keys_test.go` (`// ponytail: file wrapping key, move to HSM for production`)
- [ ] T028 پیاده‌سازی کلاینت Fabric در `gateway/internal/fabric/client.go`: امضاکننده به ازای هر درخواست ساخته‌شده از کلید بازگشایی‌شده کاربر احرازهویت‌شده، `Submit`/`Evaluate`، جابه‌جایی خودکار در فهرست peerهای پیکربندی‌شده، استخراج کد `ERR_*` از خطاهای chaincode، بازگرداندن `txId`
- [ ] T029 [P] پیاده‌سازی ثبت‌نام کاربر با Fabric CA (register + enroll با ویژگی‌های `role`، `participantId`، ذخیره کلید پوشش‌داده‌شده) در `gateway/internal/fabric/enroll.go`
- [ ] T030 پیاده‌سازی میان‌افزار bearer مربوط به OIDC در `gateway/internal/api/auth.go` (تأییدکننده go-oidc، نگاشت `sub` به `app_user`، نقش‌ها از توکن، الزام ادعای عامل دوم `acr` برای همه نقش‌ها به‌جز `importer`)
- [ ] T031 [P] پیاده‌سازی نگاشت خطا در `gateway/internal/api/errors.go` (`ERR_INVALID` 422، `ERR_FORBIDDEN`/`ERR_SOD` 403، `ERR_NOT_FOUND` 404، `ERR_STATE` 409، `ERR_HASH_MISMATCH` 409؛ بدنه `{code, message, details}`) و فهرست پیام‌های fa/en در `gateway/internal/api/messages.go` که بر اساس `Accept-Language` انتخاب می‌شود
- [ ] T032 پیاده‌سازی مسیریاب و سرور در `gateway/internal/api/router.go` و `gateway/cmd/gateway/main.go` با `/healthz`، `/metrics`، ثبت‌کننده JSON مبتنی بر slog که فیلدهای `txId` و `consignmentId` را می‌افزاید، و ارائه SPA ساخته‌شده از `web/dist`
- [ ] T033 پیاده‌سازی اسکلت projector رویدادهای بلوک در `gateway/internal/projector/projector.go`: ازسرگیری از `projection_checkpoint`، توزیع رویدادها بر اساس نام به handlerهای ثبت‌شده، به‌روزرسانی checkpoint در همان تراکنش پایگاه داده
- [ ] T034 پیاده‌سازی endpointهای شرکت‌کنندگان `POST /api/v1/participants` و `POST /api/v1/participants/{id}/revoke` (مدیر) در `gateway/internal/api/participants.go`

### پوسته وب

- [ ] T035 [P] راه‌اندازی i18n در `web/src/i18n/index.ts` با `web/src/i18n/fa.json` (پیش‌فرض) و `web/src/i18n/en.json`؛ تنظیم `<html lang>` و `<html dir="rtl|ltr">` هنگام تغییر زبان؛ استفاده انحصاری از ویژگی‌های منطقی CSS در `web/src/styles.css`
- [ ] T036 [P] پیاده‌سازی `formatDate`/`formatDateTime` در `web/src/lib/date.ts` با استفاده از `Intl.DateTimeFormat('fa-IR-u-ca-persian')` یا `'en-GB'` بر اساس تقویم کاربر، ورودی همواره رشته‌های ISO با زمان UTC؛ آزمون در `web/tests/date.test.ts`
- [ ] T037 پیاده‌سازی ورود OIDC با `oidc-client-ts` در `web/src/auth.ts` و پوشش API در `web/src/lib/api.ts` (توکن bearer، `Accept-Language`، کد خطا → پیام ترجمه‌شده)
- [ ] T038 پیاده‌سازی پوسته برنامه با مسیرهای مبتنی بر نقش و تنظیمات کاربر (زبان، تقویم) در `web/src/App.tsx` و `web/src/pages/Settings.tsx`

### زیرساخت آزمون یکپارچگی

- [ ] T039 پیاده‌سازی توابع کمکی یکپارچگی در `tests/integration/harness_test.go`: توکن‌ها از طریق `network/scripts/token.sh <user>`، توابع کمکی HTTP مبتنی بر JSON برای `:8080`/`:8081`، `dockerStop`/`dockerStart`، دسترسی مستقیم به پایگاه داده برای آزمون‌های دست‌کاری

**نقطه بازبینی**: شبکه بالا آمده، chaincode با شرکت‌کنندگان و مجموعه قواعد آزمایشی مستقر شده، gateway کاربران را احرازهویت می‌کند و به ازای هر شخص امضا می‌کند، پوسته وب وارد می‌شود

---

## فاز 3: داستان کاربری 1 - ثبت محموله ورودی به‌عنوان رکوردی ضددست‌کاری (اولویت: P1) 🎯 MVP

**هدف**: بازرسان محموله‌ها را همراه با اسناد ثبت می‌کنند؛ تاریخچه فقط‌افزودنی است؛ اسناد تغییریافته شناسایی می‌شوند

**آزمون مستقل**: quickstart.md سناریوی 1 — ثبت، اصلاح، دست‌کاری بایت‌های فاکتور، نمایش `mismatch` در راستی‌آزمایی، نبود `standardRef` → 422

### آزمون‌های داستان کاربری 1 ⚠️ (ابتدا نوشته شوند، **باید** شکست بخورند)

- [ ] T040 [P] [US1] آزمون‌ها در `chaincode/conformity/consignment_test.go`: `RegisterConsignment` فقط توسط `inspector`؛ `importerId`، `productCategory`، `standardRef` `{number, edition}` الزامی؛ `productDescription` «حداکثر 500 نویسه»؛ `quantity` «> 0»؛ `origin` «ISO 3166-1 alpha-2»؛ `arrivalDate` «≤ زمان ثبت»؛ وضعیت به `Registered` تبدیل می‌شود؛ شناسه تکراری → `ERR_INVALID`
- [ ] T041 [P] [US1] آزمون‌ها در `chaincode/conformity/correction_test.go`: `CorrectConsignment` به `reason` نیاز دارد («الزامی، حداکثر 500 نویسه»)؛ `consignmentId` و `registeredBy` قابل اصلاح نیستند؛ تاریخچه کلید مقادیر قدیم و جدید را نگه می‌دارد؛ رویداد حاوی `{targetId, field, oldValue, newValue, reason, by}` است
- [ ] T042 [P] [US1] آزمون‌ها در `chaincode/conformity/documents_test.go`: `AttachDocument` فقط‌افزودنی توسط `inspector`/`lab`؛ `sha256` هگزادسیمال «64 نویسه»؛ `kind` در `invoice`، `packingList`، `photo`، `testReport`، `dispositionEvidence`، `other`؛ `RecordDocumentRemoval` فقط برای مدیر، `removed` «هرگز دوباره به false برنمی‌گردد»؛ `VerifyDocument` مقدار `ok`/`mismatch`/`removed` را برمی‌گرداند
- [ ] T043 [P] [US1] آزمون‌ها در `chaincode/conformity/split_test.go`: `SplitConsignment` محموله‌های فرزند با `parentId` ایجاد می‌کند، والد → `Split`؛ در وضعیت‌های نهایی با `ERR_STATE` رد می‌شود
- [ ] T044 [P] [US1] آزمون‌ها در `chaincode/conformity/history_test.go`: `GetHistory` مدخل‌ها را مرتب‌شده بر اساس مهر زمانی همراه با `txId`، `timestamp`، `function`، `actor`، `payload` برمی‌گرداند؛ برای `importer` و `lab` رد می‌شود
- [ ] T045 [P] [US1] آزمون یکپارچگی برای سناریوی 1 در quickstart در `tests/integration/us1_registration_test.go`

### پیاده‌سازی داستان کاربری 1

- [ ] T046 [US1] پیاده‌سازی جدول ماشین حالت محموله (همه گذارهای بخش «Consignment state machine» در data-model.md؛ هر گذار دیگر → `ERR_STATE`) در `chaincode/conformity/statemachine.go`
- [ ] T047 [US1] پیاده‌سازی `RegisterConsignment`، `CorrectConsignment`، `SplitConsignment`، `GetConsignment` در `chaincode/conformity/consignment.go`
- [ ] T048 [P] [US1] پیاده‌سازی `AttachDocument`، `RecordDocumentRemoval`، `VerifyDocument` در `chaincode/conformity/documents.go`
- [ ] T049 [P] [US1] پیاده‌سازی `GetHistory` (از طریق `GetHistoryForKey` روی محموله و کلیدهای فرزند آن) در `chaincode/conformity/history.go`
- [ ] T050 [P] [US1] پیاده‌سازی مخزن داده‌های شخصی واردکننده (هرگز به زنجیره ارسال نمی‌شود) در `gateway/internal/store/importers.go`
- [ ] T051 [P] [US1] پیاده‌سازی مخزن اسناد در `gateway/internal/store/documents.go`: ذخیره بایت‌ها ≤ 25 MB، محاسبه SHA-256، تهی‌کردن بایت‌ها و تنظیم `removed_at` هنگام حذف قانونی
- [ ] T052 [US1] پیاده‌سازی endpointهای `POST /consignments`، `GET /consignments/{id}`، `GET /consignments/{id}/history`، `POST /consignments/{id}/corrections`، `POST /consignments/{id}/split`، `POST|GET|DELETE /consignments/{id}/documents[/{docId}]`، `GET /consignments/{id}/documents/{docId}/verify` در `gateway/internal/api/consignments.go`؛ واردکنندگان فقط محموله‌های خود را می‌بینند (FR-021)
- [ ] T053 [US1] افزودن handlerهای projector برای `RegisterConsignment`، `CorrectConsignment`، `SplitConsignment`، `AttachDocument`، `RecordDocumentRemoval` که در `rm_consignment` می‌نویسند در `gateway/internal/projector/consignments.go`
- [ ] T054 [US1] پیاده‌سازی `GET /consignments` با فیلترهای `status`، `category`، `standard`، `importer`، `from`، `to` از مدل خواندنی در `gateway/internal/api/consignments_list.go`
- [ ] T055 [P] [US1] ساخت صفحات بازرس `web/src/pages/inspector/RegisterConsignment.tsx` (فرم + بارگذاری سند، خطاهای فیلد به fa/en) و `web/src/pages/ConsignmentList.tsx`
- [ ] T056 [US1] ساخت `web/src/pages/ConsignmentDetail.tsx` با خط زمانی تاریخچه (کنشگر، زمان جلالی/میلادی)، پنجره گفت‌وگوی اصلاح، و وضعیت راستی‌آزمایی هر سند
- [ ] T057 [P] [US1] آزمون مؤلفه برای پیام‌های اعتبارسنجی ثبت در `web/tests/RegisterConsignment.test.tsx`

**نقطه بازبینی**: US1 به‌تنهایی کار می‌کند — محموله‌ها ثبت و اصلاح می‌شوند، دست‌کاری شناسایی می‌شود

---

## فاز 4: داستان کاربری 2 - ارزیابی خودکار انطباق از نتایج آزمون (اولویت: P2)

**هدف**: آزمایشگاه‌ها نتایج را (از طریق فرم یا CSV) ارسال می‌کنند؛ chaincode رأی‌های قطعی را با استناد به نسخه مجموعه قواعد محاسبه می‌کند

**آزمون مستقل**: quickstart.md سناریوی 2 — قبول، مرزی، رد، ناقص، CSV با سطر نادرست، ارزیابی مجدد یکسان

### آزمون‌های داستان کاربری 2 ⚠️ (ابتدا نوشته شوند، **باید** شکست بخورند)

- [ ] T058 [P] [US2] آزمون‌های جدول‌محور در `chaincode/conformity/assess_test.go`: همه در محدوده حدود → `Conforming`؛ مقدار برابر با حد شامل → قبول، برابر با حد غیرشامل → رد؛ هر ردی → `NonConforming` همراه با `measured` و `limit`؛ نبود آزمون الزامی → `missing` + `RequiresReview`؛ دو نتیجه متفاوت برای یک نمونه/آزمون → `conflict` + `RequiresReview`؛ تبدیل `A` به `mA` با ضریب؛ واحد ناشناخته → `unitError`؛ نبود مجموعه قواعد تأییدشده نافذ در `arrivalDate` → `RequiresReview`؛ کیفی `pass`/`fail`؛ ورودی‌ها + نسخه یکسان → رأی و `inputsHash` یکسان؛ رأی به نسخه نافذ در `arrivalDate` استناد می‌کند، نه آخرین نسخه
- [ ] T059 [P] [US2] آزمون‌ها در `chaincode/conformity/results_test.go`: `RecordSample` توسط `inspector`، `takenAt` «≥ arrivalDate»، وضعیت `Registered` → `Sampling`؛ `SubmitResults` توسط `lab` که MSP آن برابر با `assignedLabMsp` نمونه است؛ `testCode` «باید در مجموعه قواعد مربوط وجود داشته باشد»؛ `value` به‌صورت رشته اعشاری برای آزمون عددی و `passFail` برای آزمون کیفی الزامی است؛ هر نتیجه نامعتبر کل دسته را رد می‌کند؛ وضعیت → `ResultsReceived`
- [ ] T060 [P] [US2] آزمون‌ها در `chaincode/conformity/manual_verdict_test.go`: `RecordManualVerdict` توسط `assessor` به توجیه نیاز دارد و `manual=true` را تنظیم می‌کند؛ `ReassessDryRun` چیزی نمی‌نویسد و رأی اصلی را برمی‌گرداند
- [ ] T061 [P] [US2] آزمون‌ها در `gateway/internal/resultsfile/validate_test.go` مطابق contracts/results-file-template.md: سرستون دقیق، رد ارقام فارسی/عربی، حداکثر 18 رقم معنادار، `pass_fail` فقط `pass`/`fail`، `method` حداکثر 100 نویسه، حداکثر 5,000 سطر و 5 MB، رد `(sample_id, test_code)` تکراری، گزارش هر خطا به‌صورت `{row, column, code, message}`
- [ ] T062 [P] [US2] ایجاد داده‌های آزمون CSV شامل `testdata/results-pass.csv`، `testdata/results-boundary.csv`، `testdata/results-fail.csv`، `testdata/results-missing.csv`، `testdata/results-bad-row.csv` منطبق با `testdata/rulesets/iec60335-1-ed6-v1.json`
- [ ] T063 [US2] آزمون یکپارچگی برای سناریوی 2 در quickstart در `tests/integration/us2_assessment_test.go`

### پیاده‌سازی داستان کاربری 2

- [ ] T064 [P] [US2] پیاده‌سازی `RecordSample` در `chaincode/conformity/samples.go`
- [ ] T065 [P] [US2] پیاده‌سازی `SubmitResults` در `chaincode/conformity/results.go`
- [ ] T066 [US2] پیاده‌سازی تابع خالص `evaluate(ruleSet, results) Verdict` با استفاده از `shopspring/decimal` و پیمایش مرتب، به‌علاوه `AssessConsignment` (نقش‌های `assessor`، `service`)، `RecordManualVerdict`، `ReassessDryRun` در `chaincode/conformity/assess.go`
- [ ] T067 [P] [US2] پیاده‌سازی اعتبارسنجی CSV در `gateway/internal/resultsfile/validate.go`
- [ ] T068 [US2] پیاده‌سازی endpointهای `POST /consignments/{id}/samples`، `POST /results`، `POST /results/upload`، `POST /consignments/{id}/assess`، `POST /consignments/{id}/manual-verdict`، `POST /consignments/{id}/reassess` در `gateway/internal/api/results.go`؛ در اینجا ارزیابی خودکاری انجام نمی‌شود (gatewayهای آزمایشگاه هویت `service` ندارند)
- [ ] T069 [US2] افزودن handlerهای projector برای `RecordSample`، `SubmitResults`، `AssessConsignment`، `RecordManualVerdict` که در `rm_consignment`/`rm_verdict` می‌نویسند در `gateway/internal/projector/verdicts.go`؛ با رخداد `SubmitResults`، projector مربوط به gateway سازمان StdOrg تابع `AssessConsignment` را با هویت `service` سازمان StdOrg فراخوانی می‌کند (gatewayهای آزمایشگاه از این گام صرف‌نظر می‌کنند)
- [ ] T070 [P] [US2] ساخت صفحات آزمایشگاه `web/src/pages/lab/ResultsEntry.tsx` و `web/src/pages/lab/ResultsUpload.tsx` (جدول خطا به ازای هر سطر)
- [ ] T071 [US2] افزودن ثبت نمونه به `web/src/pages/ConsignmentDetail.tsx` و پنل رأی `web/src/components/VerdictPanel.tsx` (نتیجه هر آزمون، مقدار اندازه‌گیری‌شده در برابر حد، نسخه مجموعه قواعد، نشان دستی)

**نقطه بازبینی**: US1 + US2 کار می‌کنند — نتایج، رأی‌های بازتولیدپذیر ایجاد می‌کنند

---

## فاز 5: داستان کاربری 3 - تصمیم ارزیاب، بازنگری رأی، و ثبت گواهی (اولویت: P3)

**هدف**: ارزیابان با ارائه توجیه، رأی را تأیید یا بازنگری می‌کنند؛ گواهی‌ها یا ردها ثبت می‌شوند؛ اعتراض‌ها و تعیین تکلیف‌ها پیگیری می‌شوند؛ به واردکنندگان اطلاع‌رسانی می‌شود

**آزمون مستقل**: quickstart.md سناریوی 3

### آزمون‌های داستان کاربری 3 ⚠️ (ابتدا نوشته شوند، **باید** شکست بخورند)

- [ ] T072 [P] [US3] آزمون‌ها در `chaincode/conformity/decision_test.go`: `DecideConsignment` توسط `assessor`؛ `override` بدون `justification` → `ERR_INVALID` («حداکثر 2000 نویسه»)؛ `certificateRef` «هنگامی که نتیجه `Conforming` است الزامی است»؛ `rejectionReasons` «هنگامی که نتیجه `NonConforming` است الزامی است»؛ فراخوانی که هر نتیجه‌ای برای محموله ارسال کرده، یا از MSP آزمایشگاه ارسال‌کننده است → `ERR_SOD`؛ رأی خودکار و بازنگری رأی هر دو در تاریخچه باقی می‌مانند؛ هش `documentChecks` که با `sha256` روی زنجیره متفاوت باشد → `ERR_HASH_MISMATCH`، و نبود بررسی برای هر سند حذف‌نشده → `ERR_INVALID`
- [ ] T073 [P] [US3] آزمون‌ها در `chaincode/conformity/appeal_test.go`: `FileAppeal` فقط توسط `service` برای واردکننده ثبت‌شده، `grounds` «حداکثر 2000 نویسه»، وضعیت → `UnderAppeal`؛ `ResolveAppeal` توسط تصمیم‌گیرنده اصلی → `ERR_SOD`؛ رسیدگی، تصمیمی از نوع `appealOutcome` با `previousDecisionId` ایجاد می‌کند
- [ ] T074 [P] [US3] آزمون‌ها در `chaincode/conformity/disposition_test.go`: `ChooseDisposition` به `NonConforming` نهایی نیاز دارد، `type` در `reExport`/`destruction`/`rework`، وضعیت → `DispositionPending`؛ `CompleteDisposition` توسط `inspector` با `evidenceDocumentIds` «≥ 1 هنگام تکمیل»؛ `rework` → `Sampling`، سایر موارد → `Closed`
- [ ] T075 [US3] آزمون یکپارچگی برای سناریوی 3 در quickstart در `tests/integration/us3_decisions_test.go`

### پیاده‌سازی داستان کاربری 3

- [ ] T076 [P] [US3] پیاده‌سازی `DecideConsignment` در `chaincode/conformity/decision.go`
- [ ] T077 [P] [US3] پیاده‌سازی `FileAppeal`، `ResolveAppeal` در `chaincode/conformity/appeal.go`
- [ ] T078 [P] [US3] پیاده‌سازی `ChooseDisposition`، `CompleteDisposition` در `chaincode/conformity/disposition.go`
- [ ] T079 [P] [US3] پیاده‌سازی اعلان‌ها (SMTP از طریق `net/smtp` + سطرهای جدول `notification`، الگوهای fa/en) در `gateway/internal/notify/notify.go`
- [ ] T080 [US3] پیاده‌سازی endpointهای `POST /consignments/{id}/decisions`، `POST /consignments/{id}/appeals`، `POST /appeals/{id}/resolve`، `POST /consignments/{id}/dispositions`، `POST /dispositions/{id}/complete`، `GET /me/notifications` در `gateway/internal/api/decisions.go`؛ اطلاع‌رسانی به واردکننده هنگام تصمیم نهایی و نتیجه اعتراض؛ پیش از فراخوانی `DecideConsignment`، SHA-256 هر سند حذف‌نشده از `gateway/internal/store/documents.go` دوباره محاسبه و به‌عنوان `documentChecks` ارسال شود
- [ ] T081 [US3] افزودن handlerهای projector برای تصمیم‌ها، اعتراض‌ها، تعیین تکلیف‌ها که در `rm_decision`/`rm_disposition` می‌نویسند (شامل نمایه ارجاع گواهی) در `gateway/internal/projector/decisions.go`
- [ ] T082 [P] [US3] ساخت `web/src/pages/assessor/DecisionForm.tsx` (تأیید/بازنگری رأی، توجیه برای بازنگری رأی الزامی است) و `web/src/pages/inspector/Disposition.tsx`
- [ ] T083 [P] [US3] ساخت صفحات درگاه واردکننده `web/src/pages/importer/MyConsignments.tsx`، `web/src/pages/importer/Appeal.tsx`، `web/src/pages/importer/Notifications.tsx`

**نقطه بازبینی**: چرخه کامل تصمیم از ابتدا تا انتها کار می‌کند

---

## فاز 6: داستان کاربری 4 - ممیزی و راستی‌آزمایی مستقل (اولویت: P4)

**هدف**: ممیزان تاریخچه، اسناد، و رأی‌ها را به‌صورت فقط‌خواندنی و منطبق با peerهای آزمایشگاه راستی‌آزمایی می‌کنند

**آزمون مستقل**: quickstart.md سناریوی 4

- [ ] T084 [P] [US4] آزمون در `chaincode/conformity/auditor_readonly_test.go`: هر تابع نوشتنی که با نقش `auditor` فراخوانی شود → `ERR_FORBIDDEN`؛ `GetHistory`، `VerifyDocument`، `ReassessDryRun` مجاز هستند
- [ ] T085 [P] [US4] نوشتن `network/scripts/peer-query.sh <org> <function> <args...>` که `peer chaincode query` را روی peer آن سازمان اجرا می‌کند
- [ ] T086 [US4] آزمون یکپارچگی برای سناریوی 4 در quickstart در `tests/integration/us4_audit_test.go` (تاریخچه از طریق gateway برابر با تاریخچه از `peer0.lab1` است)
- [ ] T087 [US4] پیاده‌سازی `GET /certificates/{ref}` در `gateway/internal/api/certificates.go`
- [ ] T088 [US4] ساخت `web/src/pages/auditor/AuditLookup.tsx` (بر اساس شناسه محموله یا شماره گواهی) و `web/src/pages/auditor/AuditConsignment.tsx` (تاریخچه، راستی‌آزمایی همه اسناد، ارزیابی مجدد و نمایش انطباق/عدم انطباق با رأی ثبت‌شده)

**نقطه بازبینی**: ممیزان می‌توانند هر تصمیمی را بدون کمک اپراتور راستی‌آزمایی کنند

---

## فاز 7: داستان کاربری 5 - مدیریت مجموعه قواعد انطباق (اولویت: P5)

**هدف**: کارشناسان فنی نسخه‌های مجموعه قواعد را از طریق رابط کاربری با تاریخ‌های اعتبار پیش‌نویس و تأیید می‌کنند

**آزمون مستقل**: quickstart.md سناریوی 5

- [ ] T089 [US5] آزمون یکپارچگی برای سناریوی 5 در quickstart در `tests/integration/us5_rulesets_test.go` (خودتأییدی `403 ERR_SOD`؛ `effectiveFrom` آینده، v1 را نافذ نگه می‌دارد؛ رأی‌های قدیمی همچنان به v1 استناد می‌کنند)
- [ ] T090 [US5] پیاده‌سازی endpointهای `GET /rule-sets`، `GET /rule-sets/{id}/versions/{v}`، `POST /rule-sets`، `POST /rule-sets/{id}/versions/{v}/approve` در `gateway/internal/api/rulesets.go`
- [ ] T091 [P] [US5] ساخت `web/src/pages/techOfficer/RuleSetList.tsx` و `web/src/pages/techOfficer/RuleSetEditor.tsx` (آزمون‌ها با بند، واحد، حداقل/حداکثر، شامل‌بودن، الزامی‌بودن، samplesRequired؛ جدول ضرایب واحد؛ تاریخ اعتبار)

**نقطه بازبینی**: همه داستان‌های کاربری به‌طور مستقل کار می‌کنند

---

## فاز 8: بهبود نهایی و موضوعات مشترک

- [ ] T092 پیاده‌سازی `GET /reports/summary?groupBy=status|category|standard|importer&from=&to=` شامل شمار بازنگری‌های رأی و تعیین تکلیف‌های در انتظار در `gateway/internal/api/reports.go`، و `web/src/pages/Reports.tsx`
- [ ] T093 آزمون یکپارچگی تاب‌آوری برای سناریوی 6 در quickstart در `tests/integration/resilience_test.go` (توقف `peer0.lab1`، ارسال Lab1 از طریق peer سازمان StdOrg، راه‌اندازی مجدد، تطابق تاریخچه‌ها)
- [ ] T094 [P] افزودن فرمان بازسازی مدل خواندنی `gateway/cmd/rebuild-readmodel/main.go` (خالی‌کردن `rm_*`، بازنشانی checkpoint، بازپخش از بلوک 0)
- [ ] T095 [P] افزودن `network/scripts/add-lab.sh` (ایجاد MSP آزمایشگاه، به‌روزرسانی پیکربندی کانال، بازتولید سیاست تأیید با آزمایشگاه جدید، approve/commit چرخه حیات chaincode توسط همه سازمان‌ها) و مستندسازی آن در `README.md`
- [ ] T096 [P] افزودن پیکربندی Prometheus + کاوشگر blackbox در `network/config/prometheus/prometheus.yml` که `/healthz` و یک خواندن احرازهویت‌شده را در ساعات کاری هر دقیقه کاوش می‌کند (SC-009)
- [ ] T097 [P] آزمون دود Playwright برای هر داستان در `web/tests/e2e/us1.spec.ts` … `web/tests/e2e/us5.spec.ts`، هر کدام در هر دو زبان `fa` و `en` اجرا شود
- [ ] T098 [P] آزمون کامل‌بودن ترجمه در `web/tests/i18n.test.ts` (هر کلید در `fa.json` در `en.json` وجود داشته باشد و برعکس)
- [ ] T099 مقاوم‌سازی امنیتی در `gateway/internal/api/router.go`: CSP و سرآیندهای امنیتی، تشخیص MIME بارگذاری‌ها در برابر `kind` اعلام‌شده، محدودیت اندازه درخواست (25 MB برای اسناد، 5 MB برای فایل‌های نتایج)
- [ ] T100 [P] آزمون بار پشت برچسب ساخت `load` در `tests/load/load_test.go`: نیمرخ 500 محموله در روز با 50 کاربر هم‌زمان؛ اطمینان از اینکه 95% رأی‌ها در کمتر از 1 دقیقه پس از ارسال نتایج صادر می‌شوند (SC-002، SC-008)
- [ ] T101 اجرای همه سناریوهای quickstart.md و `make test-chaincode test-integration test-web`؛ ثبت نتایج در یادداشت‌های `specs/001-goods-conformity-ledger/quickstart.md`
- [ ] T102 نوشتن `network/scripts/upgrade-chaincode.sh` (بسته‌بندی نسخه جدید، تأیید توسط هر سازمان، commit با همان سیاست تأیید، ثبت نسخه و MSPهای تأییدکننده در مدخل انتشار) و بخش روال ارتقا در `README.md`؛ افزودن آزمون یکپارچگی `tests/integration/upgrade_test.go` که نشان دهد محموله‌ها، رأی‌ها، و تاریخچه ثبت‌شده تحت v1 پس از ارتقا به v2 بدون تغییر و قابل خواندن هستند
- [ ] T103 [P] افزودن `CHANGELOG.md` با بخش‌های «نسخه‌های قرارداد» و «نسخه‌های مجموعه قواعد» (نسخه، تاریخ اعتبار، تأییدکنندگان) و یک گام CI در `.github/workflows/ci.yml` که هرگاه `chaincode/` یا `testdata/rulesets/` بدون تغییر در `CHANGELOG.md` تغییر کند، شکست می‌خورد

---

## وابستگی‌ها و ترتیب اجرا

### وابستگی‌های فازها

- **راه‌اندازی (فاز 1)**: ندارد
- **زیربنایی (فاز 2)**: پس از راه‌اندازی — همه داستان‌ها را مسدود می‌کند
- **داستان‌های کاربری (فازهای 3–7)**: پس از زیربنایی
- **بهبود نهایی (فاز 8)**: پس از داستان‌هایی که به آن‌ها مربوط است

### وابستگی‌های داستان‌های کاربری

- **US1 (P1)**: پس از زیربنایی؛ بدون وابستگی به داستان دیگر
- **US2 (P2)**: به محموله‌های US1 (T046، T047) و مجموعه قواعد فاز زیربنایی (T021، T023) نیاز دارد
- **US3 (P3)**: به رأی‌های US2 (T066) نیاز دارد
- **US4 (P4)**: فقط‌خواندنی روی داده‌های US1–US3؛ می‌تواند پس از US1 با سناریوهای جزئی آغاز شود
- **US5 (P5)**: chaincode آن از قبل در فاز زیربنایی است؛ فقط به gateway + وب نیاز دارد؛ می‌تواند به‌موازات US1 اجرا شود

### درون هر داستان کاربری

- ابتدا آزمون‌های chaincode نوشته شوند و شکست بخورند (منشور V)، سپس پیاده‌سازی chaincode
- chaincode پیش از endpointهای gateway؛ endpointها پیش از handlerهای projector و صفحات وب
- آزمون یکپارچگی پیش از نقطه بازبینی داستان قبول شود

### فرصت‌های موازی‌سازی

- راه‌اندازی: T003، T004، T005، T006، T011
- زیربنایی: T013، T014؛ T026، T027، T029، T031؛ T035، T036
- US1: همه آزمون‌های T040–T045؛ T048، T049، T050، T051؛ T055، T057
- US2: T058–T062؛ T064، T065، T067؛ T070
- US3: T072–T074؛ T076–T079؛ T082، T083
- US5 را می‌تواند توسعه‌دهنده دوم هم‌زمان با پیشرفت US1 بسازد

---

## نمونه موازی‌سازی: داستان کاربری 1

```bash
# همه آزمون‌های شکست‌خورده chaincode را با هم بنویسید:
Task: "آزمون‌ها در chaincode/conformity/consignment_test.go"
Task: "آزمون‌ها در chaincode/conformity/correction_test.go"
Task: "آزمون‌ها در chaincode/conformity/documents_test.go"
Task: "آزمون‌ها در chaincode/conformity/split_test.go"
Task: "آزمون‌ها در chaincode/conformity/history_test.go"

# سپس فایل‌های پیاده‌سازی مستقل:
Task: "پیاده‌سازی AttachDocument، RecordDocumentRemoval، VerifyDocument در chaincode/conformity/documents.go"
Task: "پیاده‌سازی GetHistory در chaincode/conformity/history.go"
Task: "پیاده‌سازی مخزن اسناد در gateway/internal/store/documents.go"
```

---

## راهبرد پیاده‌سازی

### ابتدا MVP (فقط داستان کاربری 1)

1. فاز 1 راه‌اندازی → فاز 2 زیربنایی
2. فاز 3 US1
3. **توقف و اعتبارسنجی**: سناریوی 1 در quickstart قبول شود
4. نمایش ثبت ضددست‌کاری به ذی‌نفعان

### تحویل تدریجی

1. US1 → دفتر ثبت ضددست‌کاری (MVP)
2. US2 → رأی‌های خودکار
3. US3 → تصمیم‌های قابل استفاده از نظر حقوقی (حداقل لازم برای بهره‌برداری)
4. US4 → ممیزی آماده اعتباربخشی
5. US5 → مدیریت خودخدمت قواعد (تا آن زمان اسکریپت seed آن را پوشش می‌دهد)

### راهبرد تیم موازی

1. کل تیم: راه‌اندازی + زیربنایی
2. سپس: توسعه‌دهنده A روی chaincode داستان‌های US1 → US2 → US3؛ توسعه‌دهنده B روی gateway/وب برای همان
   داستان‌ها؛ توسعه‌دهنده C روی US5 و US4

---

## یادداشت‌ها

- [P] = فایل‌های متفاوت، بدون وابستگی به وظایف ناتمام
- پس از هر وظیفه یا گروه منطقی commit کنید
- هر تغییری در `chaincode/conformity/` به دو بازبین نیاز دارد (دروازه گردش کار منشور)
