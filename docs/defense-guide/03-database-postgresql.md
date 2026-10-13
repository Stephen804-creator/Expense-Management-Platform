# Session 3 — PostgreSQL, Data Modelling, and Data Integrity

## 1. What the database is responsible for

A database is the organised, persistent store for application records. In the Expense Tracker, those records may include users, expenses, categories, budgets, and calculated summaries. The database must preserve relationships and reject invalid combinations even if a route contains a programming mistake or two requests arrive at the same time.

The project uses PostgreSQL with SQLAlchemy. PostgreSQL is the database engine; SQLAlchemy is the Python layer that helps the backend represent rows as objects and execute database operations. They are related, but they are not the same technology.

A record in React state or browser `localStorage` is not automatically a PostgreSQL record. To claim database persistence, the feature must call the backend, and the backend must successfully commit the record.

## 2. Tables, rows, columns, and keys

A **table** holds records of one general kind. A **row** is one record. A **column** is one attribute of that record. For example, an expense row may contain an ID, user ID, category ID, amount, description, expense date, and creation timestamp.

A **primary key** uniquely identifies a row. It lets other records refer to it reliably even when two people have similar names or descriptions.

A **foreign key** is a database rule that says a value in one table must refer to an allowed row in another table. An expense's `category_id`, for example, should refer to a real category. Foreign keys prevent orphaned relationships.

A **unique constraint** prevents two rows from sharing a value or combination of values that must be unique. A **check constraint** can enforce a rule such as a positive amount. Application validation is useful, but database constraints protect integrity when concurrent requests or other code paths bypass a pre-check.

## 3. Why relationships matter

An expense belongs to a user. It references a category. A budget belongs to a user and a month, and may optionally reference a category. These relationships affect what can be deleted and how totals are calculated.

Suppose a budget of ₦50,000 is for “Transport” in October. If deleting the Transport category silently sets that budget's category ID to null, the budget might now be interpreted as a whole-month budget. That changes the meaning of saved data without the user editing the budget. This is not just a UI problem; it is a data-integrity problem.

The correction proposed in PR #54 adds migration 004 to change the category-budget foreign key to `ON DELETE RESTRICT`, so a category referenced by a budget cannot be deleted until the relationship is handled. It also adds a route-level conflict response. This correction must be described as proposed until the relevant branch is merged and deployed.

## 4. What SQLAlchemy does

SQLAlchemy's ORM maps database tables to Python model classes. A model defines how the application represents a row and which columns or relationships exist. For example, an `Expense` model may map to the `expenses` table.

Typical operations include:
- query a user's expenses;
- create an object and add it to the session;
- modify fields on an existing object;
- delete a record;
- commit or roll back a transaction.

The ORM makes application code easier to write, but the actual database still enforces constraints and stores the data. An ORM model and a SQL migration must agree. If the model says one thing and the deployed database schema says another, the app can behave differently between environments.

## 5. Transactions: commit and rollback

A transaction groups database operations into a unit. The aim is that the operation completes consistently or does not partially apply.

For an expense creation:
1. Validate the amount, date, category, and authenticated user.
2. Create the database object.
3. Add it to the session.
4. Commit the transaction.
5. Return success only after the commit succeeds.

If the commit fails, roll back. A rollback tells the database session to abandon the failed transaction state. Without it, later requests may continue to fail even after the original problem has passed.

For operations involving multiple records, transaction boundaries matter even more. If a process updates a budget and a ledger record, the team must define whether both updates must succeed together. Partial updates can make totals inconsistent.

## 6. Money, decimals, and dates

Financial amounts should not be handled carelessly with binary floating-point arithmetic. Decimal values such as 0.1 cannot always be represented exactly as binary floats. The backend's expense route uses Python's `Decimal` for amount validation, which is a sensible approach for decimal money values. The database column precision and serialisation should also be checked to ensure the value is stored consistently.

The API should define the accepted precision. If the application supports two decimal places, a request with three decimal places should be rejected or handled by a documented rounding rule. Silent rounding can produce unexpected totals.

Dates also need a clear contract. The backend uses ISO-like date strings in API responses and supports month filtering. A month filter should use a precise range: from the first day of the requested month up to, but not including, the first day of the next month. This avoids missing records on the final day.

The application should distinguish an **expense date** (when the transaction happened) from a **created timestamp** (when the record was stored). Those values answer different questions.

## 7. Budget uniqueness and concurrent requests

A user should not accidentally create two indistinguishable budgets for the same month and category if the business rule permits only one. The intended unique key may include user, month, and category. But a whole-month budget has a null category, and database engines differ in how unique constraints treat nulls. The team must inspect the actual schema and use a constraint/index strategy that correctly handles both category-specific and whole-month budgets.

An application-only duplicate check is not race-safe. Two requests can arrive at almost the same time, both see no existing budget, and both attempt to insert. A database unique constraint is the final protection. The backend should then return 409 for the expected duplicate conflict and 500 for an unexpected database failure.

## 8. Migrations and why they exist

A migration is a controlled change to the database schema, such as adding a column, index, table, or changing a foreign key. It allows the team to evolve the schema in a repeatable way instead of relying on someone manually editing a live database.

This repository includes numbered SQL migration files. The CI workflow applies migrations in order against a fresh PostgreSQL service and checks for expected tables. Docker's PostgreSQL initialization directory behaves differently: initialization scripts run when a new data directory is created. If a persistent volume already exists, adding another SQL file to that directory does not automatically rerun it.

