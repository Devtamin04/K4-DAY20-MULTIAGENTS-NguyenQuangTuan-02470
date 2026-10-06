---
name: generate-log-triage-json
description: Use when extracting error entries from logs and writing errors.json to satisfy Acme log‑triage conventions.
---
1. **Read the log file** (`app.log`) line‑by‑line.

2. **Identify entry lines** with the regex:  
   `^(?P<ts>\S+) \[(?P<lvl>[^\]]+)\] (?P<svc>[^:]+): (?P<msg>.*)$`

3. **Collect extra lines** (tracebacks, etc.) until the next entry line or a repeat‑line.

4. **Detect repeat lines** with the regex:  
   `^-- last message repeated (\d+) times --$`  
   - Initialise `repeat_count = 1`.  
   - For each matching repeat line that follows the current entry, add the captured number to `repeat_count`.

5. **Filter by level**:  
   - Convert `lvl` to lower case.  
   - Keep the entry only if it is `error` or `critical`.

6. **Transform fields**:  
   - **timestamp_utc**: parse `ts` (`datetime.fromisoformat`), convert to UTC (`astimezone(timezone.utc)`), format `YYYY‑MM‑DDTHH:MM:SSZ`.  
   - **level**: upper‑case (`ERROR` or `CRITICAL`).  
   - **service**: lower‑case and replace hyphens with underscores (`payment-service` → `payment_service`).  
   - **message**: the text captured as `msg` (everything after the first colon).  
   - **exception**: if extra lines contain a traceback, take the **last non‑empty line**; otherwise set to `null`.

7. **Create the error object**:  
   ```json
   {
     "timestamp_utc": "<formatted>",
     "service": "<service_name>",
     "level": "<LEVEL>",
     "message": "<message>",
     "exception": "<exception line or null>",
     "repeat_count": <int>
   }
   ```

8. **Accumulate all error objects** into a list `errors`.

9. **Sort `errors`**:  
   - Primary key: `service` (alphabetical).  
   - Secondary key: `timestamp_utc` (ascending).  

10. **Build `counts_by_service`**:  
    - For each distinct service, sum the `repeat_count` of its error objects.

11. **Assemble the top‑level JSON** with required schema fields:  
    ```json
    {
      "schema_version": 2,
      "generated_by": "log-triage",
      "errors": [ ...sorted list... ],
      "counts_by_service": { ... }
    }
    ```

12. **Write to `workspace/errors.json`** using `json.dump(..., indent=2, sort_keys=False)` to preserve ordering.

13. **Validate**:  
    - Load the file back with `json.load` and confirm the presence of `schema_version` and `generated_by`.  
    - Verify that `errors` is sorted correctly and that service names follow the lower‑case‑underscore rule.  

14. **Self‑check**:  
    - Only ERROR/CRITICAL entries are present.  
    - Timestamps are UTC‑formatted with trailing `Z`.  
    - Service names are lower‑case with underscores.  
    - `errors` list is sorted by service then timestamp.  
    - `schema_version` equals 2 and `generated_by` equals `"log-triage"`.  
    - `counts_by_service` correctly aggregates repeat counts.
