# Results File Template (CSV)

UTF-8, comma-separated, first line is the header below, exactly. Decimal separator is `.`
(Persian or Arabic digits are rejected, to avoid ambiguity). One row per sample-test result.

```text
sample_id,test_code,value,pass_fail,unit,method,report_document_id
01JB3Q...,IR_MIN,12.5,,MOhm,ISO-XXXX-5.2,01JB3R...
01JB3Q...,LABEL,,pass,,VISUAL,01JB3R...
```

| Column | Required | Rule |
|--------|----------|------|
| sample_id | yes | Existing sample assigned to the uploading lab |
| test_code | yes | Exists in the consignment's rule set |
| value | numeric tests | Decimal, max 18 significant digits |
| pass_fail | qualitative tests | `pass` or `fail` |
| unit | numeric tests | Rule unit or convertible by the rule set |
| method | yes | Free text, max 100 chars |
| report_document_id | yes | Document already uploaded for this consignment |

## Validation

- The whole file is validated before anything is submitted.
- Any invalid row rejects the whole file. Response lists `{row, column, code, message}` for
  every problem, in the user's language.
- Max 5,000 rows and 5 MB per file.
- Duplicate `(sample_id, test_code)` rows inside one file are an error.
