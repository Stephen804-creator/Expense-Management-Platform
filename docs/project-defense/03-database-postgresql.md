# Session 3 — Database: PostgreSQL, Relationships, Constraints, and Migrations

## 1. What is a database?

A database is an organised system for storing and retrieving information. Expense Tracker needs more than a list visible on one browser: it must retain users, expenses, categories, budgets, and summary data across page reloads and sessions. PostgreSQL is the relational database used by the backend.

A **relational database** stores data in tables. A table has columns describing the fields and rows representing individual records. A user row represents one registered account; an expense row represents one expense record. Tables can refer to each other through keys.

Why not keep all records only in React state? React state lives in the browser and is not a durable, shared source of truth. A central database allows the server to retrieve data consistently and apply integrity rules.

## 2. Tables, rows, columns, and data types

A table is similar to a spreadsheet in appearance, but it has formal rules and supports queries, constraints, concurrent transactions, and relationships.

The initial schema defines these important tables:

- **users:** identity details, email, password hash, and timestamps.
- **expense_categories:** category name, description, and creation time.
- **expenses:** user, category, amount, description, expense date, and creation time.
- **budgets:** user, optional category, amount, month, and creation time.
- **monthly_expense_summary:** summary values grouped by user, optional category, and month.

A database schema describes the structure and rules of the data. It is different from the application source code, which describes the operations performed on that data.

Types matter. IDs use large integer types. Descriptions use text fields. Expense amounts use NUMERIC(12,2), which stores decimal monetary values with a defined scale. Expense dates use DATE because a purchase date is a calendar date, not necessarily a moment with a timezone. Creation timestamps record when the row was created.

## 3. Primary keys and foreign keys

A **primary key** uniquely identifies a row. In this project, each main table has an ID column. The ID lets the application refer to one record even when two expenses have the same description or amount.

A **foreign key** says that a value in one table must refer to a valid row in another table. For example, expenses.user_id refers to users.id, and expenses.category_id refers to expense_categories.id.

These relationships express real rules:

- An expense belongs to one user.
- An expense refers to a category.
- A budget belongs to one user.
- A budget may refer to a category, or category_id may be NULL for a whole-month budget.

Foreign keys prevent orphan records. Without them, the database could contain an expense whose user or category does not exist. The application would then need to guess what that reference means.

A foreign key does not automatically implement every business rule. It can prove that a category exists, but it does not by itself prove that the category is permitted for a particular user. That requires a policy and appropriate backend checks.

## 4. Constraints: the database as a final guard

A constraint is a rule that the database enforces regardless of which application path sends the query.

Examples in this schema include:

- NOT NULL for required fields.
- UNIQUE for email and category name.
- CHECK that an expense amount is greater than zero.
- CHECK that a budget amount is greater than zero.
- CHECK that the stored budget month is the first day of a month.
- Foreign keys to valid users and categories.

Why duplicate validation across backend and database? The backend can return a clear message, but other code paths or simultaneous requests can bypass a pre-check. The database is the final authority for constraints it can express.

For example, suppose two requests attempt to register the same email at almost the same time. Both requests may check that the email is not present before either insert commits. Only a unique database constraint can reliably prevent both rows from being stored. The API then catches the known uniqueness conflict and returns an appropriate response, such as 409 Conflict.

## 5. How a transaction works

A transaction is a group of database operations treated as one logical unit. A successful commit makes the changes durable. If an operation fails before commit, the transaction can be rolled back so partial changes are not left behind.

In the backend, adding a SQLAlchemy object to the session is not the same as committing it. The commit can fail due to a constraint, a connection problem, or another database error. The application must handle failure and roll back the session before reuse.

Imagine creating an expense and updating a stored summary. If the application performs two writes and the second fails, it must not leave the system in a state where the expense exists but the summary is falsely treated as updated. A safer design calculates summaries from the authoritative expense records or updates related data within a carefully controlled transaction.

PostgreSQL supports concurrent transactions, but concurrency creates edge cases. A check-then-insert sequence is not a reliable uniqueness guarantee by itself. Constraints and appropriate transaction handling are required.

## 6. Money, precision, and totals

