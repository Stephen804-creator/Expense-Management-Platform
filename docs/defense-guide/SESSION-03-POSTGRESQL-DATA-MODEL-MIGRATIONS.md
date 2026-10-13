# Session 3 — PostgreSQL, Data Modelling, Integrity, and Migrations

## 1. Why the database exists
The database stores records that must remain available after a browser refresh, application restart, or container replacement. This project uses PostgreSQL. The browser sends requests to Flask; Flask communicates with PostgreSQL. The browser must not connect directly to the database because database credentials and unrestricted SQL access do not belong in client-side code.

A relational database organizes information into tables, rows, and columns. A primary key identifies a row. A foreign key connects a row to a related row. Constraints prevent invalid states. SQL is the language used to define, query, and change relational data.

## 2. Main tables and relationships
The current models and migration files include:
- **users:** account identity, full name, email, password hash, timestamps, and profile/settings fields such as phone, currency, budget threshold, and notification preferences.
- **expense_categories:** shared category names and descriptions. Categories are not currently tied to a user in the model; this is related to an open authorization-policy finding.
- **expenses:** user reference, category reference, amount, description, expense date, and creation timestamp.
- **budgets:** user reference, optional category reference, amount, month, and creation timestamp.
- **monthly_expense_summary:** a summary-oriented table defined by the initial schema. A table existing does not prove that a feature uses it; inspect the actual routes/queries before claiming it drives a report.

One user can have many expenses. Each expense refers to a category. A user can have budgets for different months and categories. A budget with a category is category-specific; a budget with a NULL category is a whole-month budget. That NULL value has business meaning, so it must not be changed casually.

## 3. Primary keys and foreign keys
A primary key such as users.id gives each record a stable identifier. A foreign key such as expenses.user_id requires the referenced user to exist. The deletion rule determines what happens to child rows when a parent row is deleted.

- **CASCADE** deletes dependent rows when the referenced parent is deleted. The user-to-expenses relationship uses this behavior.
- **RESTRICT** prevents deletion while protected child references exist. The intended final category relationships use RESTRICT so categories cannot be removed while expenses or budgets still refer to them.
- **SET NULL** clears a child foreign key when the parent is deleted. It can be appropriate for optional relationships, but it is risky if NULL means something different in the business model.

The budget-category rule demonstrates why this matters. If deleting a category silently set a linked budget's category_id to NULL, a category budget could become a whole-month budget without the user changing it. Migration 004 changes the budget-category foreign key to RESTRICT, and the category route checks for references. Always inspect the full migration sequence: migration 001 is the initial schema, not necessarily the final schema after migrations 002–005.

## 4. Constraints are stronger than application checks
Application checks help produce friendly errors, but concurrent requests can race. Two requests can both query “does this budget exist?” before either inserts a row. Both can then attempt the insert. A database constraint is the final protection for the stored state.

Examples:
- User email uniqueness prevents duplicate email values.
- Category names have a uniqueness rule.
- Expense and budget amounts must be positive.
- Budget month values must use the first day of a month.
- Budget uniqueness is enforced with separate partial unique indexes for category-specific and whole-month budgets.

PostgreSQL ordinarily permits multiple NULL values under a standard unique constraint. Since a whole-month budget uses NULL for category_id, a single unique index on user_id, month, category_id would not reliably enforce one whole-month budget per user/month. Migration 003 uses two partial unique indexes: one for non-NULL category IDs and one for NULL category IDs.

A partial index covers rows that match a condition. Here it expresses two related but distinct uniqueness rules.

## 5. Money and dates
Amounts use PostgreSQL NUMERIC(12,2), a fixed-precision decimal representation appropriate for currency. The backend validates that amounts are finite, positive, within supported range, and have at most two decimal places. A database check constraint also prevents non-positive values from entering through another code path.

Dates are stored as SQL DATE values rather than arbitrary text. Expense dates represent calendar dates. Budget month values are normalized to the first day, such as 2026-10-01. This gives each month a canonical representation.

NUMERIC does not automatically eliminate every money bug: calculations must still use consistent rules, appropriate rounding, and the correct user/date scope. DATE does not automatically prevent frontend timezone mistakes if the browser converts a date-only value as a timestamp.

## 6. Migrations and the current sequence
A migration is a versioned change to the database schema or data. The repository has ordered SQL files under database/migrations:
- 001_initial_schema.sql: core tables and indexes.
- 002_user_settings.sql: user settings columns.
- 003_budget_uniqueness.sql: duplicate-data checks and partial unique indexes.
- 004_restrict_category_budget_delete.sql: category-budget deletion integrity.
- 005_user_phone.sql: phone column.

