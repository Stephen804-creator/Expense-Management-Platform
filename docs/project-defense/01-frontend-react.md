# Session 1 — Frontend Development: React and the User Interface

## 1. What is the frontend?

The frontend is the part of a web application that users see and interact with in a browser. In Expense Tracker, it includes navigation, dashboard cards, transaction lists, forms, category controls, budget screens, charts, reports, loading messages, error messages, and theme controls.

Think of a restaurant. The frontend is the menu and the waiter who receives an order. It helps the customer choose and submit what they want. It does not replace the kitchen, stock records, or payment ledger. In a web application, the backend performs trusted processing and the database preserves the official records.

A frontend can look complete while the underlying system is broken. A form may accept an amount visually, but if it never sends a request to the backend, the expense is not saved centrally. A chart may look beautiful but show incorrect totals if it receives stale or wrongly grouped data. A login modal is not proof of secure authentication; the server must verify credentials and protect the API.

**Defence principle:** The frontend presents information and collects user actions. It is not the trusted authority for security, ownership, or financial correctness.

## 2. Technologies used

The project uses React, JavaScript modules, Vite, Tailwind CSS, Lucide React icons, and Recharts.

- **React** builds the interface from reusable components. Developers describe the interface for the current state, and React updates the relevant parts when state changes.
- **JavaScript** runs browser behaviour: handling clicks, preparing requests, processing responses, filtering visible records, and coordinating state.
- **Vite** provides the development server and production build process. The production build creates static assets that can be served by a web server or container.
- **Tailwind CSS** provides utility classes for layout, spacing, colours, responsiveness, and visual consistency.
- **Lucide React** supplies icons. Important actions should still have understandable labels; an icon alone may not communicate its meaning to everyone.
- **Recharts** renders charts from data supplied by the application. It draws the visual representation; it does not decide whether the underlying financial calculations are correct.

Why not use plain HTML, CSS, and JavaScript? That is possible. React is useful when an application has many interactive screens and reusable pieces. Its component model helps organise changes. It is a design choice, not the only valid way to build a frontend.

## 3. Components and pages

A **component** is a reusable piece of interface. A summary card can display total expenses, total income, or net balance using the same layout but different values.

The project includes:

- **App.jsx:** coordinates main application state and selects which page is visible.
- **Auth.jsx:** provides authentication context and the authentication modal.
- **Sidebar.jsx and Header.jsx:** provide navigation, search, and theme controls.
- **Dashboard components:** display summary values, charts, recent transactions, and budget status.
- **Budgets.jsx, Categories.jsx, Transactions.jsx, Reports.jsx, Settings.jsx:** implement the main feature screens.
- **ToastContext.jsx:** makes temporary feedback messages available across the interface.
- **services/api.js:** centralises HTTP requests to the Flask API.

This separation makes a change easier to locate. If the chart legend is wrong, inspect the chart component. If every API request uses the wrong base path, inspect the shared API service. If a total is wrong, trace the value back to the response and calculation instead of assuming the chart is at fault.

Reusable components reduce copied code. If the same delete-confirmation UI is copied into five pages, one copy might be fixed while the others remain wrong. Reuse improves consistency and makes testing easier. However, creating a separate component for every tiny line of UI can also make the code difficult to follow; components should have a useful responsibility.

## 4. State, props, and effects

These concepts explain how React keeps the screen connected to changing information.

### State

State is information that can change while a component runs. In this app, examples include the active tab, search query, loaded expenses, categories, budgets, monthly summary, and dark-mode choice.

When state changes through a React state setter, React renders the interface using the new value. Developers should avoid directly mutating state arrays or objects because React may not detect the intended update correctly, and the previous state may be altered unexpectedly.

For example:

1. The user opens the transaction screen.
2. The frontend requests expenses from the API.
3. The server returns an array of expense objects.
4. React state stores that response.
5. The component renders a row for each expense.
6. After a successful create or delete, the app reloads or updates the relevant data.
7. The list, budget progress, and reports should reflect the saved state.

The array held by React is a working representation of the records, not the permanent database itself.

### Props

Props are inputs passed from a parent component to a child. For example, App.jsx passes categories to the Budgets screen so that it can show the available categories without independently recreating all data-loading logic.

Props and state are not the same. State is managed by a component or hook; props are supplied by another component. Both affect what the user sees.

### Effects

