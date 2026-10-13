# Session 1 — Frontend Development and React

## 1. What the frontend is

The frontend is the part of the Expense Tracker that a person sees and uses in a browser. It includes the page layout, navigation, forms, buttons, transaction lists, budget indicators, charts, messages, and the interaction that happens after a user clicks something.

Imagine a restaurant. The frontend is like the menu and the waiter who receives the order. It helps the customer express what they want and shows the result. The kitchen and store room are different parts of the system. In a web application, the backend validates and processes requests, and the database stores persistent records. A good-looking page is not proof that data has been saved safely.

The project frontend uses React, JavaScript, Vite, Tailwind CSS, Lucide React icons, and Recharts. The documented UI includes dashboard summaries, income and expense pages, budgets, categories, a unified transaction ledger, and monthly reports.

## 2. What React contributes

React is a JavaScript library for building user interfaces from components. Instead of writing one enormous page, a developer divides the interface into smaller pieces that can be understood and reused.

A **component** is a reusable piece of UI. Examples include a navigation sidebar, a summary card, a budget progress indicator, a confirmation dialog, or a transaction row. A component usually receives information, renders some UI, and may respond to user events.

A **parent component** can pass information to a child through **props**. Props are inputs. The child should not silently change the parent's data; it should notify the parent through a callback when the user performs an action.

**State** is data that can change while the application is running. For example, the currently selected month, text typed into a search box, whether a modal is open, or the transactions currently loaded. When React state changes, React can render the affected UI again.

A useful distinction:
- Props answer, “What information did the parent give this component?”
- State answers, “What changeable information does this component need to remember?”
- A normal local variable does not automatically make React re-render when it changes.

## 3. The React concepts a tutor may probe

### JSX

JSX is a syntax that lets a developer describe UI in JavaScript using markup-like expressions. It is not HTML sent directly to the browser unchanged. The build tool transforms the source into browser-executable JavaScript.

### Hooks

- `useState` stores component state.
- `useEffect` runs side effects after rendering, such as synchronising with browser storage or reacting to an external system.
- `useMemo` can cache a calculated value between renders when its dependencies have not changed. It is an optimisation, not a correctness or persistence mechanism.

Hooks must be used according to React's rules; they should not be called conditionally in a way that changes their order between renders.

### Context

React Context lets a value be made available to components lower in the tree without passing the same prop through every intermediate component. This project documents context providers for authentication and toast notifications. Context is useful for shared UI state, but it is not a database and does not replace backend authorization.

### Event handling

A click handler or form-submit handler responds to a user action. For a form, the handler should read the submitted values, validate them, prevent unintended page reload where appropriate, submit or save the data, then show a result. If an operation fails, the UI should not pretend that it succeeded.

## 4. How a form should work

Consider adding an expense:

1. The user enters an amount, category, date, and optional description.
2. The frontend checks basic input rules and provides understandable feedback.
3. The frontend sends the data to its configured persistence layer.
4. The backend, if used, independently validates the request. Frontend validation is not a security boundary because a user can bypass the UI and send a request directly.
5. The backend commits the valid record to PostgreSQL and returns a response.
6. The frontend checks the response status, updates its displayed data, and shows success or error feedback.
7. If the user refreshes or opens another device, the data should still exist if it was successfully saved to the shared backend database.

The last two steps expose an important distinction in this repository. The frontend README describes state stored in browser `localStorage`, while the Flask backend separately provides database-backed API endpoints. Browser storage and PostgreSQL are not the same data store. Before claiming that an expense submitted in the visible UI is saved to PostgreSQL, verify that the relevant React handler actually calls the API and that the response is used correctly.

## 5. Browser storage versus a backend database

`localStorage` is a key-value store inside one browser profile on one device. It can be useful for preferences, theme selection, or a local prototype. It is not a shared multi-user database, does not enforce server-side permissions, and can be cleared or edited by the user.

PostgreSQL is a server-side database. If the application sends a record to the Flask API and the backend commits it to PostgreSQL, the record can persist across browsers and devices subject to the account's permissions and the database's availability.

A common defense trap is saying, “We use PostgreSQL, so every screen saves to PostgreSQL.” That conclusion does not follow from the existence of PostgreSQL files. Trace the specific UI action to the actual API call, backend route, and database commit.

## 6. Validation, accessibility, and responsive design

Frontend validation improves usability: required fields, readable date formats, sensible error messages, disabled submit buttons while saving, and confirmation before destructive actions. The backend must repeat security- and integrity-critical validation.

Responsive design means the interface adapts to different viewport sizes. It is not just making the page smaller; the developer must ensure that navigation, forms, tables, charts, and modals remain usable on mobile and desktop.

Accessibility includes semantic headings, labels associated with inputs, keyboard operation, visible focus, meaningful button text, and not relying on colour alone to convey whether a budget is over its limit. An icon-only button should have an accessible name.

## 7. What can go wrong in the frontend

- **Stale state:** a displayed total is calculated from an old array after a mutation.
- **Incorrect effect dependencies:** a side effect runs too often or fails to run when its input changes.
- **Duplicate submission:** a slow request allows the user to submit the same expense twice.
- **False success:** the UI displays success before the server confirms the save.
- **Unsafe rendering:** untrusted text is inserted as executable HTML rather than rendered as text.
- **Broken error handling:** a failed API request leaves the screen looking as though data was saved.
- **State and database drift:** browser storage contains a record that the server does not, or vice versa.
- **Missing tests:** a build can pass even though a user journey is functionally broken.

