# Session 1 — Frontend Development

## What the frontend is
The frontend is the part of the Expense Tracker the user sees and interacts with. This project uses React, JavaScript/JSX, Vite, CSS/Tailwind, Lucide React icons, and Recharts. The active frontend lives in the frontend directory. The repository also has legacy root-level frontend assets, but Docker Compose builds the application from frontend/.

The frontend displays pages and forms, gathers input, sends requests to Flask, and turns responses into visible records, charts, reports, and messages. It is not the entire application. A button can be visible while the API request fails, and a successful browser interaction does not automatically prove that the record was safely saved.

Think of the frontend as a control panel, Flask as the rule-enforcing service, and PostgreSQL as the durable ledger. The control panel should not be the final authority for financial data or security.

## React components and state
React builds the interface from components. The app contains pages such as Dashboard, Budgets, Categories, Transactions, Reports, and Settings, plus shared navigation, authentication, and toast feedback. Components help separate responsibilities and reuse behavior. A page should manage its workflow; shared components should provide consistent UI behavior.

State is information that can change while the application runs: form values, selected page, user status, search text, data lists, loading indicators, theme, and modal visibility. The app uses useState for changing values, useEffect for lifecycle-related work such as loading data, useMemo for derived calculations, and useCallback to preserve function identity where useful. Context shares authentication and toast behavior with components that need it.

State is temporary browser information, not the database. It can become stale or disappear on refresh. PostgreSQL remains the durable source of records, so the frontend must load and update data through the API.

## Shared API client
The shared client is frontend/src/services/api.js. It uses /api as the default base path, sends Fetch requests, attaches a bearer token when one is stored, serializes JSON request bodies, parses JSON responses, handles no-content responses, and throws for unsuccessful HTTP status codes. When a request returns 401, the client removes the stored token and notifies the authentication context.

A shared client prevents every page from inventing its own token/header/error logic. The current app stores the JWT in localStorage. That survives a page refresh but is readable by JavaScript, so a cross-site scripting flaw could expose the token. Do not describe localStorage as risk-free. Preventing script injection and reviewing token-storage design are security responsibilities.

## One expense from click to database
1. The user fills in category, amount, date, and description.
2. React holds the form values in component state.
3. The submit handler sends a request through the shared client.
4. Nginx forwards /api/expenses to Flask.
5. Flask validates the token, identifies the user, validates the request, and checks the category.
6. SQLAlchemy sends the write to PostgreSQL.
7. PostgreSQL constraints protect the stored state.
8. Flask returns a status and response body.
9. The frontend updates its state or shows a meaningful error.

This full chain is why a working button alone does not prove the feature works. The browser, proxy, API, database, and response handling must all behave correctly.

## Validation, dates, and money
Frontend validation makes the app easier to use, but it can be bypassed. Flask must repeat security-relevant validation. The backend checks that amounts are finite and positive and have no more than two decimal places; PostgreSQL stores them in a fixed-precision numeric column. The browser should not be treated as the accounting authority.

Date-only values need care. A date such as 2026-10-13 is a calendar date, not a UTC timestamp. If code converts it incorrectly through a timezone, the displayed date can shift. Test form defaults, filters, dashboard totals, and reports at timezone boundaries.

Charts and dashboard cards are derived information. Their correctness depends on the records included, user scope, date boundaries, calculation, and rounding. Create a small known dataset, calculate the expected total independently, and compare the API, database, and displayed value. A chart can be visually polished and still be wrong.

## Loading, errors, accessibility, and feedback
A professional interface distinguishes loading, empty, success, and failure states. An empty report means there are no matching records; a failed API request means the data could not be loaded. Treating both as an empty list misleads users.

Form controls need labels, keyboard navigation should work, focus should be visible, and color should not be the only way to communicate status. Toasts provide feedback, while confirmation dialogs reduce accidental deletion. Neither replaces backend authorization or database constraints. A success toast should only appear after the server confirms success.

## Connections to the rest of the stack
- Flask validates requests and returns status codes.
- PostgreSQL stores durable records; React state is temporary.
- Nginx routes /api to Flask and other paths to the frontend.
- Docker Compose packages and connects the services.
- AWS EC2 hosts the deployed containers.
- GitHub Actions checks configured lint/build/test steps.
- QA verifies complete user journeys and compares results to known data.

The capstone requires a React frontend connected to Flask and PostgreSQL, packaged with Docker Compose, routed by Nginx, checked through GitHub Actions, and deployed to the approved AWS target. A successful frontend build proves that a bundle can be produced; it does not prove all browser interactions work. Keep browser evidence tied to the commit that was tested.

## Tough defense questions
**Why is React state different from PostgreSQL data?** React state is temporary UI information; PostgreSQL stores durable records. The frontend fetches and changes records through the API.

**Why validate in the frontend and backend?** Frontend validation gives quick feedback; backend validation protects requests that bypass the browser.

**Why use one API client?** It centralizes API path, token header, JSON parsing, error handling, and 401 behavior.

**Does hiding a delete button secure deletion?** No. A direct API request can bypass the UI, so Flask must enforce permission.

**Why can a date show one day off?** A date-only value can be treated as a timestamp and shifted by timezone conversion.

**Does a chart prove the total is correct?** No. Check the data scope, date boundaries, arithmetic, rounding, and agreement with API/database values.

**What is the risk of localStorage tokens?** JavaScript can read them, so an XSS flaw may expose a token.

**What evidence proves the frontend works?** Current lint/build plus browser tests for login, expense CRUD, filtering, budgets, reports, export, errors, and data refresh, tied to the tested version.

## Defense-ready summary
“The frontend is a React interface that manages temporary UI state and communicates with Flask through a shared HTTP client. It does not own authoritative financial records or security rules. Flask validates requests, PostgreSQL persists data, Nginx routes traffic, Docker packages services, AWS hosts the deployment, and CI/QA provide evidence about the tested version.”
