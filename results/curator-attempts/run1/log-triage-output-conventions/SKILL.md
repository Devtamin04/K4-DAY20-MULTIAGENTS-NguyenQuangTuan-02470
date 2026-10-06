---
name: log-triage-output-conventions
description: Use when you transform raw log files into the structured `errors.json` report so that service names, sorting, and schema metadata obey the Acme log‑triage standards.
---
1. Parse each line of the log with a regex that captures `timestamp`, `level`, `service`, and `message`.  
2. Treat level names case‑insensitively; keep only entries whose level is `ERROR` or `CRITICAL`.  
3. Convert the captured timestamp (which may include an offset) to UTC and format it as `YYYY‑MM‑DDTHH:MM:SSZ`.  
4. Normalise the service name:  
   a. Lower‑case the entire string.  
   b. Replace any hyphens (`-`) with underscores (`_`).  
5. Extract the message text that follows `<service>: ` on the first line of the entry.  
6. Gather any following lines that are not a new log entry as “extra” lines:  
   a. If the last extra line looks like a traceback exception (e.g., `ValueError: …`), store it as the `exception` field; otherwise set `exception` to `null`.  
7. Detect a repeat line of the form `-- last message repeated N times --` immediately after the entry (or its extra lines) and compute `repeat_count = 1 + Σ N`.  
8. Build an error object with keys:  
   - `timestamp_utc` (UTC string)  
   - `service` (normalised name)  
   - `level` (upper‑case)  
   - `message`  
   - `exception` (string or `null`)  
   - `repeat_count`  
9. Append each error object to a list.  
10. After processing all lines, sort the `errors` list:  
    a. Primary key: `service` alphabetically.  
    b. Secondary key: `timestamp_utc` ascending.  
11. Compute `counts_by_service` by summing `repeat_count` for each distinct service.  
12. Assemble the final JSON object with the required top‑level fields:  
    ```json
    {
      "schema_version": 2,
      "generated_by": "log-triage",
      "errors": [ …sorted list… ],
      "counts_by_service": { … }
    }
    ```  
13. Write the object to `workspace/errors.json` using `json.dump(..., indent=2)`.  
14. Validate the file against any provided JSON schema; ensure no validation errors.  
15. **Self‑check**: service names are lower‑case with underscores, the `errors` array is sorted correctly, and the top‑level object contains `schema_version: 2` and `generated_by: "log‑triage"`. If any condition fails, return to the relevant step.