Money needs careful representation. Many decimal fractions cannot be represented exactly by binary floating-point values. PostgreSQL NUMERIC(12,2) stores decimal amounts at two fractional digits, which is suitable for many ordinary expense amounts.

The backend should reject non-numeric, zero, negative, or overly precise amounts according to the application's rules. The database also checks that an expense amount is positive. This is defence in depth: the API provides helpful validation, while the database protects the invariant even if a code path misses the check.

For summaries, define the rounding policy and aggregation logic. A category total should be the sum of qualifying expenses for the correct user and month. If the API converts database decimals to JSON numbers, the frontend must display them consistently and tests should use realistic monetary values. Never answer that a chart library guarantees the accuracy of a financial total.

## 7. Why nullable category_id is important for budgets

The budget schema allows category_id to be NULL. That supports two different concepts:

- A **category budget**, such as a limit for transport in October.
- A **whole-month budget**, which does not target one category.

NULL is not the same as zero or an empty string. It means there is no category reference for that budget.

This creates a subtle uniqueness issue in PostgreSQL: ordinary UNIQUE constraints allow multiple rows with NULL in a constrained column because NULL is treated as unknown for uniqueness purposes. The correction branch therefore uses separate partial unique indexes: one for budgets with a category, keyed by user, month, and category; and one for whole-month budgets where category_id is NULL, keyed by user and month.

Why is this important? A single user should not accidentally create multiple identical budgets for the same category and month, nor multiple whole-month budgets for the same month, if that is the agreed product rule. The API should return a meaningful conflict response when a duplicate is attempted.

## 8. Deletion rules and their business meaning

A foreign key can define what happens when a referenced row is deleted. Common actions include:

- **CASCADE:** delete dependent rows too.
- **RESTRICT / NO ACTION:** prevent deletion while dependent rows exist, subject to the database's constraint semantics.
- **SET NULL:** preserve the dependent row but clear its reference.

For expenses, the schema uses a restrictive category relationship because deleting a category should not leave an expense pointing to a nonexistent category.

For budgets, the original migration used ON DELETE SET NULL. That is risky when NULL means “whole-month budget”: deleting a category could silently change a category-specific budget into a whole-month budget. The correction branch proposes migration 004 to change the budget-category foreign key to ON DELETE RESTRICT and an API-level guard that returns HTTP 409 if a category has a linked budget. That proposal must not be described as part of the main release unless it has been merged and deployed. During defence, state which revision you are demonstrating.

This example shows why a database design decision is also a product decision. The database action must preserve the meaning of the user's financial plan.

## 9. Indexes and query performance

An index is an additional data structure that helps the database find rows more efficiently. The initial schema includes indexes for common lookups such as expense user, expense category, expense date, budget user, and budget month.

Without a suitable index, the database may need to scan many rows to find a small matching set. As data grows, that can make filtering and summaries slower.

Indexes are not free. They use storage and must be maintained during inserts, updates, and deletes. Add them based on real query patterns, not simply because every column exists. A query filtering by user and month may benefit from a composite index aligned with those filters; the actual query plan and workload should guide optimisation.

A common defence trap is to say that adding an index always makes every query faster. It does not. The database chooses a plan based on statistics, selectivity, and the query. Explain which lookup the index is intended to improve.

## 10. Schema migrations

A migration is a versioned change to database structure or data. The initial schema creates tables and constraints. Later migrations can add settings columns, enforce budget uniqueness, or change a foreign key.

Why not edit only the model file? SQLAlchemy model definitions describe the expected schema to the application, but changing a model file does not automatically update an existing production database. A migration makes the change explicit and repeatable.

Migrations must be safe for both a fresh database and an existing database containing real records. A script that works on an empty database may fail if existing duplicate budgets violate a new unique index. The budget uniqueness migration checks for duplicates before creating the indexes and stops with an explanation if conflicts exist.

PostgreSQL container initialisation scripts run only when the database data directory is first initialised. Mounting a new SQL file into the init directory does not automatically apply it to an existing persistent volume. Therefore the deployment workflow must explicitly apply migrations in order. Backups, failure handling, and verification on a populated database are important before production changes.

## 11. What should the team be able to show?

