---
name: sfdc-data360migrate-noabstract
author: Chandra Nayak/Snowflake
description: "Migrate Salesforce V1 connector to V2 connector without abstracted views. Mode A: swap database names. Mode B: drop existing and recreate. Trigger: dbswap-noabstract"
---

# Migrate SFSHARES — No Abstraction Layer

## Overview
This skill guides customers through migrating from the Salesforce V1 connector database to the Salesforce V2 connector database when there is **no abstraction layer** (no views pointing to the old database). Two modes are supported:

- **Mode A** — Swap database names (rename V2 to V1's name, rename V1 to a backup name). Downstream objects continue to work without changes.
- **Mode B** — Drop the existing V1 database and recreate/replace it with V2. Used when a clean cutover is preferred and there is no need to preserve the old database.

## Workflow

### Step 1: Confirm Prerequisites
Before proceeding, confirm with the customer:
1. There are **no abstracted views** or other objects pointing to the V1 database (use `SHOW VIEWS` and search for references).
2. The V2 connector database exists and is fully synced.
3. A maintenance window or low-traffic period has been agreed upon.
4. Downstream consumers (BI tools, pipelines, apps) have been identified.

Ask the customer which mode they want:
- **Mode A** if they want minimal downstream disruption (name swap).
- **Mode B** if they want a clean drop-and-recreate.

### Step 2: Identify Databases and Schema

#### Single-Pair vs. Multi-Pair Swap

First, ask the customer whether they need to swap **one** database pair or **multiple** pairs.

> "Are you swapping a single database pair, or do you need to swap multiple databases at once?"

**If the customer wants to swap more than 2 databases (multiple pairs):**

Prompt the customer to provide their database pairs in `(old, new)` format — one pair per line or comma-separated. Each pair represents an old database to be retired and the new database that will take its name.

> "Please list your database pairs in `(old, new)` format, where `old` is the existing database name to retire and `new` is the replacement database. For example:
> ```
> (L0_V1, L0_V2)
> (ANALYTICS_V1, ANALYTICS_V2)
> (REPORTING_OLD, REPORTING_NEW)
> ```
> Each pair will be swapped independently: `old` gets renamed to `old_BACKUP`, and `new` gets renamed to `old`."

Parse the input into a list of pairs. Store as:
- `$PAIRS` = list of `(OLD_DB, NEW_DB)` tuples
- For each pair, derive `$BACKUP_DB` = `<OLD_DB>_BACKUP` (or ask the customer for custom backup names)

Then ask for the schema to validate (can be the same across all pairs or specified per pair):

> "Which schema should be validated after the swap? You can specify one schema for all databases, or provide one per pair in the same order."

Execute **Step 3a (or 3b)** sequentially for each pair — completing all phases (grant capture → rename → grant re-apply) for one pair before moving to the next. Log results per pair so any failure is isolated.

**If the customer wants to swap a single pair (default):**

**Ask the customer for:**

1. V1 database name (the existing Salesforce V1 connector database being retired)
2. V2 database name (the new Salesforce V2 connector database)
3. Backup database name (holding name for old V1 after swap — Mode A only)
4. **Schema name to validate after swap** — the schema within the database that contains the Salesforce objects to be smoke-tested post-migration (e.g., `SALESFORCE`, `PUBLIC`, or a quoted case-sensitive name like `"schema_DEMO_SHARE"`)

Store as variables: `$V1_DB`, `$V2_DB`, `$BACKUP_DB`, `$SCHEMA`

Defaults:
- `V1_DB` = `L0_V1`
- `V2_DB` = `L0_V2`
- `BACKUP_DB` = `L0_V1_BACKUP`
- `SCHEMA` = `SALESFORCE`

If the customer uses different names, substitute accordingly throughout.

### Step 3a: Execute Mode A — Database Name Swap with Grant Mirroring

#### Phase 0: Determine Swap Method

Before executing the swap, determine the **database types** for the old and new databases:

> "Is the old database (`$V1_DB`) created from a **share** (imported database), and the new database (`$V2_DB`) a **catalog-linked database (CLD)**?"

**If YES — old DB is from a share and new DB is a CLD:**

Use the zero-copy swap function instead of manual renames. This atomically swaps the imported database with the catalog-linked database in a single operation:

```sql
SELECT SYSTEM$ZEROCOPY_SWAP_IMPORTED_DB_WITH_CLD('$V1_DB', '$V2_DB');
```

This function:
- Swaps the two databases atomically (the CLD takes the share DB's name and vice versa)
- Preserves grants on the original database name
- Does not require a backup rename step
- Is the recommended approach when migrating from a Salesforce V1 share to a V2 catalog-linked database

After the swap completes, skip directly to **Step 4: Validate** — no manual grant re-application is needed since grants are preserved on the database name.

**If NO — both are standard databases:**

Proceed with the manual rename approach below (Phase 1 → Phase 3).

#### Phase 1: Capture Grants on $V1_DB Before the Swap
Run this **before** any renaming. Save the output — it will be used to re-apply grants after the swap.

Also capture schema-level grants on `$SCHEMA`:

```sql
-- Capture all grants currently on $V1_DB
SHOW GRANTS ON DATABASE $V1_DB;

-- Capture grants on the schema to validate
SHOW GRANTS ON SCHEMA $V1_DB.$SCHEMA;
```

Alternatively, query the account usage view for a persistent record:

```sql
SELECT
    grantee_name,
    privilege,
    granted_on,
    name AS object_name,
    granted_by
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE object_type = 'DATABASE'
  AND name = 'L0_V1'
  AND deleted_on IS NULL
ORDER BY grantee_name;
```

Save this result set — it defines all role-level privileges to re-apply after swap.

#### Phase 2: Perform the Name Swap
Run the following in order. Verify each step succeeds before proceeding.

```sql
-- Step 1: Move L0_V1 out of the way
ALTER DATABASE L0_V1 RENAME TO L0_V1_BACKUP;

-- Step 2: Promote L0_V2 into L0_V1's position
ALTER DATABASE L0_V2 RENAME TO L0_V1;

-- Step 3: Verify both names exist as expected
SHOW DATABASES LIKE 'L0_V1';
SHOW DATABASES LIKE 'L0_V1_BACKUP';
```

#### Phase 3: Re-Apply Grants to New L0_V1
For each row captured in Phase 1, issue the corresponding `GRANT` statement on the new `L0_V1` database.

Generate the grant statements automatically:

```sql
-- Generate GRANT DDL from the saved grant records
SELECT
    'GRANT ' || privilege || ' ON DATABASE L0_V1 TO ROLE ' || grantee_name || ';' AS grant_statement
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE object_type = 'DATABASE'
  AND name = 'L0_V1_BACKUP'   -- query the backup — it still holds original grant history
  AND deleted_on IS NULL
ORDER BY grantee_name;
```

Copy the output and execute each `GRANT` statement against the new `L0_V1`. Then verify:

```sql
-- Confirm grants are in place on the new L0_V1
SHOW GRANTS ON DATABASE L0_V1;
```

Cross-check that every role from Phase 1 appears in the output.

After the swap:
- Downstream objects using `L0_V1` now point to the V2 connector data automatically.
- Original V1 data and grant history are preserved under `L0_V1_BACKUP` for rollback.

### Step 3b: Execute Mode B — Drop and Recreate

#### Phase 1: Ask for the V1 Database Name
Ask the customer:

> "What is the exact name of your Salesforce V1 connector database that you want to drop and replace? (e.g., `L0_V1`)"

Store this as `<L0_V1DB>`. All commands below use this name.

#### Phase 2: Capture and Save All Grants
Run the following to capture all grants on `<L0_V1DB>`. This must complete before any destructive action.

```sql
-- Show all grants on the V1 database
SHOW GRANTS ON DATABASE <L0_V1DB>;
```

For a persistent, scriptable copy generate the full grant DDL:

```sql
-- Generate ready-to-execute GRANT statements
SELECT
    'GRANT ' || privilege || ' ON DATABASE <L0_V1DB> TO ROLE ' || grantee_name || ';' AS grant_statement
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE object_type = 'DATABASE'
  AND name = '<L0_V1DB>'
  AND deleted_on IS NULL
ORDER BY grantee_name;
```

Save the output of the above query as a SQL script (e.g., `l0_v1db_grants.sql`). This script will be re-executed after the new database is created.

Optionally, store the grants in a temporary table for easier replay:

```sql
CREATE OR REPLACE TEMPORARY TABLE GRANT_BACKUP_L0_V1DB AS
SELECT
    grantee_name,
    privilege,
    'GRANT ' || privilege || ' ON DATABASE <L0_V1DB> TO ROLE ' || grantee_name || ';' AS grant_statement
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE object_type = 'DATABASE'
  AND name = '<L0_V1DB>'
  AND deleted_on IS NULL
ORDER BY grantee_name;

-- Confirm rows were captured
SELECT COUNT(*) AS grants_captured FROM GRANT_BACKUP_L0_V1DB;
```

**Stop here and confirm with the customer:**

> "I have captured [N] grants for `<L0_V1DB>` and saved them. Please review the grant list above and confirm you are ready to proceed with dropping the database. Reply **Yes** to continue or **No** to abort."

Do not proceed until the customer explicitly confirms.

#### Phase 3: Drop the V1 Database
Only after the customer confirms grant capture is complete:

> **Warning**: `DROP DATABASE` is irreversible. There is no undo. Proceed only after the customer has confirmed in the previous step.

```sql
DROP DATABASE <L0_V1DB>;

-- Confirm it no longer exists
SHOW DATABASES LIKE '<L0_V1DB>';
-- Expected: 0 rows returned
```

Inform the customer:

> "`<L0_V1DB>` has been dropped successfully. You can now go ahead and create the new database using the Salesforce V2 connector, using the **exact same name**: `<L0_V1DB>`."

#### Phase 4: Wait for Customer to Create the New Database
Prompt the customer:

> "Have you created the new Salesforce V2 connector database with the name `<L0_V1DB>`? Reply **Yes** when it is ready and I will apply the saved grants."

Do not run any further SQL until the customer replies **Yes**.

Once confirmed, verify the new database exists:

```sql
SHOW DATABASES LIKE '<L0_V1DB>';
-- Expected: 1 row with the new database
```

#### Phase 5: Re-Apply Grants to New Database
With the new `<L0_V1DB>` in place, replay the saved grant statements.

If using the temporary table from Phase 2:

```sql
-- Review the grants about to be applied
SELECT grant_statement FROM GRANT_BACKUP_L0_V1DB ORDER BY grantee_name;
```

Execute each row as a SQL statement. Then verify all grants are applied:

```sql
-- Confirm grants on the new database
SHOW GRANTS ON DATABASE <L0_V1DB>;
```

Cross-check that the role list matches what was captured in Phase 2. If any grants are missing, re-run the specific `GRANT` statement from the backup.

Inform the customer:

> "All grants have been re-applied to `<L0_V1DB>`. The migration is complete. Please proceed to the validation step to confirm data access."

### Step 4: Validate — Replay Historical Queries from Past 24 Hours

#### Phase 0: Request Validation Role
Before running any validation SQL, ask the customer which role should be used.

> **Ask the customer:**
> "Which Snowflake role should be used to run the validation queries? This role must have `USAGE` on `L0_V1` and `SELECT` privileges on the Salesforce schemas."

Once the customer provides the role, set it at the start of the validation session:

```sql
USE ROLE <CUSTOMER_PROVIDED_ROLE>;
```

Do not proceed to Phase 1 until the role is confirmed and set. If the customer is unsure, suggest querying the roles that currently have access to the new `L0_V1`:

```sql
-- Help the customer pick a role: show roles with USAGE on the new L0_V1
SELECT grantee_name AS role_name, privilege
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE object_type = 'DATABASE'
  AND name = 'L0_V1'
  AND deleted_on IS NULL
ORDER BY grantee_name;
```

#### Phase 1: Identify Recent Queries Against L0_V1
Pull queries that ran against the old database in the past 24 hours from query history:

```sql
SELECT
    query_id,
    query_text,
    user_name,
    role_name,
    warehouse_name,
    start_time,
    execution_status,
    rows_produced,
    total_elapsed_time
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD('hour', -24, CURRENT_TIMESTAMP())
  AND UPPER(query_text) LIKE '%L0_V1%'
  AND execution_status = 'SUCCESS'
ORDER BY start_time DESC
LIMIT 50;
```

#### Phase 2: Re-Run Representative Queries Against New $V1_DB
Select a representative sample (at minimum: one read query per major Salesforce object). Run each against the new `$V1_DB` using the schema name (`$SCHEMA`) provided in Step 2, and compare `rows_produced` to the historical baseline.

```sql
-- Example: re-run a spot-check on core objects using $SCHEMA
SELECT COUNT(*) AS account_count   FROM $V1_DB.$SCHEMA.ACCOUNT;
SELECT COUNT(*) AS opportunity_count FROM $V1_DB.$SCHEMA.OPPORTUNITY;
SELECT COUNT(*) AS contact_count   FROM $V1_DB.$SCHEMA.CONTACT;
SELECT COUNT(*) AS lead_count      FROM $V1_DB.$SCHEMA.LEAD;
```

If the schema name is case-sensitive (quoted identifier), use double quotes: `$V1_DB."schema_name".TABLE`.

Compare row counts to what was returned by the same queries before the swap. Row counts should be equal to or greater than (V2 may have ingested more recent data).

Additionally, list all objects in the validated schema to confirm they are accessible:

```sql
SELECT table_name, table_type
FROM $V1_DB.INFORMATION_SCHEMA.TABLES
WHERE table_schema = '$SCHEMA'
ORDER BY table_name;
```

#### Phase 3: Check for Errors After Swap
Look for any query failures referencing `L0_V1` in the 30 minutes after the swap completed:

```sql
SELECT
    query_id,
    query_text,
    user_name,
    start_time,
    error_message
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD('minute', -30, CURRENT_TIMESTAMP())
  AND UPPER(query_text) LIKE '%L0_V1%'
  AND execution_status != 'SUCCESS'
ORDER BY start_time DESC;
```

If any failures are returned, inspect `error_message` to determine whether they are swap-related (missing schemas, missing tables) or pre-existing issues.

#### Phase 4: Validate Object Dependencies

Query `SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES` to find all objects that reference the swapped database. Then verify each referencing object can still access its referenced objects.

**Step 1: Identify all dependencies on $V1_DB**

```sql
SELECT
    referencing_object_domain,
    referencing_database || '.' || referencing_schema || '.' || referencing_object_name AS referencing_object,
    referenced_object_domain,
    referenced_database || '.' || referenced_schema || '.' || referenced_object_name AS referenced_object
FROM SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES
WHERE referenced_database = '$V1_DB'
ORDER BY referencing_object;
```

This returns all views, procedures, tasks, streams, and other objects that depend on tables/views in the swapped database.

**Step 2: Validate each referencing object can resolve its dependencies**

For each referencing object returned above, run a compilation check to confirm it can still access the referenced objects. The approach depends on the object type:

- **Views**: Run `SELECT * FROM <referencing_object> LIMIT 0;` — if it compiles without error, the dependency is intact.
- **Procedures/Functions**: Run `DESCRIBE PROCEDURE/FUNCTION <referencing_object>(...);` to confirm it resolves.
- **Tasks**: Run `DESCRIBE TASK <referencing_object>;` and check for errors.
- **Streams**: Run `DESCRIBE STREAM <referencing_object>;` and verify `stale = false`.

Generate and execute the validation queries automatically:

```sql
-- Generate validation queries for all VIEW dependencies
SELECT
    referencing_database || '.' || referencing_schema || '.' || referencing_object_name AS referencing_view,
    'SELECT * FROM ' || referencing_database || '.' || referencing_schema || '.' || referencing_object_name || ' LIMIT 0;' AS validation_query
FROM SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES
WHERE referenced_database = '$V1_DB'
  AND referencing_object_domain = 'VIEW'
ORDER BY referencing_view;
```

Execute each `validation_query`. If any query fails with an error like `Object does not exist` or `View definition is invalid`, report it to the customer as a broken dependency that needs remediation.

**Step 3: Summarize results**

Present a table to the customer showing each referencing object and whether validation passed or failed:

| Referencing Object | Type | Referenced Object | Status |
|----|----|----|------|
| `DB.SCHEMA.MY_VIEW` | VIEW | `$V1_DB.$SCHEMA.ACCOUNT` | PASS |
| `DB.SCHEMA.BAD_VIEW` | VIEW | `$V1_DB.$SCHEMA.DELETED_TABLE` | FAIL — Object does not exist |

If all dependencies pass, inform the customer:

> "All object dependencies on `$V1_DB` have been validated. Every referencing object can successfully access its referenced objects."

If any dependencies fail, list the broken objects and recommend remediation (e.g., recreate the view, update the reference, or confirm the object was intentionally removed in V2).

Ask the customer to validate their BI tools and downstream pipelines after confirming the above checks pass.

### Step 5: Post-Migration Cleanup (Mode A)
Once validation passes, ask the customer whether they want to drop the backup database.

**MANDATORY STOPPING POINT** — Ask the customer:

> "All validation checks have passed. Would you like to drop the backup database `$BACKUP_DB`? This is irreversible. Reply **Yes** to drop it, or **No** to keep it for now (you can drop it later manually with `DROP DATABASE $BACKUP_DB;`)."

Do **not** execute the drop until the customer explicitly confirms **Yes**.

**If the customer confirms Yes:**

```sql
DROP DATABASE $BACKUP_DB;

-- Verify it no longer exists
SHOW DATABASES LIKE '$BACKUP_DB';
```

**If the customer declines:** Leave `$BACKUP_DB` in place. Inform the customer they can drop it later when ready:

> "`$BACKUP_DB` has been preserved. You can drop it at any time with: `DROP DATABASE $BACKUP_DB;`"

## Examples

**Example — Mode A (l0_v1 / l0_v2)**

```sql
-- 1. Capture grants
SHOW GRANTS ON DATABASE L0_V1;

-- 2. Swap names
ALTER DATABASE L0_V1 RENAME TO L0_V1_BACKUP;
ALTER DATABASE L0_V2 RENAME TO L0_V1;

-- 3. Re-apply grants (generated from Phase 1 output)
GRANT USAGE ON DATABASE L0_V1 TO ROLE SYSADMIN;
-- ... (repeat for each role captured)

-- 4. Validate row counts
SELECT COUNT(*) FROM L0_V1.SALESFORCE.ACCOUNT;

-- 5. Check for post-swap errors
SELECT query_id, error_message
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD('minute', -30, CURRENT_TIMESTAMP())
  AND UPPER(query_text) LIKE '%L0_V1%'
  AND execution_status != 'SUCCESS';
```

**Example — Mode B**
- V1_DB: `SALESFORCE`
- V2_DB: `SALESFORCE_V2`

```sql
ALTER DATABASE SALESFORCE_V2 RENAME TO SALESFORCE_NEW;
DROP DATABASE SALESFORCE;
ALTER DATABASE SALESFORCE_NEW RENAME TO SALESFORCE;
```

**Example — Mode A Multi-Pair Swap**

Customer input:
```
(L0_V1, L0_V2)
(ANALYTICS_V1, ANALYTICS_V2)
```

Execution (sequential, one pair at a time):

```sql
-- Pair 1: (L0_V1, L0_V2)
SHOW GRANTS ON DATABASE L0_V1;
ALTER DATABASE L0_V1 RENAME TO L0_V1_BACKUP;
ALTER DATABASE L0_V2 RENAME TO L0_V1;
-- Re-apply grants to new L0_V1...
SHOW GRANTS ON DATABASE L0_V1;

-- Pair 2: (ANALYTICS_V1, ANALYTICS_V2)
SHOW GRANTS ON DATABASE ANALYTICS_V1;
ALTER DATABASE ANALYTICS_V1 RENAME TO ANALYTICS_V1_BACKUP;
ALTER DATABASE ANALYTICS_V2 RENAME TO ANALYTICS_V1;
-- Re-apply grants to new ANALYTICS_V1...
SHOW GRANTS ON DATABASE ANALYTICS_V1;
```

Validation runs against all swapped databases before cleanup.

**Example — Mode A Zero-Copy Swap (Share → CLD)**

When the old database is from a share and the new database is a catalog-linked database:

```sql
-- Single atomic swap — no rename or grant re-application needed
SELECT SYSTEM$ZEROCOPY_SWAP_IMPORTED_DB_WITH_CLD('L0_V1', 'L0_V2');

-- Validate
SELECT COUNT(*) FROM L0_V1.SALESFORCE.ACCOUNT;
```

## When to Apply
- Customer is migrating from Salesforce Connector V1 to V2
- There is **no abstraction layer** (no views or objects pointing to V1 by name that would need updating)
- Customer triggers with: `/dbswap-noabstract` or mentions `dbswap-noabstract`
- Customer wants either a name swap (Mode A) or a clean drop-and-recreate (Mode B)

## Related Skills
- `dbswitch` — for Salesforce V1→V2 migration with abstracted views (managed redirect of dependent objects)
- `database-swap` — general-purpose database name swap with grant mirroring
