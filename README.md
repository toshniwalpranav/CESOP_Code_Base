# CESOP Spring Boot API (DIP v2 + CESOP v4.03)

Java 17+, Spring Boot 3.3, Maven. Run: `mvn spring-boot:run`  - Test: `mvn test`

## Endpoints
| Method | Path | Purpose |
|---|---|---|
| POST | `/api/cesop/validate` | multipart `file` **or** raw `application/xml` body -> JSON `ValidationResult` |
| POST | `/api/cesop/export` | JSON (`ExportRequest`) -> DIP/CESOP XML (new CESOP100, nil CESOP102, or any correction) |
| POST | `/api/cesop/resubmission/export` | multipart `original` (XML) + `request` (JSON) -> CESOP101 correction/deletion XML |

`?skipValidation=true` on the export endpoints skips the post-generation check. Without it, an invalid result returns HTTP 422 with the validation report.
Responses carry `X-Cesop-Message-Ref-Id` and `X-Dip-Transferticket-Id` headers (store the MessageRefId; you need it to correct later).

## curl
```
curl -F file=@Valid_initial_message.xml localhost:8080/api/cesop/validate

curl -X POST localhost:8080/api/cesop/export -H "Content-Type: application/json" -d @examples/export-initial.json -o out.xml

curl localhost:8080/api/cesop/resubmission/export \
  -F "original=@Valid_initial_message.xml" \
  -F "request={\"action\":\"DELETE_PAYEE\",\"targetDocRefId\":\"660f1ecb-1764-40b8-8153-72793e2af465\"};type=application/json" -o delete.xml
```

## Resubmission actions (mirror the BZSt samples)
- `CORRECT_PSP` + `reportingPsp` -> CESOP101, no payees (sample 4)
- `DELETE_PAYEE` + `targetDocRefId` -> payee copied without transactions, DocTypeIndic CESOP3 (sample 5)
- `CORRECT_PAYEE` + `targetDocRefId` + either full `payee` or `transactionPatches` e.g. `{"transactionIdentifier":"...","refund":true}` -> DocTypeIndic CESOP2 (sample 6; amount is auto-negated when refund=true and no amount is given)

In all cases `CorrMessageRefId` = original `MessageRefId`, `CorrDocRefId` = target payee `DocRefId`, new UUIDs are generated.

## Official XSDs
The BZSt XSDs were not part of the upload. Put them somewhere and set in `application.yml`:
```
cesop.xsd-files: [file:/opt/cesop/xsd/dip.xsd, file:/opt/cesop/xsd/cesop_PaymentData.xsd]
```
Uploaded files are then validated against them in addition to the built-in rules (XSD findings have code `XSD`).

## Limits
Built-in rules cover what the samples demonstrate, not the full EU business-rule catalogue. DOM-based, so very large files need heap. Only VATId is modelled under TAXIdentification.
