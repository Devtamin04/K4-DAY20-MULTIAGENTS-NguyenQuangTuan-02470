---
name: generate-correct-log-triage-json
description: Use when converting a raw log file into the structured `errors.json` required by the Acme log‑triage conventions.
---
1. **Read the log file** – iterate line‑by‑line, using a regular expression to capture: timestamp, level, service, and message.  
2. **Collect extra lines** – any lines following a log entry that do not match the entry pattern belong to that entry (e.g., traceback).  
3. **Detect repeat lines** – if a line matches `-- last message repeated (\d+) times --`, add the captured number to the current entry’s `repeat_count`.  
4. **Filter by level** – keep only entries whose level (case‑insensitive) is `ERROR` or `CRITICAL`.  
5. **Normalize service name** – convert the captured service to lower case and replace hyphens (`-`) with underscores (`_`).  
6. **Convert timestamp to UTC** – parse the original ISO‑8601 timestamp (with its offset), convert to UTC, and format as `YYYY‑MM‑DDTHH:MM:SSZ`.  
7. **Upper‑case level** – store the level string in upper case (`ERROR` or `CRITICAL`).  
8. **Extract message** – use the text after `<service>: ` on the first line of the entry as the `message` field.  
9. **Capture exception** – if extra lines contain a traceback, set `exception` to the last non‑empty line of that block; otherwise set it to `null`.  
10. **Assemble entry dict** with keys: `timestamp_utc`, `service`, `level`, `message`, `exception`, `repeat_count`.  
11. **Build `errors` list** – after processing all entries, sort the list first by `service` (lexicographically) and then by `timestamp_utc` (ascending).  
12. **Compute `counts_by_service`** – sum `repeat_count` for each distinct service and store the result in a dictionary.  
13. **Create top‑level JSON object** – include:  
    - `"schema_version": 2`  
    - `"generated_by": "log-triage"`  
    - `"errors": [...]` (the sorted list)  
    - `"counts_by_service": {...}`  
14. **Write to `workspace/errors.json`** – use `json.dump(..., indent=2, sort_keys=False)` to preserve ordering.  
15. **Self‑check** – open the generated file and verify: (a) service names are lower‑case with underscores, (b) `errors` is sorted correctly, (c) `schema_version` equals 2 and `generated_by` equals `"log-triage"`, (d) `repeat_count` reflects any “repeated” lines, and (e) `exception` is `null` when no traceback is present.