## Important repository detail: two frontend source locations

The repository contains both a root-level `src/` directory and a `frontend/src/` directory. The root `package.json` can build the root Vite app, but the Docker Compose configuration builds the frontend service from `./frontend`. That means the frontend packaged by the Compose deployment is the one under `frontend/`, not automatically the root-level `src/` version.

This matters in a defense: do not point to a component in the root `src/` tree and assume that it is running on AWS. Check the Docker build context, the active workflow, and the deployed image. The root README describes localStorage-backed records, while `frontend/src/App.jsx` includes state loaded through the Flask API. These are different persistence designs. The team should identify which interface is actually deployed and keep the README aligned with it.

**Likely tutor question: “Why do you have two frontend folders, and which one is running?”**  
A strong answer is: “The repository currently contains two frontend source trees, so we verify the actual entry point rather than guessing. Docker Compose builds `./frontend`, so that is the frontend used by that Compose configuration. We should consolidate or clearly label the unused tree and update the README to avoid describing one version while deploying another.”

## 8. How the frontend connects to the other sections

React is responsible for presentation and interaction. The API contract connects it to Flask. Authentication supplies the token needed for protected requests. PostgreSQL is the durable source of truth when the feature is truly backend-connected. Nginx can route browser requests to the frontend or API. Docker packages the frontend. GitHub Actions can lint and build it. AWS hosts the running deployment.

If any link is missing, the application may look complete but fail as a full-stack product. For example, a budget card can render perfectly while displaying browser-only data that is absent from PostgreSQL.

## 9. Capstone and AWS alignment

For a credible demonstration, show the actual frontend in the deployed application, perform a real action, refresh the page, sign in as another user if supported, and show that the first user's records are not exposed. Capture the exact deployed URL and the commit or image version tested. Do not present a local Vite development server as proof of an AWS deployment.

The frontend checks commonly include:
- install dependencies using the lockfile;
- run the configured lint command;
- run the production build;
- test real user journeys in a browser;
- confirm mobile layout and keyboard-accessible controls;
- confirm API failures are displayed honestly.

The package currently lists lint and build scripts but no dedicated frontend test script. Therefore, a green build is useful evidence, but it is not the same as a passed browser end-to-end test suite.

## 10. Tough defense questions and strong answers

**Q1. Why use React instead of writing one HTML page?**  
React supports reusable components and state-driven rendering. That helps keep a multi-page dashboard consistent and makes interactive features easier to organise. It is not automatically better for every website; it is appropriate here because the interface contains multiple interactive views and shared UI.

**Q2. What is the difference between state and props?**  
Props are inputs passed into a component by its parent. State is data the component or its state owner manages over time. Both influence rendering, but they have different ownership and update rules.

**Q3. Does `useEffect` save data to a database?**  
No. `useEffect` is a mechanism for running side effects. It might synchronise with browser storage or call an API if written to do so. Persistence depends on the actual operation and its successful completion.

**Q4. Why is frontend validation not enough?**  
Users can bypass the page and send arbitrary HTTP requests. The backend must validate data and enforce permissions because it is the trusted boundary protecting the database.

**Q5. If a record remains after refreshing the page, does that prove PostgreSQL works?**  
No. It might have been read from `localStorage`. To prove PostgreSQL persistence, trace the network request, verify the backend response, and confirm the record is retrieved from the backend/database in a controlled test.

**Q6. Why use Context?**  
Context makes selected shared values available to multiple components without repeatedly passing props through every layer. It does not replace a server-side user identity or authorization check.

**Q7. What is the difference between a production build and a running production app?**  
A build proves the source can be compiled into deployable assets. It does not prove those assets were deployed, that Nginx serves them, that the API is reachable, or that the app behaves correctly in a browser.

**Q8. How would you debug a dashboard total that is wrong?**  
First identify the displayed number and its source. Inspect the transaction data, filters, date boundaries, and aggregation formula. Compare the frontend calculation with the backend response and database records. Reproduce with a small known dataset and test edge cases such as an empty month.

**Q9. Why not store passwords or trusted authorization decisions in React state?**  
The browser is controlled by the user and its state can be inspected or altered. The frontend may hold temporary display state or an access token according to the design, but the backend must verify authentication and authorization for protected data.

**Q10. What would you improve first?**  
I would first prove and standardise the frontend-to-API integration for each feature, then add repeatable browser tests. A polished screen is not enough if the persistent data flow is incomplete.

## 11. Explain it in one minute

“The frontend is the user's interface. We use React components to divide the dashboard into manageable pieces, state to represent changing UI data, props to pass information, and Context for selected shared concerns such as notifications or authentication state. The frontend must validate input for usability, call the correct persistence layer, and show the server's actual result. Our repository documents localStorage-based frontend state as well as a Flask/PostgreSQL backend, so we must verify that each feature is truly API-connected before claiming database persistence. We validate that through network inspection, backend tests, browser journeys, and a deployed demonstration.”
