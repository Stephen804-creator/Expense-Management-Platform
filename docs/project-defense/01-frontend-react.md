# Session 1 — Frontend Development with React
## Expense Tracker — Enhanced Personal Expense Management Platform

**Audience:** Team members preparing for project defense. **Purpose:** Explain the frontend from basic ideas to difficult questions, and connect it to the rest of the system.

## 1. What is the frontend?

The frontend is the part of the application a person sees and uses in a browser. In Expense Tracker, it provides the interface for signing in, viewing expenses, entering or editing a transaction, choosing a category, inspecting summaries and budgets, filtering records, and requesting CSV export.

Think of a restaurant. The menu and waiter are the customer-facing interface; the kitchen and its rules do the actual preparation. In a web application, the frontend presents controls and sends requests. The backend checks the request and applies business rules. The database stores lasting records. A button being visible does not prove the feature works, and a click does not prove a record was saved.

## 2. What React contributes

React is a JavaScript library for building user interfaces from reusable components. A component is a unit of interface and behavior. A page can be composed of smaller parts such as navigation, an expense form, a filter panel, a summary card, a chart, or a transaction table.

Components are useful because repeated behavior can be maintained in one place and pages become easier to reason about. React also organizes interfaces around state. State is information that can change while the application runs: a form's current values, selected date range, loading status, error message, or data returned by the API. When state changes, React updates the relevant rendered interface.

Do not confuse state with permanent storage. A form may hold an amount in browser memory, but the expense is not persistent until the backend accepts the request and the database transaction commits successfully.

## 3. HTML, CSS, JavaScript, and React

HTML describes the structure and meaning of the page: headings, forms, buttons, inputs, and tables. CSS controls layout, spacing, colors, responsiveness, and visual presentation. JavaScript provides executable behavior in the browser. React uses JavaScript to compose interactive interfaces and keep the rendered page synchronized with state.

If asked why all are needed, explain that they solve different problems. HTML describes content, CSS presents it, JavaScript handles browser behavior, and React provides a maintainable way to build and update complex interfaces.

Responsive design matters because the application should remain usable on different screen widths. The presence of CSS does not prove mobile usability; the team must test that fields, tables, buttons, errors, and charts remain accessible and understandable on smaller screens.

## 4. The complete journey when a user saves an expense

1. The user opens the expense form.
2. React displays the fields and tracks the values being entered.
3. The user submits the form.
4. The frontend performs early checks to give quick feedback.
5. The frontend sends an HTTP request to the Flask API, including authentication information when the endpoint requires it.
6. Flask validates the request, identifies the user, checks permissions and business rules, and asks PostgreSQL to perform the change.
7. PostgreSQL commits the transaction or rejects it.
8. Flask returns a response and HTTP status.
9. React handles the response and updates the screen.

Every step can fail. The network can be unavailable; authentication can expire; the server can reject a date; a database constraint can reject a duplicate; or the response may be an error. The frontend must not show success unless the server confirms success.

**Important principle:** Frontend validation improves usability, but backend validation protects the system. A user can bypass the browser by sending a request directly to the API. Security and data-integrity rules must therefore be enforced on the server.

## 5. Forms and validation

Validation means checking whether input follows the rules. There are several different kinds:

- **Presence and shape:** Is a required field present? Is the amount a number? Is the date in the required format?
- **Business rules:** Is the amount allowed? Does the selected category exist? Is the operation consistent with the budget rules?
- **Authorization:** Is this authenticated user allowed to create or change the requested record?

Frontend checks can catch obvious errors before a request is sent, but the backend must repeat the important checks. A hidden or disabled button is not an authorization control.

A numeric input is not a complete financial validation system. Non-finite values such as NaN and infinity, unexpected strings, negative values, and unusually large amounts need deliberate handling. The API and database remain responsible for canonical validation and storage rules.

Date-only fields need special care. A date such as 2026-10-12 represents a calendar day, not necessarily a time instant. JavaScript date/time conversions can shift a date across time zones. If the API contract expects YYYY-MM-DD, preserve that calendar-date representation instead of converting it unnecessarily into a timestamp.

## 6. Loading, success, empty, and error states

A robust screen handles more than the successful path:

- **Loading:** A request is in progress. Avoid confusing duplicate submissions and show that the system is working.
- **Success:** The server confirmed the operation. Update the view using the confirmed response or reload authoritative data.
- **Empty:** The request succeeded, but the user has no records or the active filter matches none.
- **Validation error:** The server rejected the submitted values.
- **Authentication error:** The user must sign in again or provide valid credentials.
- **Network/server error:** The operation did not complete as expected. Do not report success.

