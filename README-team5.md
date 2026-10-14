# Team 5 – Error Handling, Logging and MUnit (v2)

Works with the two apps: Experience API (8081) and Process API (8082).

## Files
| File | Copy to |
|---|---|
| `global-error-handler.xml` | `src/main/mule/` in BOTH projects |
| `experience-munit-suite.xml` | `src/test/munit/` in the Experience project |
| `process-munit-suite.xml` | `src/test/munit/` in the Process project |

Add the **VM** module to both projects (Mule Palette > Search in Exchange > VM). Only one `<configuration>` element is allowed per app.
Process project also needs the Database connector and the MySQL driver in `pom.xml`.

## Changes needed in Process-API.xml (3 small edits)
1. Listener: return the status set by the error handler.
```xml
<http:listener doc:name="Receive Inventory Event" config-ref="HTTP_Listener_config" path="/api/v1/inventory/process">
    <http:response statusCode="#[vars.httpStatus default 200]" />
    <http:error-response statusCode="#[vars.httpStatus default 500]">
        <http:body>#[payload]</http:body>
    </http:error-response>
</http:listener>
```
2. Last step of `inventory-processingFlow`, before `</flow>` (logs one METRIC line per event and an ALERT_CANDIDATE line for stale/conflict):
```xml
<flow-ref name="log-event-outcome-subflow" doc:name="Log Event Outcome" />
```
3. Remove `mimeType="application/json"` from the `Return Update Success` Set Payload. A DataWeave object with a JSON mime type fails at runtime ("requires a Stream Compatible content").

## How errors work
- Business errors: `raise-error type="INVENTORY:INVALID_EVENT"` (also `UNKNOWN_MOVEMENT`, `UNKNOWN_REFERENCE`, `NEGATIVE_AVAILABLE`, `VERSION_CONFLICT`, `STALE_EVENT`, `DUPLICATE_EVENT`).
  Response follows the RAML (`ApiError`). For a single event inside the Process API the HTTP status is 200 and `status` is `REJECTED` / `CONFLICT` / `STALE` / `DUPLICATE`.
- System errors (DB, HTTP, queue, unexpected): logged at ERROR, original payload and reason sent to the DLQ (`inventory-dlq`, visible as `DLQ_MESSAGE` in the log), caller gets 503 `RETRY_PENDING` or 500 `FAILED` with no internal details.
- Retry: wrap DB and queue calls in `<until-successful maxRetries="${retry.maxAttempts}" millisBetweenRetries="${retry.delayMs}">`. When retries run out the handler returns 503 and sends the event to the DLQ.
- Logs: `ERROR | ...`, `REJECTED | ...`, `DLQ | ...`, `METRIC | ...`, `ALERT_CANDIDATE | ...`, always with correlationId.

## Known gaps in the current flows (not Team 5)
- Experience API sends the whole batch result, Process API expects one `{event, sourceSystem}`.
- `expectedVersion` is not in the first update payload, so the first update never matches. New SKUs are never inserted.
- No audit insert, queue publish, low-stock flag, master data check or DB transaction yet.
- Retry loop after a version conflict is not implemented (`Return Version Conflict` cannot be reached).
- DB host, user and password are in plain text in `Process-API.xml`. Move them to secure properties and rotate the password.