An effect synchronises a component with something outside its pure rendering process, such as loading data from an API or registering an event listener. Incorrect effect dependencies can cause repeated requests, stale values, or missed updates.

In App.jsx, the app loads categories, expenses, budgets, and summaries when a user is available. The main requests use Promise.all so independent resources can be requested concurrently. This can reduce waiting time, but it does not make the requests one atomic transaction. If one required request fails, the app must avoid pretending that all data loaded successfully. Some summary requests have separate fallback handling, so summary failure may be treated differently from failure to load core lists.

## 5. The complete frontend data flow

The central path is:

**User action → React event handler → API service → HTTP request → Flask route → PostgreSQL → HTTP response → React state → rendered screen**

Consider adding an expense:

1. The user enters an amount, date, description, and category.
2. The frontend checks obvious input errors and gives quick feedback.
3. The API service sends JSON to the Flask endpoint and includes the bearer token when one is available.
4. Flask authenticates the token, validates the fields, applies business rules, and saves the record.
5. PostgreSQL checks its constraints and commits the transaction.
6. Flask returns an HTTP status and response body.
7. The frontend handles success or failure.
8. The UI reloads or updates the expense list. Budget progress and reports should reflect the new record.

The UI must not show “saved successfully” merely because the user clicked Save. It should wait for a successful server response. If the network fails, the UI should say that the operation could not be confirmed.

## 6. Why use a shared API service?

The shared API service in services/api.js uses /api as its default base URL unless VITE_API_BASE_URL overrides it. It sets JSON request headers, reads the token from browser local storage, adds an Authorization: Bearer header when a token exists, calls fetch, handles unsuccessful HTTP responses, handles 204 No Content, and parses JSON responses.

Centralising this behaviour avoids every page inventing its own token handling and error parsing. If the request conventions change, one shared service is easier to update.

Centralisation does not make the service infallible. A network failure may occur before an HTTP response exists. A server may return a non-JSON error page. A token may expire. The helper responds to HTTP 401 by removing the stored token and dispatching an unauthorised event. The authentication layer should respond cleanly, and private data should not remain presented as if the session were valid.

The /api path matters because the browser does not directly connect to PostgreSQL or need to know the internal Docker hostname of the backend. Nginx routes public API requests to Flask. The browser can use one public origin while Nginx routes traffic internally.

## 7. Local storage: useful but limited

Browser local storage is a key-value store inside one browser profile. The project uses it to remember the theme preference and currently stores the access token under the key token.

Local storage is not PostgreSQL. It is not shared automatically across devices, and it is not a safe place for a high-value secret against browser script compromise. The app loads core expense-related data through the API; theme preference is a browser-side preference. Do not claim an expense is permanently saved just because it appears in React state or local storage.

Because the token is stored in local storage, a successful cross-site scripting attack could potentially read it. A stronger design may use a properly configured HttpOnly, Secure, SameSite cookie, with appropriate CSRF protections where required. Do not claim the current application uses that alternative unless it has actually been implemented.

## 8. Validation and error states

Frontend validation improves usability. It can highlight a missing field, reject an obviously invalid amount, or tell the user which date format is expected. But frontend validation is not a security boundary. A person can bypass React and call the API directly with curl or another client.

Use three layers:

- **Frontend:** fast feedback and clearer forms.
- **Backend:** trusted business validation and meaningful HTTP responses.
- **Database:** final integrity rules such as positive amounts, foreign keys, non-null fields, and uniqueness where defined.

The UI also needs loading, empty, error, and success states. An empty state means there are no records to show. An error state means the application could not retrieve or process the records. They are not interchangeable. If a failed request is silently treated as an empty array, a user may think their expenses disappeared.

## 9. Charts and financial calculations

A chart is a view of data. Recharts can display bars or category distributions, but it cannot repair incorrect input data.

For a monthly report, the application needs a precise definition of the month, the date range included, how deleted or uncategorised records are treated, and how monetary values are rounded. A common date-range design includes the first day of the month and excludes the first day of the following month. That avoids ambiguity about the final second of the month.

Never answer “Recharts calculates the totals.” The charting library renders supplied values. The backend query or tested calculation defines those values. Compare charts against known sample transactions and expected totals.

## 10. Responsive design and accessibility

Responsive design adapts the interface to different screen sizes. It is not simply shrinking a desktop page. Buttons must remain usable, tables may need horizontal scrolling or a mobile layout, and important actions should not disappear.

