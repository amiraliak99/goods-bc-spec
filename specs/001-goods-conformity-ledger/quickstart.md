# راهنمای سریع: اعتبارسنجی سرتاسری دفتر کل انطباق

هر داستان کاربر را روی یک شبکهٔ محلی اثبات می‌کند. جزئیات endpointها:
[contracts/gateway-api.md](contracts/gateway-api.md). موجودیت‌ها:
[data-model.md](data-model.md).

## پیش‌نیازها

- Linux یا macOS، Docker نسخهٔ 24+ همراه با Compose v2، Go 1.23، Node 20، `jq`، `curl`.
- ۸ گیگابایت RAM آزاد (۳ سازمان: StdOrg + Lab1 + Lab2).

## راه‌اندازی

```bash
make network-up        # Fabric CAs, 3 orderers, StdOrg x2 peers, Lab1/Lab2 peers, channel
make chaincode-deploy  # package, approve by all orgs, commit `conformity`
make stack-up          # PostgreSQL, Keycloak (seed users), gateways (StdOrg :8080, Lab1 :8081)
make seed              # pilot rule set "IEC 60335-1 ed.6 v1", one importer, test users
```

کاربران اولیه (گذرواژه `Passw0rd!`، TOTP در محیط توسعه غیرفعال است): `inspector1`، `assessor1`،
`assessor2`، `techofficer1`، `techofficer2`، `auditor1`، `admin1` روی StdOrg؛ `lab1user`
روی Lab1؛ `importer1` در درگاه واردکننده. `scripts/token.sh <user>` یک bearer
token چاپ می‌کند.

## سناریو ۱ — ثبت تغییرناپذیر (داستان ۱)

1. در نقش `inspector1`، `POST /consignments` همراه با یک فاکتور ← انتظار: `201` همراه با
   `consignmentId` و `txId`.
2. `POST /consignments/{id}/corrections` با تغییر مقدار (quantity) همراه با دلیل ← `GET .../history`
   هم ورودی اصلی و هم ورودی اصلاحیه را نشان می‌دهد.
3. بایت‌های فاکتور را مستقیماً در PostgreSQL بازنویسی کنید، سپس
   `GET .../documents/{docId}/verify` ← `status: "mismatch"`.
4. `POST /consignments` بدون `standardRef` ← `422` با ذکر نام `standardRef`.

## سناریو ۲ — ارزیابی خودکار (داستان ۲)

1. `inspector1` نمونه‌ای را که به Lab1 تخصیص یافته ثبت می‌کند.
2. در نقش `lab1user` روی `:8081`، فایل `testdata/results-pass.csv` را بارگذاری کنید ← رأی `Conforming`.
3. روی یک محمولهٔ جدید با `testdata/results-boundary.csv` تکرار کنید (مقدار برابر با یک
   حد شامل است) ← `Conforming`؛ با `results-fail.csv` ← `NonConforming` همراه با نمایش مقدار
   و حد؛ با `results-missing.csv` ← `RequiresReview` همراه با فهرست آزمون‌های جاافتاده.
4. فایل `testdata/results-bad-row.csv` را بارگذاری کنید ← `422` همراه با خطاهای هر سطر و بدون ثبت هیچ داده‌ای.
5. `POST /consignments/{id}/reassess` در نقش `auditor1` ← رأی و نسخهٔ مجموعه قواعد یکسان.

## سناریو ۳ — تصمیم‌ها، بازنگری‌های رأی، اعتراض‌ها، تعیین تکلیف‌ها (داستان ۳)

1. `assessor1` رأی Conforming را همراه با یک `certificateRef` تأیید می‌کند ← `importer1`
   تصمیم را در `/me/notifications` می‌بیند.
2. `assessor1` یک رأی NonConforming را بدون توجیه بازنگری می‌کند ← `422`؛ با
   توجیه ← `201`؛ تاریخچه هم رأی و هم بازنگری رأی را نشان می‌دهد.
3. `importer1` یک اعتراض ثبت می‌کند؛ `assessor1` (تصمیم‌گیرندهٔ اولیه) تلاش می‌کند آن را رسیدگی کند ← `403`؛
   `assessor2` رسیدگی می‌کند ← تصمیم جدید پیوندشده.
4. روی یک محمولهٔ NonConforming، `rework` را انتخاب کنید و با ارائهٔ شاهد تکمیل کنید ← وضعیت
   `Sampling`؛ نتایج جدید رأی جدیدی تولید می‌کنند که به همان محموله پیوند می‌خورد.

## سناریو ۴ — ممیزی مستقل (داستان ۴)

1. در نقش `auditor1`، `GET /certificates/{ref}` و سپس `/history` ← ورودی‌های زمانی‌مرتب همراه با
   کنشگر و مُهر زمانی.
2. هر `POST` در نقش `auditor1` ← `403`.
3. همان تاریخچه را مستقیماً از یک peer در Lab1 تأیید کنید:
   `scripts/peer-query.sh lab1 GetHistory <consignmentId>` ← ورودی‌های یکسان.

## سناریو ۵ — نسخه‌های مجموعه قواعد (داستان ۵)

1. `techofficer1` پیش‌نویس v2 را با حدی سخت‌گیرانه‌تر و `effectiveFrom` برابر فردا تهیه می‌کند؛
   `techofficer1` تأیید می‌کند ← `403 ERR_SOD`؛ `techofficer2` تأیید می‌کند ← `200`.
2. محموله‌ای با تاریخ ورود امروز را ارزیابی کنید ← از v1 استفاده می‌شود. رأی‌های قدیمی همچنان به v1 ارجاع می‌دهند.

## سناریو ۶ — تاب‌آوری (SC-010)

1. `docker stop peer0.lab1` ← Lab1 نتایج را از طریق peer سازمان StdOrg ارسال می‌کند؛ نوشتن‌ها همچنان موفق می‌شوند
   (با endorsement توسط StdOrg + Lab2).
2. `docker start peer0.lab1` ← پس از همگام‌سازی، `scripts/peer-query.sh lab1 GetHistory`
   با StdOrg مطابقت دارد.

## بررسی‌های خودکار

```bash
make test-chaincode     # unit tests + determinism static check
make test-integration   # gateway against local network, includes Scenario 6
make test-web           # Vitest + Playwright smoke per story
```

همهٔ آن‌ها **باید** بدون هیچ شکستی موفق شوند.