For database defence, show the schema or migration files, identify primary and foreign keys, explain the uniqueness rules, and run representative queries. Demonstrate that an expense belongs to a user and category, that a budget is unique under the intended rule, and that invalid amounts are rejected.

The team should also be able to explain which migration revision is in the deployed application. A migration file in Git does not prove that the live AWS database has applied it. Use deployment logs and a schema inspection query as evidence.

## 12. Connections to the other sections

- **Frontend:** displays records and submits data, but does not connect directly to PostgreSQL.
- **Backend:** uses SQLAlchemy to validate, query, insert, update, delete, and commit records.
- **Security:** user-scoped queries prevent one account from reading another account's expenses.
- **Testing:** constraints, duplicate conflicts, rollback, and report totals need automated tests.
- **Docker Compose:** provides a PostgreSQL service and persistent volume.
- **CI/CD:** can create a temporary database, apply migrations, and verify constraints.
- **AWS operations:** persistent database data must survive container recreation, and migration success must be verified on the deployed database.

## 13. Difficult defence questions and answers

**Q1. Why use PostgreSQL instead of a spreadsheet or local storage?**  
PostgreSQL supports durable central storage, concurrent access, relationships, constraints, transactions, and indexed queries. A spreadsheet or browser storage does not provide the same server-side integrity and access-control model.

**Q2. What is the difference between a primary key and a foreign key?**  
A primary key identifies a row in its own table. A foreign key references a key in another table and enforces a relationship.

**Q3. What is the difference between a constraint and application validation?**  
Application validation produces useful responses and checks business rules. A database constraint is enforced at the storage layer regardless of which application path submits the data. They complement each other.

**Q4. Why use NUMERIC for money?**  
It stores decimal values with defined precision and avoids many binary floating-point representation errors. The application must still define validation, rounding, and display rules.

**Q5. Why can a normal UNIQUE constraint be insufficient for nullable category budgets?**  
PostgreSQL permits multiple NULL values under a standard unique constraint. Separate partial unique indexes can enforce one category budget per user/month/category and one whole-month budget per user/month.

**Q6. What is an orphan record?**  
It is a record whose reference points to a row that no longer exists. Foreign keys prevent many such cases by enforcing that the referenced row exists.

**Q7. What is a migration and why not change production tables manually?**  
A migration records a repeatable schema/data change. It allows the team to review, apply, test, and trace changes consistently across development, CI, and production.

**Q8. Why might a migration pass on a new database and fail on AWS?**  
The existing database may contain data that violates a new constraint, may be at a different schema version, or may have different permissions or connectivity. Test both fresh and populated databases and record the deployed migration state.

**Q9. What is the danger of ON DELETE SET NULL for category budgets?**  
If NULL means a whole-month budget, deleting a category can change a category-specific budget's meaning without the user editing it. A restrictive foreign key and clear API conflict response can preserve that meaning.

**Q10. Does an index guarantee fast queries?**  
No. It can improve selected lookups but costs storage and write work. The actual query plan, data volume, and filter patterns determine whether it helps.

**Q11. Why is a transaction important?**  
It groups related operations so they can succeed together or be rolled back. It reduces the chance of partial state when a step fails.

**Q12. Does a successful migration job prove the live database is updated?**  
No. It proves the migration job succeeded in that environment. The live AWS database needs separate deployment evidence or a schema inspection.

**Q13. What happens if two requests create the same budget simultaneously?**  
Both can pass an application pre-check. A database unique index prevents duplicate records; the backend should map the known constraint conflict to a clear 409 response and roll back the failed session.

**Q14. Why persist a monthly summary table if totals can be calculated from expenses?**  
A stored summary can improve performance for certain workloads, but it introduces a consistency obligation: every expense change must update the summary correctly. If the system does not need that optimisation, calculating summaries from source expenses may be simpler. Explain what the current implementation actually reads and updates.

## 14. Database defence checklist

Be able to explain every main table, primary key, foreign key, nullable field, unique rule, check constraint, deletion action, and index. Know which migration files are in the version being defended. Demonstrate a valid insert, an invalid insert, a duplicate conflict, and a query scoped to one user. Do not confuse “migration exists in Git” with “migration is applied in AWS.”
