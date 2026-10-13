# Session 3 — PostgreSQL, Data Modelling, and Transactions
## Expense Tracker — Enhanced Personal Expense Management Platform

## 1. Why the database exists

The database stores the application's lasting information. React can temporarily display an expense, and Flask can hold an object while handling a request, but those temporary states disappear when the page reloads or the server restarts. A database preserves records so they can be read again later.

Expense Tracker uses PostgreSQL as its relational database. A relational database organizes information into tables made of rows and columns, and relationships connect records across tables. For example, a user can have many expenses, a category can be referenced by expenses, and a budget can be associated with a user and optionally a category.

The database is not just a storage box. It is also an enforcement layer. Constraints can reject invalid relationships and duplicate values even when an application route contains a bug or two requests arrive simultaneously.

## 2. Tables, rows, columns, and keys

A **table** represents a kind of record, such as users, expenses, categories, or budgets. A **row** is one record. A **column** stores one property, such as an identifier, amount, date, or description.

A **primary key** uniquely identifies a row. An expense ID is not the same thing as an owner's permission: knowing an ID does not authorize a caller to access the row. The backend must still verify that the authenticated user owns or may access it.

A **foreign key** links a column in one table to a key in another. If an expense has a category ID, the foreign key can ensure the category exists. Foreign keys protect referential integrity—the rule that relationships point to real records.

A **unique constraint** or unique index prevents duplicate values or duplicate combinations of values. It is especially important when the product says that only one budget of a certain kind may exist for a user and month.

## 3. Why relationships matter

Relational modelling avoids repeating the same information unnecessarily. Instead of writing a category name into every expense, the application can store a category ID and join the category record when it needs the name. This reduces inconsistency: renaming the category does not require changing every expense row.

But relationships require deliberate deletion rules. If a category is deleted, what should happen to expenses or budgets that refer to it? The answer is a product decision, not a detail to leave to chance. Cascade deletion may remove dependent records; setting a foreign key to NULL may detach them; restricting deletion may prevent removal until dependencies are resolved.

For this project, the category-budget relationship deserves particular care. If deleting a category sets the budget's category ID to NULL, a category-specific budget may accidentally become a whole-month budget. That changes the meaning of a financial rule. The proposed correction adds a pre-check in the API and migration 004 with ON DELETE RESTRICT so the database rejects deleting a category that is still referenced by a budget. The expected API behavior is a clear conflict response rather than silent reclassification.

## 4. Budget uniqueness is a modelling question

The application distinguishes category-linked budgets from whole-month budgets. A whole-month budget has no category; a category budget is associated with a category. The uniqueness rule must match that distinction.

A useful business rule might be: one whole-month budget per user and month, and one category budget per user, month, and category. The exact database design must account for how PostgreSQL treats NULL values in unique constraints. In many ordinary unique indexes, NULL values do not compare equal in the way beginners expect, so simply adding a single unique constraint may fail to prevent multiple whole-month budgets.

A robust solution can use separate partial unique indexes: one for rows where category_id IS NULL and another for rows where category_id IS NOT NULL. The index conditions are part of the rule and should be verified against the actual migration, not assumed from its filename.

Application-level checks still help provide friendly messages, but the database constraint protects against race conditions. If two requests arrive at once, both can pass an initial “does this budget exist?” query. The unique index makes the database reject the duplicate when the transactions compete to commit.

## 5. Transactions: all or nothing

A transaction groups database operations into a unit. If the transaction commits, the changes become persistent. If the operation fails and is rolled back, the partial changes should not remain as if the whole operation succeeded.

Imagine moving money between two accounts. If the debit succeeds but the credit fails, the system should not leave the user with only half the intended operation. Expense Tracker is simpler, but the principle still applies to multi-step operations and database integrity.

SQLAlchemy sessions manage application changes and transactions. A commit can fail because a constraint is violated, the connection is lost, or the database rejects the statement. After a failed transaction, the application should roll back before reusing the session. The API should translate known failures into suitable HTTP responses while hiding internal details from the client.

## 6. Data types and money

Database types matter because they determine which values can be stored and how operations behave. A date-only field should represent a calendar date rather than an arbitrary timestamp if the business concept is “the day this expense occurred.” This helps avoid time-zone shifts in date filtering.

Financial amounts require special care. Binary floating-point formats cannot exactly represent every decimal fraction. For money, systems often use fixed-precision decimal types or store integer minor units (such as cents or kobo) according to a defined currency model. The actual schema and API contract determine which approach this project uses; the team must inspect those files before making a claim. Never say the project uses an exact-decimal column unless the schema confirms it.

The application should define currency assumptions, rounding rules, accepted precision, and amount boundaries. If these are unspecified, that is a design question to raise rather than silently assume. Aggregation and chart totals must follow the same rules.

## 7. Querying and aggregation

A query reads records. Filtering narrows the records by conditions, joining combines related tables, grouping collects rows by a key, and aggregation calculates values such as count or sum.

Monthly summaries may need to group expenses by month or category. Date boundaries are a common source of defects. For a date-only field, a range may be inclusive of both selected dates; for timestamps, a safer pattern is often a half-open interval that includes the start and excludes the next period's start. The correct approach depends on the stored type and API contract.

