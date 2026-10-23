# طرح پیاده‌سازی: دفتر کل انطباق کالای وارداتی

**شاخه**: `001-goods-conformity-ledger` | **تاریخ**: 2026-10-23 | **مشخصات**: [spec.md](spec.md)

**ورودی**: مشخصات ویژگی از `/specs/001-goods-conformity-ledger/spec.md`

## خلاصه

یک دفتر کل مجوزدار که میان سازمان استاندارد (StdOrg) و هر آزمایشگاه اعتباربخشی‌شده
مشترک است، هر مرحله از چرخهٔ عمر یک محمولهٔ وارداتی را به‌صورت تراکنش‌های امضاشده و فقط‌افزودنی
ثبت می‌کند. chaincode مجموعه قواعد انطباق نسخه‌بندی‌شده را نگه می‌دارد و رأی‌ها را
به‌صورت قطعی در هر peer تأییدکننده (endorsing) محاسبه می‌کند، بنابراین نه سازمان و نه یک آزمایشگاه
نمی‌تواند به‌تنهایی نتیجه‌ای جعلی ایجاد کند. یک gateway مبتنی بر Go برای هر سازمان، یک REST API و یک برنامهٔ وب فارسی/انگلیسی
در اختیار کارکنان، آزمایشگاه‌ها، واردکنندگان و ممیزان قرار می‌دهد، داده‌های شخصی و اسناد را
خارج از زنجیره (با لنگر هش) نگه می‌دارد و رویدادهای دفتر کل را برای گزارش‌ها در یک مدل خواندنی بازتاب می‌دهد.
جزئیات: [research.md](research.md).

## زمینهٔ فنی

**زبان/نسخه**: Go 1.23 (chaincode + gateway)؛ TypeScript 5 (وب)

**وابستگی‌های اصلی**: Hyperledger Fabric 2.5 LTS (peerها، ordererهای Raft، Fabric CA)،
`fabric-contract-api-go` v2، `fabric-gateway` Go SDK، `shopspring/decimal`، Keycloak 26
(OIDC)، React 18 + Vite + `react-i18next`

**ذخیره‌سازی**: دفتر کل Fabric (روی زنجیره، وضعیت LevelDB)؛ PostgreSQL 16 خارج از زنجیره (داده‌های
شخصی، اسناد ≤ 25 MB، کلیدهای رمزشدهٔ کاربران، مدل خواندنی)

**آزمون**: `go test` (آزمون واحد chaincode با stub جعلی، gateway)، آزمون‌های یکپارچگی روی
یک شبکهٔ محلی Fabric سه‌سازمانی در Docker Compose، Vitest، Playwright

**پلتفرم هدف**: سرورهای Linux در کانتینرها (Docker Compose برای توسعه/پایلوت)؛ مرورگرهای
دسکتاپ همیشه‌به‌روز

**نوع پروژه**: سرویس وب + chaincode + SPA وب

**اهداف کارایی**: رأی در کمتر از 1 دقیقه پس از تکمیل نتایج برای 95% محموله‌ها
(SC-002)؛ 500 محموله در روز، 50 کاربر هم‌زمان (SC-008)؛ فرم ثبت قابل تکمیل در
کمتر از 5 دقیقه (SC-007)

**محدودیت‌ها**: دسترس‌پذیری 99.5% در ساعات کاری (SC-009)؛ تحمل از دست رفتن هر
نود واحد بدون از دست رفتن داده (SC-010)؛ chaincode قطعی (بدون اعداد اعشاری شناور/ساعت/تصادفی‌بودن)؛
بدون داده‌های شخصی روی زنجیره؛ پیش‌فرض فارسی RTL + انگلیسی؛ نمایش جلالی/میلادی، ذخیره‌سازی
به UTC

**مقیاس/دامنه**: پایلوت با 1 StdOrg + 2–10 آزمایشگاه؛ حدود 180 هزار محموله در سال؛ در ابتدا حدود 10 مجموعه
قواعد؛ حدود 25 صفحه

