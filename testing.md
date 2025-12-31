# Care Key Test CSV Files

### 01. Happy Path - Creates
- File: `01-happy-path-creates.csv`
- Purpose: Basic functionality test with valid create operations
- Expected Result: All 3 records should be created successfully

---

### 02. Duplicates
- File: `02-duplicates.csv`
- Purpose: Duplicates ignored via idempotency
- Expected Result: First 3 records should be ignored, 1 created for row_id 1004

--


### 03. conflicts
- File: `03-conflicts.csv`
- Purpose: Duplicates ignored via idempotency
- Expected Result: 
    - Ignored (duplicate): first row_id `1001` (version `1`, same content).
    - Rejected (conflict): second row_id `1001` (version `1`, different content).
    - Accepted: third row_id `1001`. member_id is "MEM001_UPDATED"

--

### 04. Updates and Deletes
- File: `04-updates-and-deletes.csv`
- Purpose: Test UPDATE and DELETE operations on existing records
- Test Cases:
  - Row 1: Update existing record (id=1001) with new data, version=2
  - Row 2: Delete existing record (id=1002), version=2
  - Row 3: Update existing record (id=1003) with ALL empty business fields (tests "blank out" behavior), version=2
  - Row 4: Create new record (id=1004), version=1
- Expected Result:
  - Record 1001 should be updated with new member_id and benefit_codes
  - Record 1002 should be deleted
  - Record 1003 should have ALL business fields blanked out (per spec: "Empty or NULL values will be treated as 'blank out this field'")
  - Record 1004 should be rejected

---

### 05. Version Ordering
- File: `05-version-ordering.csv`
- Purpose: Test version-based conflict resolution and ordering
- Test Cases:
  - Record 2001: Three versions (1, 2, 3) with increasing timestamps
  - Record 2002: Create then delete (versions 1, 2)
- Expected Result:
  - Record 2001 should end with version 3 data ("Third Name")
  - Record 2002 should be deleted (version 2 delete wins)
  - System should apply changes in order: version → changed_at

---

### 06. Validation Errors
- File: `06-validation-errors.csv`
- Purpose: Test schema validation and error handling
- Test Cases:
  - Row 1: Missing `id` field (should reject - CDC required field)
  - Row 2: Invalid `op` value ('X' instead of C/U/D) (should reject - invalid enum)
  - Row 3: Invalid timestamp format (MM/DD/YYYY instead of ISO 8601) (should reject - format error)
  - Row 4: Missing `version` field (should reject - CDC required field)
  - Row 5: Missing `op` field (should reject - CDC required field)
  - Row 6: Valid record for comparison (should accept)
  - Row 7 (3007): CREATE with ALL business fields empty (should reject - missing NOT NULL fields: member_id, dates, benefit_codes)
- Expected Result:
  - Rows 1-5, 7 should be rejected with appropriate error messages
  - Row 6 (id=3006) should be created successfully
  - System should log all validation failures to BigQuery
  - Row 7 tests that CREATEs must have NOT NULL fields populated (unlike UPDATEs which can blank them out)
  - (6 rows rejected)

---

### 07. Idempotency and Duplicates
- File: `07-idempotency-duplicates.csv`
- Purpose: Test duplicate detection and idempotent processing
- Test Cases:
  - Rows 1-2: Exact duplicate records (same id, version, timestamp, checksum)
  - Rows 3-5: Record 4002 with versions 1 and 2, where version 2 is duplicated
  - Rows 6-7: Duplicate CREATE attempts for same id (second should be ignored)
- Expected Result:
  - Record 4001 created once (duplicate ignored)
  - Record 4002 created then updated once (duplicate update ignored)
  - Record 4003 created once (second create ignored per CDC rules)
  - Checksums prevent re-processing of identical rows

---

### 08. Edge Cases
- File: `08-edge-cases.csv`
- Purpose: Test handling of unusual data including validation failures and valid edge cases
- Test Cases:
  - Row 2 (5001): Empty `benefit_codes` field - has member_id, dates, but missing benefit_codes (should reject - NOT NULL)
  - Row 3 (5002): Empty `member_id` field - has dates, benefit_codes, but missing member_id (should reject - NOT NULL)
  - Row 4 (5003): Empty date fields - has member_id, benefit_codes, but missing dates (should reject - NOT NULL)
  - Row 5 (5004): Empty `provider_name`, `type_1_npi`, `facility_name` - has all NOT NULL fields (should accept - nullable fields)
  - Row 6 (5005): Multiple pipe-delimited benefit codes (should accept - valid array format)
  - Row 7 (5006): Special characters + empty `type_2_npi`, `tin` (should accept - optional fields)
  - Row 8 (5007): Very long field values (should accept - tests field size handling)
  - Row 9 (5008): DELETE for non-existent record (should fail silently)
  - Row 10 (5009): UPDATE for non-existent record (should fail silently)
- Expected Result:
  - success: 5001 (even though missing benefit id), 5005, 5006, 5007
  - 5001: accepted with a blank benefit code
  - 5002: 
  - Rejected (validation errors): Rows 2-4 (each missing one NOT NULL field: benefit_codes, member_id, or dates)
  - Accepted: Rows 5-8 (valid creates with nullable empty fields or special chars)
  - Silent failures: Rows 9-10 (ignored per CDC rules - operations on non-existent records)
- Key Validation: Tests that each NOT NULL field is independently validated; tests nullable fields are optional

---

### 09. Out-of-Order Delivery
- File: `09-out-of-order-delivery.csv`
- Purpose: Test correct ordering when rows arrive out of sequence
- Test Cases:
  - Record 6001: Versions 3, 1, 2 arrive in wrong order
  - Record 6002: DELETE (v2) arrives before CREATE (v1)
  - Record 6003: UPDATE (v2) arrives before CREATE (v1)
  - Record 6004: Two CREATEs with same version (v1) but different timestamps (later appears first in file)
- Expected Result:
  - System should sort by: id → version → changed_at
  - Record 6001 should end with version 3 data ("Third Update")
  - Record 6002 should be created then deleted (final state: deleted)
  - Record 6003 should be created then updated (final state: updated with v2 data)
  - Record 6004 should be created with earlier timestamp data (08:10:00Z, benefit_codes=600), later CREATE (08:11:00Z) ignored
- Key Validation: Proves the staging table sort logic works correctly regardless of row order in file - timestamp determines processing order, not file position

---