An empty list is not the same as a failed request. Treating both as empty can make users think their records have disappeared. Similarly, displaying a success toast before receiving the server response can falsely imply persistence.

## 7. Dashboard charts and summaries

A chart is a visual representation of data, not the source of truth. The team must be able to explain which records are included in a monthly total, what date boundaries apply, how categories are grouped, and whether edits and deletions are reflected.

A chart can look correct but be mathematically wrong. Possible causes include duplicate counting, stale data, inconsistent filters, date-boundary mistakes, or different definitions of the total. During testing, compare the chart with the underlying records and independently check the expected total.

## 8. Filters and CSV export

A date or category filter should communicate which criteria are active. The team should know whether filtering affects the displayed list only, the summary, the export, or all three. The behavior should be consistent and explained.

CSV export is a data feature, not merely a download button. Verify that the export contains only authorized records, follows the intended filter rules, includes clear headers, and correctly handles commas, quotes, and line breaks. Text that begins with spreadsheet formula characters can create formula-injection risks when opened in spreadsheet software. Do not claim that the application mitigates this risk unless the actual implementation and tests prove it.

## 9. Accessibility and usability

Forms need understandable labels, visible keyboard focus, and errors that identify the field that needs attention. Color should not be the only way to communicate a problem. Tables and charts should have clear headings or summaries. Accessibility is part of correctness: if a person cannot identify or operate a control, the feature is not fully usable.

## 10. How the frontend connects to the other sections

- **Flask backend:** Defines endpoint paths, methods, request fields, response shapes, status codes, and authentication requirements.
- **PostgreSQL:** Stores persistent records. The browser must not connect directly to the database or contain database credentials.
- **Authentication and security:** The interface can hide controls, but the API must enforce permissions.
- **Nginx:** Routes deployed browser traffic to the correct application service. Development URLs and deployed URLs may differ, so configuration must be explicit.
- **Docker and CI:** Package and check the application, but a successful build alone does not prove that the screen's requests work.
- **AWS:** Hosts the deployed environment. The browser reaches the application through its public endpoint; private database configuration must stay server-side.
- **QA:** Component tests, API tests, and end-to-end tests cover different layers. One type does not replace the others.

## 11. Difficult defense questions

**Q: If the frontend says an expense was saved, is it saved?**  
Only if the server confirms success after the write succeeds. A click or local state update is not proof of persistence.

**Q: Why validate in both frontend and backend?**  
Frontend validation gives fast feedback. Backend validation protects the system from requests that bypass the browser.

**Q: What is the difference between state and database data?**  
State is current interface/application information and may be temporary. Database data is persistent information committed by the backend.

**Q: Why can the chart total disagree with the list?**  
The list and summary may use different date boundaries or filters, the summary may be stale, records may be counted twice, or a request may have failed. Trace the API response and compare it with a manual calculation.

**Q: Why not let React connect directly to PostgreSQL?**  
The browser is not a trusted server environment. Direct database access could expose credentials and bypass server-side authorization and business rules. Flask is the controlled boundary.

**Q: What should happen when the network fails during save?**  
The interface must not claim success. It should explain the failure and preserve the user's work where possible. Retrying requires care because the first request may have reached the server even if the response was lost.

**Q: Is hiding a delete button enough to protect a record?**  
No. A caller can send the API request directly. The backend must enforce authorization.

## 12. A reliable answer structure

For any frontend feature, explain it in this order: **user need → visible control → state/data → API request → backend rule → database result → feedback shown → test evidence**. This connects the screen to the full system instead of stopping at “we used React.”

## 13. What to verify in the actual repository

Before defense, identify the real component/page, the function that calls the API, the matching Flask endpoint, and the test that covers the behavior. Check the current frontend package scripts and whether functional frontend tests actually run in CI. Do not claim every screen has automated tests merely because CI builds the frontend. Demonstrate the deployed behavior where available; a successful build is not proof of a working browser journey.

## 14. Capstone alignment

The handout requires React and features including authentication, adding/editing/deleting expenses, categories, monthly summaries, budgets, charts, filtering, and CSV export. The frontend should make these features usable and explainable. The handout also asks for an architecture diagram, a well-documented repository and README, a live URL, project objectives, team members, presentation slides, deployment process, challenges, solutions, results, future improvements, and demonstration screenshots.

The handout originally names an Azure VM. The tutor approved AWS as an alternative, and the team uses AWS. Explain this as an approved infrastructure substitution; do not claim the original PDF literally says AWS.