Accessibility includes meaningful labels, keyboard access, visible focus, adequate contrast, and text alternatives for meaningful icons. A chart should have a readable title or textual summary so its meaning is not available only through colour.

A feature that technically exists but cannot be used on a phone or with a keyboard is not fully usable.

## 11. Connections to other sections

- **Backend:** accepts requests, validates them again, authenticates users, and returns structured responses.
- **Database:** persists records and enforces integrity; the browser must not connect directly to PostgreSQL.
- **Security:** the frontend sends credentials/tokens but cannot enforce server-side ownership rules.
- **Testing:** component behaviour, API error handling, and complete user journeys need testing; a successful build alone is not proof of correct behaviour.
- **Nginx:** routes /api requests to Flask and other paths to the frontend server.
- **Docker:** packages the frontend build and the server that serves its static assets.
- **CI/CD and AWS:** the source is built into a deployable artifact or image, then the deployed version must be verified.

## 12. Difficult defence questions and answers

**Q1. If React already has an expense in state, why do we need a backend?**  
React state exists in the browser. It can be edited, lost on reload, and is not a trusted source. The backend applies rules and saves data centrally so it can be retrieved later and protected by authentication and database constraints.

**Q2. Why not let the browser connect directly to PostgreSQL?**  
That would expose database access details and credentials to an untrusted client, bypass backend validation and authorisation, and tightly couple the interface to the database schema. The browser calls a controlled API; only the backend uses database credentials.

**Q3. What is the difference between rendering and persistence?**  
Rendering displays a value. Persistence stores it in a durable data store so it can be retrieved later. A successful-looking screen does not prove persistence. Verify the API response and retrieve the record again.

**Q4. Why validate on both frontend and backend?**  
Frontend validation gives immediate feedback but can be bypassed. Backend validation protects the system regardless of the client. Database constraints protect core invariants if application code misses a check.

**Q5. What happens when a token expires?**  
The API can return 401. The shared service removes the stored token and dispatches an unauthorised event. The authentication layer should clear the user state and require login again. Verify the listener and ensure private data is not shown as if the session were still valid.

**Q6. Does a green CI build prove the frontend works?**  
No. A build proves that the source bundled under that job's conditions. Linting checks selected code-quality rules. Neither proves a user can register, create an expense, see correct totals, or export a file. Functional tests are needed.

**Q7. Is local storage the database?**  
No. It is browser-local key-value storage. This app uses it for theme preference and the current token. Core expense-related data is fetched through Flask. Browser state is not the authoritative financial record.

**Q8. If the chart total is wrong, which component is responsible?**  
Trace the value. Check the raw API response, month/category filters, frontend transformations, and chart data mapping. Use known test transactions and expected totals to isolate the fault.

**Q9. What is the difference between props and state?**  
Props are inputs passed from a parent. State is information managed by a component or hook that can change over time. Both affect rendering, but ownership and update paths differ.

**Q10. Why use a shared API service?**  
It centralises the base URL, token attachment, HTTP handling, response parsing, and error behaviour. This reduces duplication but does not eliminate network failures or the need to verify each endpoint contract.

**Q11. Does hiding a Delete button protect a record?**  
No. The browser can be manipulated and API requests can be sent directly. The backend must authenticate the caller and authorise the action.

**Q12. How would you prove the frontend works on the live AWS application?**  
Open the deployed URL, use the browser Network panel to observe a public /api request, verify the response, create a test record, reload, and confirm it persists. Record the deployed version/commit and keep screenshots or logs with secrets and personal data removed.

**Q13. What is a race condition in frontend loading?**  
Two requests can finish in a different order from the order in which they began. If the older response overwrites newer state, the UI may show stale data. Cancellation, request sequencing, or careful state management can prevent this where relevant.

**Q14. Why is an empty list not the same as a failed request?**  
An empty list is a valid response containing no records. A failed request means the client does not know the current server data. Treating failure as empty can mislead users into thinking their records were deleted.

## 13. Frontend defence checklist

Be able to locate App.jsx, Auth.jsx, services/api.js, the transaction form, budget screen, report screen, and chart components. Demonstrate one complete flow from a user action through the API and back to the rendered result. Explain the difference between a visual component, business logic, a server response, and a database record. Do not claim that a visual element proves the whole feature is complete.
