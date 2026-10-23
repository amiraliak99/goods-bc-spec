# REST API مربوط به Gateway (v1)

مسیر پایه `/api/v1`. JSON روی HTTPS. احراز هویت: OIDC bearer token از realm مربوط به Keycloak
سازمان. `Accept-Language: fa|en` زبان پیام‌ها را تعیین می‌کند. خطاها:
`{ "code": "ERR_...", "message": "...", "details": [...] }` با نگاشت کدهای chaincode به
HTTP: `ERR_INVALID` 422، `ERR_FORBIDDEN`/`ERR_SOD` 403، `ERR_NOT_FOUND` 404،
`ERR_STATE` 409، `ERR_HASH_MISMATCH` 409. هر پاسخ نوشتن شامل `txId` است.

همین API توسط gateway سازمان StdOrg و gateway هر آزمایشگاه ارائه می‌شود؛ هر gateway
فقط endpointهایی را در دسترس قرار می‌دهد که نقش‌های کاربرانش می‌توانند از آن‌ها استفاده کنند.

## محموله‌ها

| متد | مسیر | نقش | نگاشت به |
|--------|------|------|---------|
| POST | `/consignments` | inspector | `RegisterConsignment` |
| GET | `/consignments?status=&category=&standard=&importer=&from=&to=` | staff، auditor | مدل خواندنی |
| GET | `/consignments/{id}` | staff، auditor، importer (خودی) | `GetConsignment` |
| GET | `/consignments/{id}/history` | staff، auditor | `GetHistory` |
| POST | `/consignments/{id}/corrections` | inspector | `CorrectConsignment` |
| POST | `/consignments/{id}/split` | inspector | `SplitConsignment` |
| POST | `/consignments/{id}/documents` (multipart) | inspector، lab | ذخیره + `AttachDocument` |
| GET | `/consignments/{id}/documents/{docId}` | staff، auditor | دانلود فایل |
| GET | `/consignments/{id}/documents/{docId}/verify` | auditor، staff | بازمحاسبهٔ هش + `VerifyDocument` |
| DELETE | `/consignments/{id}/documents/{docId}` | admin | تهی‌کردن بایت‌ها + `RecordDocumentRemoval` |
| POST | `/consignments/{id}/samples` | inspector | `RecordSample` |

## نتایج و ارزیابی

| متد | مسیر | نقش | نگاشت به |
|--------|------|------|---------|
| POST | `/results` (آرایهٔ JSON) | lab | `SubmitResults` |
| POST | `/results/upload` (CSV، ر.ک. results-file-template.md) | lab | اعتبارسنجی، سپس `SubmitResults` |
| POST | `/consignments/{id}/assess` | assessor | `AssessConsignment` (همچنین پس از دریافت نتایج به‌طور خودکار فعال می‌شود) |
| POST | `/consignments/{id}/manual-verdict` | assessor | `RecordManualVerdict` |
| POST | `/consignments/{id}/reassess` | auditor، assessor | `ReassessDryRun` |

## تصمیم‌ها، اعتراض‌ها، تعیین تکلیف‌ها

| متد | مسیر | نقش | نگاشت به |
|--------|------|------|---------|
| POST | `/consignments/{id}/decisions` | assessor | `DecideConsignment` (اعلان برای واردکننده ارسال می‌کند) |
| POST | `/consignments/{id}/appeals` | importer | `FileAppeal` |
| POST | `/appeals/{id}/resolve` | assessor | `ResolveAppeal` |
| POST | `/consignments/{id}/dispositions` | inspector، assessor | `ChooseDisposition` |
| POST | `/dispositions/{id}/complete` | inspector | `CompleteDisposition` |

## مجموعه قواعد

| متد | مسیر | نقش | نگاشت به |
|--------|------|------|---------|
| GET | `/rule-sets`، `/rule-sets/{id}/versions/{v}` | همهٔ staff، auditor | `GetRuleSet` |
| POST | `/rule-sets` | techOfficer | `DraftRuleSet` |
| POST | `/rule-sets/{id}/versions/{v}/approve` | techOfficer | `ApproveRuleSet` |

## شرکت‌کنندگان، گزارش‌ها، اعلان‌ها

| متد | مسیر | نقش | نگاشت به |
|--------|------|------|---------|
| POST | `/participants`، `/participants/{id}/revoke` | admin | `RegisterParticipant`، `RevokeParticipant` |
| GET | `/reports/summary?groupBy=status|category|standard|importer&from=&to=` | staff، auditor | مدل خواندنی، شامل شمار بازنگری‌های رأی |
| GET | `/certificates/{ref}` | auditor، staff | مدل خواندنی ← شناسهٔ محموله |
| GET | `/me/notifications` | any | جدول اعلان‌ها |
| GET | `/healthz`، `/metrics` | none / internal | probeها، Prometheus |
