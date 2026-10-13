# Session 5 — Expense Tracker Features and Business Logic

## 1. Start with the problem the application solves

The Expense Tracker is intended to help a person record spending, organise it by category, set budgets, review totals, and understand patterns over time. The point is not simply to store numbers. The application should help users answer practical questions: How much did I spend this month? Which category used the most money? Am I close to my budget? Can I retrieve or export the transactions behind a total?

A defense answer should connect every feature to a user problem, the data it needs, the rule it applies, and how the team proves the result is correct.

## 2. Expense records

An expense generally needs an identity, owner, category, amount, date, and optional description. These fields serve different purposes:
- ID identifies the record.
- User ID determines ownership.
- Category groups the spending.
- Amount records the monetary value.
- Expense date says when the spending occurred.
- Description gives human-readable context.
- Created timestamp says when the system stored it.

The backend expense routes validate amount and date input and scope list and detail queries to the authenticated user. An expense should not be saved if its category is invalid or if its amount violates the documented rules.

The user ID must come from verified authentication, not from an arbitrary user-supplied field. Otherwise a caller might create records under another person's identity.

## 3. Categories

Categories group similar expenses, such as food, transport, utilities, or education. Categorisation supports filtering, charts, budget limits, and reports. A category is not the expense itself; many expense rows may reference one category.

A category deletion policy must consider the records that reference it. If a category is used by existing expenses or budgets, deleting it can break history or alter the meaning of a budget. The team should decide whether to prevent deletion, archive the category, or reassign linked records. That choice should be documented and tested.

The reviewed baseline returns a conflict when category deletion fails because an expense references the category. The separate budget relationship needs its own protection. The correction proposed in PR #54 adds a budget-reference check and a database-level `ON DELETE RESTRICT` rule through migration 004. Confirm whether this correction has reached the branch being demonstrated.

## 4. Budgets

A budget is a spending limit for a defined period. The project backend uses a month and may allow a budget linked to a category or a whole-month budget without a category.

A category budget asks, “How much should I spend on transport in October?” A whole-month budget asks, “What is my total spending limit for October?” These are different meanings, even if both use the same amount type.

To determine budget usage, the application needs a clear rule. A category budget should sum qualifying expenses for that user, category, and period. A whole-month budget should sum qualifying expenses for that user and period. The calculation must not include another user's records, the wrong month, or records outside the selected category.

The remaining amount is commonly:
`remaining = budget_limit - eligible_spending`

A negative remaining amount means spending exceeded the limit. The percentage used is commonly:
`percentage_used = eligible_spending / budget_limit × 100`

The implementation must decide how to display percentages above 100%, how to handle a zero or missing limit, and how to handle rounding. It should not divide by zero.

The budget's uniqueness rule also matters. If only one budget is allowed for a user/month/category, both the API and database need to enforce it. Whole-month budgets need deliberate handling because their category ID may be null.

## 5. Income and expenses are not interchangeable

The frontend README documents income management and a unified transaction ledger. The reviewed backend route registration includes authentication, categories, expenses, budgets, and summaries, but no dedicated income route is visible in that registration. The database migration check also names tables for budgets, categories, expenses, monthly summaries, and users, but not an income table.

This does not prove that every income-related screen is broken, but it does mean that the team must not assume income has the same backend persistence as expenses. Check the actual React income handler, API calls, backend routes, and database schema. If income remains in browser storage, say so clearly and do not describe it as a server-persisted feature.

A professional defense acknowledges a gap precisely: “The frontend documents income functionality, but backend persistence for income is not established by the current route and schema evidence. We are checking and aligning the feature before claiming full-stack persistence.”

## 6. Dashboards and reports

A dashboard summarises underlying records. It may show total income, total expenses, net savings, budget status, and recent transactions. A report groups records by a time period and may break spending down by category.

A total is only as correct as the records and filters used to calculate it. To test a monthly report:
1. Create a small known set of transactions with different dates and categories.
2. Calculate the expected result independently.
3. Request or open the report for the target month.
4. Compare total, transaction count, category breakdown, and net result.
5. Add a transaction on the first day of the next month and confirm it is excluded from the previous month.
6. Repeat with no transactions and with a single transaction.

If the frontend calculates the report from localStorage, its result may differ from the server's result. The team must identify which source is authoritative for the deployed feature and avoid combining two datasets accidentally.

## 7. Search, filters, and CSV export

Search and filters should narrow records without changing them. The backend supports expense filtering by category and month, and a CSV export route uses the same base query. Reusing a query builder is valuable because the list and export should apply the same ownership and filter rules.

CSV export has security and correctness concerns. It should export only the authenticated user's records and honour the chosen filters. Values should be escaped using a proper CSV writer so commas, quotes, or newlines do not break the file format. If user-controlled text is later opened in spreadsheet software, the team should consider spreadsheet formula injection for cells beginning with formula characters.

A CSV export is not the same as a database backup. It may omit internal relationships, settings, or other tables.

## 8. Consistency across the application

Suppose the dashboard says ₦20,000 was spent this month, but the transaction page totals ₦18,000. That inconsistency could come from different date filters, duplicated records, a missing category filter, stale browser state, or different data sources.