Do not claim a migration has run on AWS because its file exists in Git. The CI database job applies migrations and checks selected schema properties. The deployment workflow explicitly applies migrations 002–005 to an existing volume.

PostgreSQL processes files in docker-entrypoint-initdb.d only when initializing an empty data directory. If a named volume already exists, adding a new SQL file does not make PostgreSQL run it on restart. Existing data therefore needs an explicit migration process.

## 7. What if a migration fails?
Existing data can conflict with a new rule. Migration 003 checks for duplicate category-specific or whole-month budgets before adding unique indexes. If duplicates exist, it should stop and require the team to resolve them intentionally rather than silently deleting financial records.

A safe migration procedure:
1. Identify the exact commit and intended schema change.
2. Confirm a restorable backup exists.
3. Check whether existing data violates the new constraint.
4. Apply the migration with fail-on-error behavior.
5. Inspect the resulting columns, indexes, and foreign-key rules.
6. Restart the new application only after migration success.
7. Run database/application health checks and functional smoke tests.
8. Retain logs and a recovery plan.

A code rollback does not automatically reverse a database change. Some migration statements may already have committed, and the old app may not work against a partially changed schema. Prefer backward-compatible migrations and test recovery deliberately.

## 8. Connection configuration and least privilege
The backend assembles a PostgreSQL connection using environment variables. Inside Compose, the database hostname is the service name postgres, not localhost. Inside a container, localhost refers to that same container.

Production should avoid using the PostgreSQL bootstrap/superuser account for ordinary application queries. A separate application role should receive only the permissions it needs. The QA audit has recorded a finding about the backend using the bootstrap database user; check the current correction branch and deployed configuration before saying the issue is fixed.

Credentials belong in environment/secrets configuration, not source code. Example files should contain placeholders only. If a real credential is exposed, rotate it and assess the historical exposure rather than merely deleting the latest copy.

## 9. How the database connects to the application
React requests data over HTTP. Flask validates the caller and request, SQLAlchemy builds database operations, and PostgreSQL stores the record and enforces constraints. Docker Compose provides the network and configuration. Migrations prepare the schema. CI tests a disposable PostgreSQL instance. AWS stores the production data volume. QA compares known records to API totals and displayed dashboard/report values.

This separation matters: the frontend should not bypass Flask, Flask should not assume every request is valid, and PostgreSQL should not depend on frontend validation to enforce uniqueness or relationships.

## 10. Capstone evidence
The capstone expects PostgreSQL persistence, Docker Compose, automated checks, and deployment to the approved AWS target. Evidence should include migration files, passing migration CI logs, schema inspection for expected constraints/indexes, proof that data survives normal container recreation, and migration verification against a populated staging volume. A clean CI database test is necessary but does not replace a populated-volume upgrade test.

## 11. Tough questions
**Why PostgreSQL instead of localStorage?** PostgreSQL is centralized durable storage with transactions, relations, constraints, and server-side queries. Browser storage can be edited and is not a shared authoritative record.

**Why use foreign keys?** They prevent references to nonexistent records and preserve relationship integrity.

**Why restrict category deletion?** Because an expense or budget may depend on that category. Silent deletion could change the meaning of existing financial records.

**Why use two partial budget indexes?** NULL represents a whole-month budget, and ordinary uniqueness with NULL does not enforce the desired rule. Separate indexes protect category-specific and whole-month budgets.

**What if migration 003 finds duplicates?** It should stop and require intentional data review and cleanup; it must not silently discard records.

**Why must migrations run on existing volumes?** The PostgreSQL initialization directory is processed for a new data directory, not every restart.

**What is a transaction?** A group of database operations treated as one unit; commit makes changes durable, and rollback abandons the failed transaction.

**Why use NUMERIC(12,2)?** It stores fixed-scale decimal currency values. Validation and consistent arithmetic are still required.

**Why not use a database superuser for the app?** Least privilege limits the damage possible if the application's credentials are compromised.

**How do you prove a migration is safe?** Test on a clean database and legacy-like data, inspect the resulting schema, test staging upgrade, verify records, and retain backup/recovery evidence.

## 12. Defense-ready summary
“PostgreSQL stores durable financial records. Primary keys, foreign keys, check constraints, and unique indexes protect data integrity. SQL migrations evolve the schema in order, and existing AWS volumes need explicit upgrades. CI checks are important, but safe live migration also requires backups, existing-data tests, schema verification, and runtime evidence.”