## بررسی منشور

*دروازه: **باید** پیش از پژوهش فاز 0 قبول شود. پس از طراحی فاز 1 دوباره بررسی شود.*

| اصل | نحوهٔ انطباق طراحی | وضعیت |
|-----------|------------------------|--------|
| I. ثبت تغییرناپذیر حقیقت | همهٔ رویدادهای چرخهٔ عمر تراکنش‌های Fabric هستند؛ اصلاحات تراکنش‌های جدید هستند (`CorrectConsignment`)؛ وضعیت از تاریخچهٔ کلید قابل استخراج است | قبول |
| II. قواعد قطعی و نسخه‌بندی‌شده | مجموعه قواعد روی زنجیره نسخه‌بندی می‌شوند و به استاندارد + بند ارجاع می‌دهند؛ رأی، نسخهٔ مجموعه قواعد + هش ورودی‌ها را ذخیره می‌کند؛ محاسبات اعشاری دقیق؛ بررسی ایستای قطعیت؛ لغو رأی (بازنگری) نیازمند توجیه است | قبول |
| III. حداقل داده روی زنجیره | تنها شناسه‌ها + SHA-256 روی زنجیره؛ اسناد و داده‌های واردکننده در PostgreSQL؛ `RecordDocumentRemoval` برای حذف قانونی | قبول |
| IV. هویت مجوزدار، حداقل دسترسی | Fabric MSP برای هر سازمان، گواهی‌های فردی با ویژگی‌های نقش؛ دسترسی در chaincode بررسی می‌شود؛ تفکیک وظایف برای نتایج/تصمیم و پیش‌نویس/تأیید؛ ابطال تنها اقدامات جدید را مسدود می‌کند. واردکنندگان توسط Keycloak سازمان که نقش مرجع اعتماد آن برای واردکنندگان را دارد راستی‌آزمایی می‌شوند؛ اعتراض‌های آنان با هویت `service` سازمان StdOrg امضا می‌شود و شناسهٔ حساب واردکننده و هشی از ادعاهای توکن ورود او را ثبت می‌کند (واردکنندگان تنها برای اعتراض روی دفتر کل اقدام می‌کنند؛ صدور گواهی برای هزاران واردکننده بار پشتیبانی را افزایش می‌دهد) | قبول |
| V. ایمنی قرارداد هوشمند | chaincode آزمون‌محور؛ آزمون‌های مرزی/مخرب جدول‌محور؛ تحلیل ایستا؛ ارتقا از طریق چرخهٔ عمر Fabric با الزام تأیید همهٔ سازمان‌ها | قبول |
| VI. ردیابی‌پذیری و ممیزی‌پذیری | یک consignmentId همه‌چیز را پیوند می‌دهد؛ `GetHistory`، `VerifyDocument`، `ReassessDryRun` برای ممیزان؛ لاگ‌ها `txId` را دربر دارند | قبول |
| VII. سادگی و اجزای اثبات‌شده | تنها پلتفرم‌های بالغ؛ یک زبان برای chaincode + gateway؛ یک پایگاه داده؛ فعلاً بدون object store، بدون صف پیام، بدون Kubernetes. وابستگی‌ها در زیر توجیه شده‌اند | قبول |
| محدودیت‌های حوزه | آزمون‌های قواعد به بند ارجاع می‌دهند؛ مسیر رأی دستی؛ برچسب‌های زمانی UTC | قبول |
| دروازه‌های گردش کار | CI آزمون‌های واحد chaincode، یکپارچگی روی شبکهٔ چندنودی و آزمون‌های وب را اجرا می‌کند | قبول |

**توجیه وابستگی‌ها (اصل VII)**:

