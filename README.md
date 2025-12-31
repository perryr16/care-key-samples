# Care Key Test CSV Files

## Checking the logs
In the GCP logs explorer its highly recommended you set the Summary fields (preferences > manage summary fields) to: `resource.labels.configuration_name` and `jsonPayload.msg` 

<img width="603" height="77" alt="Screenshot 2025-12-30 at 7 58 44 PM" src="https://github.com/user-attachments/assets/93ac04bb-fe35-4121-9f90-a5b564302a52" />

Look for the care-key-processor `jsonPayload.msg` "CSV: Batch processed successfully
" or "CSV: Batch processed with rejected rows"

## Checking results
After each run, check the DB for results 
`SELECT * FROM care_keys ORDER BY row_id;`

## Running tests
Replace the final url segment with the file name (01-happy.csv -> 02-duplicates.csv)
```bash
curl -X POST "{{ PRODUCTION URL }}" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{ INSERT BEARER }}" \
  -d '{
    "version":"0",
    ...
    "detail":{
      "artifacts":[
        {
          "usage":"input",
          "url":"https://raw.githubusercontent.com/perryr16/care-key-samples/refs/heads/main/01-happy.csv"
        }
      ]
    }
  }'
```


### 01. Happy Path - Creates
- File: `01-happy.csv`
- Purpose: Basic create operations
- Expected: All 3 records created (1001, 1002, 1003)

---

### 02. Duplicates
- File: `02-duplicates.csv`
- Purpose: Idempotency handling
- Expected: First 3 ignored (duplicates), row 1004 created

---

### 03. Conflicts
- File: `03-conflicts.csv`
- Purpose: Duplicate vs conflict detection
- Expected:
  - First 1001: Ignored (duplicate, same content)
  - Second 1001: Rejected (conflict - same version, different content)
  - Third 1001: Accepted (member_id → "MEM001_UPDATED")

---

### 04. Updates and Deletes
- File: `04-updates-and-deletes.csv`
- Purpose: UPDATE and DELETE operations
- Expected:
  - 1001: Updated (member_id - MEM001-UPDATED, benefit_codes changed)
  - 1002: Deleted
  - 1003: Updated (all business fields blanked out)
  - 1004: Rejected (already exists)

---

### 05. Version Ordering
- File: `05-version-ordering.csv`
- Purpose: Version-based ordering
- Expected:
  - 2001: Final state v3 (member_id → "Third Name")
  - 2002: Created (v1) then Deleted (v2)

---

### 06. Validation Errors
- File: `06-validation-errors.csv`
- Purpose: Schema validation
- Expected:
  - Rejected: Rows 1-5, 7 (missing required fields, invalid op, bad timestamp)
  - Accepted: Row 6 (3006)

---

### 07. Idempotency and Duplicates
- File: `07-idempotency-duplicates.csv`
- Purpose: Duplicate detection via checksums
- Expected:
  - 4001: Created once (duplicate ignored)
  - 4002: Created then updated once (duplicate update ignored)
  - 4003: Created once (second CREATE ignored)

---

### 08. Edge Cases
- File: `08-edge-cases.csv`
- Purpose: Nullable fields and edge data
- Expected:
  - Rejected: 5002, 5003 (missing NOT NULL fields)
  - Accepted: 5001, 5004 (empty benefit code ok?), 5005, 5006, 5007 (nullable empties, arrays, special chars)
  - Silent failures: 5008, 5009 (operations on non-existent records)

---

### 09. Out-of-Order Delivery
- File: `09-out-of-order-delivery.csv`
- Purpose: Sorting by id → version → changed_at
- Expected:
  - 6001: Final state v3 (member_id → "Third Update")
  - 6002: Created (v1) then Deleted (v2)
  - 6003: Created (v1) then Updated (v2, member_id → "Updated Name")

---