For each calculated value, the team should identify:
- the source records;
- the user and date filters;
- the formula;
- the point where the calculation runs;
- the expected rounding;
- how the result is tested.

The same rule should not be independently implemented in three places unless there is a clear reason. If the frontend needs an instant preview, it can calculate locally, but the server should remain authoritative for persisted data and protected summaries.

## 9. Business rules versus technical choices

A business rule describes what the application must mean. For example, “one budget per user, month, and category” is a business rule. PostgreSQL's unique index is a technical mechanism used to enforce it.

A category may be global or user-owned. That is a product/security decision, not something React or PostgreSQL can decide automatically. Once the team decides, the schema, API queries, UI, tests, and documentation must agree.

A tutor may intentionally ask an ambiguous question to see whether the team distinguishes the rule from the implementation. State the rule first, explain how it is enforced, and describe the evidence.

## 10. How business logic connects to the capstone

Business logic is the meaning of the application. It explains what counts as an expense, which category a budget applies to, how a monthly total is calculated, and what should happen when data is invalid or conflicts with existing records.

- **React:** collects values and presents totals, progress, and alerts. It can make a screen easier to use, but it must not be the only place where important rules are enforced.
- **Flask:** is responsible for validating values and applying server-side rules before records are saved or totals are returned.
- **PostgreSQL:** preserves critical relationships and uniqueness rules even when concurrent requests arrive or an API route has a bug.
- **Reports and dashboards:** must use consistent date boundaries, categories, ownership filters, and formulas. A dashboard total and a report total should agree when they cover the same records and period.
- **QA and CI:** use known test data with manually calculated expected totals. Test normal cases, empty months, boundary dates, duplicate budgets, deleted categories, and invalid amounts.
- **AWS deployment:** the demonstrated result must come from the actual deployed API and database. A feature in source code or a browser-only demo does not prove that it works on the server.

**Important accuracy rule:** the repository has both a root-level frontend and a `frontend/` frontend. The Compose file builds `./frontend`; the root README also describes localStorage-backed records. Verify the active UI and persistence path before claiming that every listed screen is connected to PostgreSQL. The guide teaches the intended rules, but only tested implementation evidence proves that a feature is complete.

## 10. Tough defense questions

**Q1. Why store a category ID instead of the category name in every expense?**  
The ID creates a stable relationship to one category record. If the label changes, linked expenses can still refer to the same category. A foreign key can also ensure that the referenced category exists.

**Q2. What is the difference between an expense date and a creation timestamp?**  
The expense date is when the spending happened; the creation timestamp is when the record entered the system. A user may record yesterday's expense today.

**Q3. How would you test the monthly total?**  
Use a small known dataset with records before, within, and after the month. Calculate the expected sum independently and verify the API/report uses the correct start-inclusive and next-month-exclusive boundaries.

**Q4. What should happen when a budget is exceeded?**  
The application should calculate eligible spending, compare it with the limit, and show an accurate over-budget state or alert. The alert is a consequence of the rule, not a replacement for calculating the total correctly.

**Q5. Why is a negative remaining budget not necessarily an error?**  
It may correctly mean that spending exceeded the limit. The UI should distinguish a valid over-budget result from invalid input or a failed calculation.

**Q6. Why should a budget linked to a category not become a whole-month budget when the category is deleted?**  
Because the budget's meaning changes without the user's consent. The deletion should be blocked or handled through an explicit documented reassignment/archive process.

**Q7. Does CSV export prove that all data can be restored?**  
No. CSV is a convenient export of selected records. It may omit relationships, account settings, or other tables and is not a complete backup/restore system.

**Q8. How do you know the dashboard and report are consistent?**  
Test both against the same known records, filters, and formulas. If they use different sources or different date boundaries, the numbers may diverge.

**Q9. The UI has an income page. Does that prove income is stored in PostgreSQL?**  
No. The screen could store data locally. Verify the actual network request, backend endpoint, database schema, and persistence after reload or login on another browser.

**Q10. Why not calculate every total only in React?**  
Client-side calculations are useful for display, but they can be modified and may use stale or incomplete data. Authoritative reports and protected financial data should be calculated from the correct server-side records or verified server responses.

**Q11. What is the difference between a category budget and a whole-month budget?**  
A category budget limits spending in one category. A whole-month budget limits spending across eligible categories. The query filters and uniqueness rules must reflect that difference.

**Q12. What is your first debugging step if a budget alert is wrong?**  
Reproduce it with known transactions, verify ownership/category/month filters, calculate the expected sum independently, inspect the backend response and persisted rows, then inspect the frontend's display logic.

## 11. Explain it in one minute

“The application turns individual expense records into useful information. Categories group records, budgets compare eligible spending with a limit, and reports aggregate transactions by month. Each calculation depends on clear business rules for ownership, category, date range, precision, and uniqueness. The database must preserve the relationships behind those calculations. We test features using small datasets with known expected answers, and we verify that the frontend and backend use the same data source. We do not claim a screen is server-persisted merely because it appears in React.”