| وابستگی | مورد نیاز برای | چرا نه چیزی ساده‌تر |
|-----------|-----------|---------------------------|
| Hyperledger Fabric | دفتر کل مشترک با قابلیت آشکارسازی دست‌کاری، عضویت سازمان‌ها | الزام مشخصات؛ تنها گزینهٔ مجوزدار بالغ با تأیید (endorsement) در سطح سازمان |
| PostgreSQL | داده‌های شخصی، اسناد، مدل خواندنی | یک مخزن همهٔ نیازهای خارج از زنجیره را پوشش می‌دهد |
| Keycloak | ورود OIDC، TOTP، حساب‌های واردکنندگان | ساخت احراز هویت یک ریسک امنیتی است؛ Fabric CA ورود کاربر را انجام نمی‌دهد |
| `shopspring/decimal` | مقایسهٔ دقیق حدود | کتابخانهٔ استاندارد نوع اعشاری ندارد؛ `big.Rat` امکانات تجزیه/گردکردن مورد نیاز را ندارد |
| React + i18next | فرم‌های تعاملی، جابه‌جایی RTL/LTR | قالب‌های سادهٔ سمت سرور ورود نتایج و نماهای تاریخچه را دشوارتر می‌کنند |

**بررسی مجدد پس از طراحی (پس از فاز 1)**: مدل داده، قراردادها و quickstart همهٔ
موارد فوق را در وضعیت قبول نگه می‌دارند. هیچ تخطی‌ای وجود ندارد؛ ردیابی پیچیدگی خالی است.

## ساختار پروژه

### مستندات (این ویژگی)

```text
specs/001-goods-conformity-ledger/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/
│   ├── chaincode.md
│   ├── gateway-api.md
│   └── results-file-template.md
└── tasks.md             # Phase 2 output (/speckit-tasks)
```

### کد منبع (ریشهٔ مخزن)

```text
chaincode/conformity/          # Go chaincode
├── contract.go                # Function entry points + access checks
├── consignment.go             # Registration, corrections, split, samples, state machine
├── results.go                 # SubmitResults
├── assess.go                  # Deterministic rule evaluation
├── decision.go                # Decisions, appeals, dispositions
├── ruleset.go                 # Draft / approve / lookup by date
├── participant.go
└── *_test.go                  # Unit tests next to code

gateway/                       # Go service, one deployment per org
├── cmd/gateway/main.go
├── internal/api/              # REST handlers, OIDC middleware, error mapping
├── internal/fabric/           # Gateway SDK client, per-user signer
├── internal/store/            # PostgreSQL: importers, documents, keys, read model
├── internal/projector/        # Block event listener → read model
├── internal/resultsfile/      # CSV validation
├── internal/notify/           # SMTP + in-app notifications
└── migrations/

web/                           # React + TypeScript SPA
├── src/pages/                 # Per role: inspector, lab, assessor, techOfficer, auditor, importer, admin
├── src/components/
├── src/i18n/fa.json, en.json
└── tests/

network/                       # Local and pilot Fabric network
├── compose/                   # docker-compose files for 3 orgs, PostgreSQL, Keycloak
├── config/                    # configtx.yaml, crypto config, Keycloak realm exports
└── scripts/                   # network-up, deploy, peer-query, token

tests/integration/             # Gateway against running network (Go)
testdata/                      # Rule sets, CSV fixtures used by quickstart
Makefile
```

**تصمیم ساختاری**: سه بخش قابل استقرار (chaincode، gateway، وب)، زیرا chaincode در Fabric
**باید** جدا از سرویس کلاینت بسته‌بندی شود و SPA به‌صورت فایل‌های ایستا
ارائه می‌شود که gateway آن‌ها را سرو می‌کند. آزمون‌های واحد کنار کد Go قرار دارند؛ آزمون‌های میان‌جزئی در
`tests/integration/` قرار دارند.

## ردیابی پیچیدگی

هیچ تخطی از منشور برای توجیه وجود ندارد.