Therefore, “the migration file exists” does not mean “the production database has the migration.” A deployment process must explicitly apply pending migrations and verify success before restarting the app. Test both a clean database and a database with existing data.

## 9. Persistent volumes, backups, and recovery

A Docker container is replaceable. A PostgreSQL data volume holds data beyond the lifetime of a container. Removing and recreating the database container should not erase the volume, but deleting the volume can erase the stored data.

Backups are separate from persistent storage. A volume protects against some container lifecycle events; it does not protect against accidental deletion, disk failure, corruption, or a destructive migration. A production plan should define backup frequency, retention, access control, and restore testing.

Before a schema change on a real database, the team should know:
- what data the migration changes;
- whether it is backward compatible;
- how long it may lock tables;
- what happens if it fails halfway;
- whether rollback is possible;
- how to restore from backup;
- how the running app behaves if the new schema is not ready.

## 10. Data integrity examples the tutor may use

**Example A: Category does not exist.** The backend should reject an expense referencing a nonexistent category. The database foreign key is a final safety net.

**Example B: Two requests create the same budget.** A unique constraint prevents both from being stored when the business rule says one is allowed. The API returns a controlled conflict.

**Example C: Database commit fails.** The route rolls back and does not return success. A later valid request should still work.

**Example D: Category deletion is attempted while an expense or budget references it.** The database and API should follow a documented policy. A conflict response is often safer than silently changing the meaning of financial records.

**Example E: Migration file is added to a running AWS instance.** The database will not necessarily run it automatically just because the file exists. The deployment process must apply it and capture the result.

## 11. How the database session connects to the capstone

PostgreSQL is the persistence layer in the capstone architecture. SQLAlchemy is the Python layer that maps backend operations to database queries, but it does not replace PostgreSQL or make constraints unnecessary.

- **React and Flask:** React sends a request; Flask validates it and calls the ORM; SQLAlchemy issues SQL; PostgreSQL checks constraints and stores the change; the API returns the result.
- **Docker Compose:** the database service must be healthy before the backend depends on it. The named volume is important because deleting and recreating a container should not automatically erase persistent records.
- **Migrations:** a SQL file mounted into PostgreSQL's initialization directory normally runs only when the database data directory is initialized for the first time. An existing volume needs a deliberate migration process; adding a file does not mean it automatically ran.
- **GitHub Actions:** migration checks in a temporary test database prove the tested migration sequence in that environment. They do not prove that an existing AWS database was upgraded successfully.
- **AWS:** credentials belong in protected runtime configuration, not source code. The team must verify backups, volume persistence, and a recovery procedure for the actual server.

**Capstone proof:** show the schema and constraints, run a known API operation, read back the stored row, and show migration/test evidence. If asked whether production data is safe, explain the actual backup and recovery process; do not claim backups exist unless there is evidence.

## 11. Tough defense questions

**Q1. Why do we need foreign keys if Flask checks that a category exists?**  
The route check improves the error message, but a race or another code path can still change the data between the check and insert. A foreign key enforces the relationship at the database boundary.

**Q2. What is the difference between a primary key and a unique constraint?**  
The primary key is the row's main identity and cannot be null. A unique constraint protects a uniqueness rule for one or more columns. A table can have one primary key and multiple unique constraints.

**Q3. Why not store every amount as a JavaScript number and trust it?**  
Browser numbers use floating-point representation and are user-controlled. Financial values need defined precision, server validation, and suitable database types. The backend's use of `Decimal` helps with decimal input validation, but the whole path must be checked.

**Q4. What is a migration?**  
A versioned, repeatable database schema change. It keeps environments in sync and makes the evolution of the schema reviewable.

**Q5. Why does adding a SQL file not update an existing database automatically?**  
The PostgreSQL image's initialization scripts are designed for initial database creation. Existing persistent data directories are not treated as brand-new databases, so deployment must explicitly run migrations.

**Q6. What does rollback mean?**  
It abandons the current failed transaction state so it does not remain pending and block subsequent work. It does not automatically undo a transaction that was already committed.

**Q7. Why is a unique index still needed if the route checks for duplicates?**  
Because concurrent requests can both pass the check. The database is the final authority for a uniqueness rule.

**Q8. How would you prove migration 004 works?**  
Apply it to a clean database and to a copy of a database with existing data. Inspect the foreign-key delete rule, create a budget linked to a category, attempt deletion, and confirm that the request fails safely while the budget retains its original category and amount.

**Q9. What is the difference between persistence and backup?**  
Persistence means the data remains available across normal application restarts. Backup is a separate recoverable copy used when data is lost or damaged. One does not replace the other.

**Q10. If CI says migrations passed, can we say AWS is safe?**  
No. CI proves the migration worked in the test database it created. AWS may have existing data, different configuration, locks, or a different migration history. Runtime evidence is still required.

## 12. Explain it in one minute

“PostgreSQL stores the durable records. Tables represent entities, primary keys identify rows, foreign keys preserve relationships, and unique constraints protect rules such as one budget per user, month, and category. SQLAlchemy lets Python work with these records, but PostgreSQL remains the final authority on database constraints. A transaction should commit only when the operation succeeds and roll back on failure. Migrations change schemas in a controlled order, and existing Docker volumes need explicit migration execution. We prove database correctness with migration tests, constraint checks, rollback tests, and safe tests against existing data.”