A monthly total should be independently testable. Create a small known dataset, calculate the expected total by hand, call the summary endpoint, and compare the result. Include dates at the beginning and end of the selected range, different categories, edited records, and deleted records.

## 8. Migrations: changing a database safely

A migration is a controlled change to a database schema. Examples include adding a column, creating a unique index, or changing a foreign-key deletion rule. A migration is different from a fresh database initialization script. Initialization scripts commonly run only when a database volume is first created; they do not automatically rerun on every container restart.

This distinction matters in Docker Compose. A new database container attached to an existing persistent volume may reuse the old database and skip initialization scripts. Therefore, a migration required by an application update must have a deliberate execution path for both fresh and existing databases.

The current correction branch contains migrations 002, 003, and 004 alongside the initial schema. Migration 004 protects the budget-category relationship. The CI migration job passed for the recorded correction checkpoint, but that proves behavior in CI's test environment, not that a populated AWS database was migrated successfully. Before production use, verify both a clean database and a populated existing database, take appropriate backups, and define what happens if a migration fails.

Never assume a migration is safe merely because its SQL file exists. Check whether it is idempotent where needed, whether it can handle legacy data, whether it locks large tables, and whether rollback is possible. Some migrations cannot be safely reversed without losing data; that should be documented.

## 9. Indexes and performance

An index is an additional data structure that helps the database find rows efficiently. Indexes may improve lookups by user, date, category, or unique key, but they also consume storage and make inserts or updates more expensive because the index must be maintained.

Do not add indexes without considering the actual query patterns. The most important first step is to ensure that user-scoped queries and common filters are correct. Performance tuning can then be guided by query plans and measured response times.

## 10. Database security and configuration

The database should not be exposed directly to the public internet for ordinary browser access. The backend connects using server-side configuration. Passwords and connection strings belong in environment configuration or a managed secret store, not in committed source code.

A placeholder in an example file is not a secret and must not be used as a production credential. If a real password was ever committed, removing it in a later commit does not erase its history; rotate the credential. Least privilege means the application database account should have only the permissions it needs rather than broad administrator access.

Backups and restoration are also part of database reliability. A backup is only useful if it can be restored. The team should know who can access backups, how often they are taken, where they are stored, and how a restore is tested. Do not claim backups or point-in-time recovery are configured unless there is evidence.

## 11. How the database connects to the rest

- **React:** Never connects directly to PostgreSQL. It talks to Flask.
- **Flask/SQLAlchemy:** Converts validated application operations into database queries and transactions.
- **Nginx:** Routes HTTP traffic; it should not expose the database port as a public web endpoint.
- **Docker Compose:** Can create a PostgreSQL service and persistent volume. The volume preserves data across ordinary container replacement.
- **CI/CD:** Must apply schema changes in a deliberate order before deploying code that depends on them.
- **AWS:** Persistent data must survive intended deployments, and credentials/network access must be protected.
- **QA:** Tests constraints, migrations, isolation, date boundaries, rollback behavior, and recovery after failed transactions.

## 12. Difficult defense questions

**Q: Why do we need a database if Flask can store variables?**  
Variables are temporary and usually disappear when a process restarts. A database provides persistent, queryable, concurrent storage.

**Q: What is the difference between a primary key and a foreign key?**  
A primary key identifies a row. A foreign key links a row to another table and can enforce that the referenced record exists.

**Q: Why do we need database constraints if we validate in Flask?**  
Validation gives a useful error before a write, but concurrent requests and alternate code paths can bypass application checks. Constraints enforce invariants at the data layer.

**Q: What is a race condition in budget creation?**  
Two requests both check that no budget exists before either inserts. A unique database index makes one conflicting insert fail instead of allowing duplicates.

**Q: What is the danger of ON DELETE SET NULL here?**  
A category-linked budget could lose its category and be interpreted as a whole-month budget. That changes the meaning of the user's saved financial plan.

**Q: Why does a migration need to run on an existing database?**  
An existing Docker volume persists old schema and data. Initialization scripts may not rerun, so the deployment needs a controlled migration process.

**Q: Does a green migration test prove the AWS database is updated?**  
No. It proves the migration worked in the CI environment. AWS deployment logs and database inspection are separate evidence.

**Q: Why is NULL special in uniqueness?**  
SQL NULL represents an unknown/absent value and has special comparison semantics. A uniqueness rule for optional categories must explicitly handle whole-month rows.

## 13. What to demonstrate

Show the schema and explain the tables and relationships. Demonstrate that an expense references the intended user and category. Show a duplicate budget conflict and the category-delete protection. If you cannot access the live database, label the evidence as CI or source inspection rather than claiming a live AWS check.

## 14. Capstone alignment

PostgreSQL is an explicit capstone requirement. The project must show how data persists, how the backend queries it, how constraints protect correctness, how secrets are configured, and how migrations are deployed. The architecture should show PostgreSQL behind Flask, not directly exposed to React. AWS is the tutor-approved alternative to the Azure VM named in the original handout.
