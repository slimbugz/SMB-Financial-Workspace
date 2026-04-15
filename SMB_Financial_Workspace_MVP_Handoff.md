# SMB Financial Workspace — MVP Build Handoff

**Version:** 1.0  
**Date:** April 2026  
**Prepared by:** Oluwaseun Durojaiye, BeadsGuy LLC  
**Audience:** Designer (Google Stitch) + Fullstack Developer (Claude Code)

---

## How to Read This Document

This document is the single source of truth for building the SMB Financial Workspace MVP. It is self-contained — you do not need the PRD to build from this doc. Each feature section covers what to design, what to build, which screens are involved, the API surface, and the acceptance criteria that define done.

**Designer:** Use the Screens and Design Notes sections per feature. Screen IDs (SCR-XX) map to the full screen inventory at the end of this document.

**Developer:** Use the Dev Notes and API Endpoints sections per feature. All endpoints are prefixed with `/api/v1`. Authentication uses Bearer tokens carrying `{ userId, activeWorkspaceId, roleInWorkspace }`. Every permission check is workspace-scoped.

---

## Product Overview

The SMB Financial Workspace is a browser-based financial operations tool for small and micro business owners globally. It replaces the pattern of downloading a one-off Excel spreadsheet with a persistent, identity-aware workspace where the business profile flows into every module. The four MVP modules are Cash Book, Invoice Generator, P&L Statement, and Break Even Analysis, unified by a Dashboard and underpinned by multi-user role management.

**Target users:** Owner-Operators (primary), Bookkeepers/Accountants, Business Partners, and read-only Viewers.

**Revenue model:** Free tier (full module access, no PDF export). Paid tier: USD 5 per PDF export or USD 10/month for unlimited exports. Paystack available for NGN-currency workspaces.

---

## Architecture Notes (Read First)

**Multi-workspace data model — REQ-ARCH-01**  
Even though only one workspace is visible per user at MVP, the database must model user-to-workspace as many-to-many from day one.

- Database requires a `workspace_members` join table: `user_id`, `workspace_id`, `role`, `invited_at`, `accepted_at`
- Session tokens carry `{ userId, activeWorkspaceId, roleInWorkspace }`
- Every API permission check is workspace-scoped using `activeWorkspaceId`
- At MVP, users default to their single workspace on login with no switcher UI shown

This is a non-negotiable architectural constraint. Do not use a one-to-one user-workspace model.

**Currency**  
The workspace base currency is set during onboarding. It propagates to every monetary display across all modules. Supported currencies at launch: NGN, USD, GBP, EUR, CAD, AUD, KES, GHS, ZAR (and others).

**Tech stack**  
No hard requirement. Recommended: React + Tailwind CSS (frontend), Node.js or equivalent (backend), PostgreSQL or equivalent (database). Must be deployable as a web app on a subdomain.

---

## Must Have Features

---

### 1. Authentication (M-01)

**Description**  
Standard email and password auth with verification, reset, and session management. No social login at MVP.

**Primary use case**  
A new business owner signs up, verifies their email, and gains access to their workspace. An existing user returns, logs in, and picks up where they left off.

**Screens**  
SCR-02 Registration, SCR-03 Email Verification Pending, SCR-22 Login, SCR-23 Forgot Password, SCR-24 Reset Password

**Flows**  
Flow 1 (new user registration), Flow 2 (returning user login and password reset)

**Design notes**  
- Registration: email field, password field with strength indicator (weak/fair/strong), ToS acknowledgment checkbox, Submit CTA
- Login: email, password, Forgot Password link, Submit CTA, link to Register
- Forgot Password: single email field, success confirmation state
- Reset Password: new password + confirm password, strength indicator
- Email Verification Pending: confirmation message, resend link (60s cooldown), spam folder guidance
- All auth screens: clean, centered layout. No sidebar or navigation. Logo at top.
- Error states: inline below the relevant field. Never a separate error page.

**Dev notes**  
- Password requirements: minimum 8 characters, at least one number, at least one special character
- Verification link expires after 24 hours. Reset link expires after 1 hour.
- Sessions expire after 30 days of inactivity. On expiry, session token is invalidated and user is redirected to login.
- On logout, session token is invalidated immediately server-side.
- Do not indicate whether the email or password was wrong on login failure (ERR-AUTH-03).

**API endpoints**  
| Method | Endpoint | Auth | Notes |
|--------|----------|------|-------|
| POST | /auth/register | No | email, password |
| POST | /auth/verify-email | No | token |
| POST | /auth/login | No | email, password → returns accessToken, refreshToken, workspaceContext |
| POST | /auth/logout | Yes | invalidates token |
| POST | /auth/refresh | No | refreshToken → new accessToken |
| POST | /auth/forgot-password | No | email |
| POST | /auth/reset-password | No | token, newPassword |

**Acceptance criteria**  
- Given a new email and valid password, account is created and verification email arrives within 60 seconds
- Given a duplicate email, registration is blocked with an inline error and no duplicate account is created
- Given an expired reset link, system shows "link expired" and offers a new request — no silent failure

---

### 2. Business Workspace and Profile (M-02)

**Description**  
On first login, the Owner creates a workspace by completing a three-step business profile. The profile data (business name, currency, logo) propagates to every module automatically. Owners and Admins can edit the profile at any time from Settings.

**Primary use case**  
A new Owner completes onboarding, enters their business name, selects NGN as their base currency, uploads their logo, and arrives at a Dashboard that already knows their business context.

**Screens**  
SCR-04 Onboarding Step 1, SCR-05 Onboarding Step 2, SCR-06 Onboarding Step 3, SCR-18 Settings — Business Profile

**Flows**  
Flow 1 (new user registration and onboarding), steps 4 through 6

**Design notes**  
- Three-step onboarding wizard with a persistent progress bar (1/3, 2/3, 3/3) and Back/Next navigation
- Step 1: Business name (required), business type dropdown (Service/Product/Mixed), industry dropdown (required)
- Step 2: Base currency dropdown (required), fiscal year start month dropdown (required), business address fields
- Step 3: Logo upload (optional, max 2MB, shows preview on upload), tax/registration ID field (optional), Skip and Finish buttons
- Incomplete profile: show a persistent prompt on the Dashboard directing the user back to complete their profile
- Settings — Business Profile: mirrors the onboarding fields in a single editable form with a Save Changes CTA

**Dev notes**  
- Profile is stored at the workspace level, not user level
- Currency selection on step 2 must propagate immediately to all monetary displays — store as ISO code (NGN, USD, GBP, etc.)
- Logo: accept JPG and PNG, max 2MB, store in object storage, serve via CDN URL
- If onboarding is abandoned after step 2, profile is marked `incomplete`. On next login, user is prompted to finish before accessing modules.
- Required fields: business name, type, industry, base currency, fiscal year start

**API endpoints**  
| Method | Endpoint | Auth | Notes |
|--------|----------|------|-------|
| POST | /workspaces | Yes | Creates workspace with profile |
| GET | /workspaces/:id | Yes | Returns workspace profile |
| PUT | /workspaces/:id | Yes (Admin+) | Updates any profile field |
| DELETE | /workspaces/:id | Yes (Owner) | Schedules deletion |

**Acceptance criteria**  
- Given all required fields completed on onboarding, Dashboard loads with business name in the header and currency symbol applied to all monetary fields
- Given a logo file over 2MB, upload is rejected with an inline error and onboarding continues
- Given a Viewer on Settings, Edit Profile is not visible and a direct URL attempt returns permission denied

---

### 3. Role-Based Access Control and Team Invitations (M-03, M-04)

**Description**  
Four roles: Owner (full access, one per workspace), Admin (management access, no billing), Editor (can create and edit data, no team management), Viewer (read-only). Owners and Admins can invite members by email. Ownership can be transferred.

**Primary use case**  
An Owner invites their bookkeeper as an Editor. The bookkeeper accepts, logs in, and can enter Cash Book transactions and generate reports — but cannot touch billing, team settings, or delete the workspace.

**Screens**  
SCR-19 Settings — Team Members, SCR-25 Accept Invitation

**Design notes**  
- Settings — Team Members: members list (name, email, role badge, join date, status), Invite Member CTA at top right
- Invite flow: modal with email field and role dropdown (Admin/Editor/Viewer only — Owner not available). Send Invitation button.
- Pending invitations panel: shows sent invites with status (Pending, Expired), option to resend
- Each member row: role change dropdown (inline), Remove button (Owner only)
- Accept Invitation screen: workspace name, who invited them, role assigned. Two CTAs: Register (new user) or Log In (existing user). Expired state if link is stale.
- Role badge design: Owner (navy), Admin (slate), Editor (amber), Viewer (gray)

**Dev notes**  
- Invitation links expire after 72 hours. Store expiry timestamp at creation.
- Role enforcement is server-side on every request — never rely on frontend-only hiding.
- Owner role cannot be assigned via invitation; it can only be transferred.
- When an Owner transfers ownership: original Owner role changes to Admin, selected user becomes Owner.
- Owner cannot delete workspace while other members exist — block at API level, not just UI.
- Viewer permissions: read-only on all data endpoints. Write attempts return 403.

**API endpoints**  
| Method | Endpoint | Auth | Notes |
|--------|----------|------|-------|
| GET | /workspaces/:id/members | Yes (Admin+) | Returns members list |
| POST | /workspaces/:id/invitations | Yes (Admin+) | email, role |
| PUT | /workspaces/:id/members/:uid | Yes (Admin+) | Change role |
| DELETE | /workspaces/:id/members/:uid | Yes (Owner) | Remove member |
| POST | /workspaces/:id/transfer-ownership | Yes (Owner) | newOwnerUserId |

**Acceptance criteria**  
- Given an Editor accessing Team Member settings via direct URL, response is 403
- Given an invitation link older than 72 hours, system shows "Invitation expired" and the Owner must resend
- Given an Owner with active team members attempting to delete the workspace, deletion is blocked with a clear message listing required steps

---

### 4. Dashboard (M-09, M-17)

**Description**  
The home screen after login. Displays four financial summary cards for the current month, a module launcher grid, and a recent activity feed showing the last 10 workspace actions.

**Primary use case**  
An Owner logs in on a Monday morning and immediately sees this month's income, expenses, net position, and outstanding invoices — without opening any module. They click Cash Book to add a weekend transaction.

**Screens**  
SCR-07 Dashboard

**Design notes**  
- Greeting bar at top: business name (from profile), user's first name, current fiscal period label
- Four summary cards in a 2x2 grid (or 4-column row on desktop): This Month Income, This Month Expenses, Net Position, Outstanding Invoices
- Each card: label, large number in workspace currency, small trend indicator (optional at MVP)
- Cards with no data: show a prompt ("Start logging income" etc.) with a link to activate the relevant module — not zero
- Module launcher grid: 4 tiles for Cash Book, Invoice Generator, P&L Statement, Break Even Analysis. Each shows module name, a brief status (e.g., "12 entries this month", "3 invoices open"), and a launch CTA.
- Activity feed: last 10 workspace actions in reverse chronological order. Each item: action description, who did it, timestamp. E.g., "Invoice INV-004 created by [name] — 2 hours ago"
- Mobile: stack cards vertically, module tiles in a 2-column grid

**Dev notes**  
- Summary card data is computed from Cash Book entries for the current calendar month and from open invoices
- Activity log: write an entry on every significant action (entry created/edited/deleted, invoice created/sent/paid, report generated, member invited). Store `userId`, `workspaceId`, `action`, `entityType`, `entityId`, `timestamp`. Return last 10 per workspace.
- Dashboard is the first page after login and after onboarding completion
- Viewer role: all cards and module tiles visible, no edit controls anywhere on page

**API endpoints**  
| Method | Endpoint | Auth | Notes |
|--------|----------|------|-------|
| GET | /workspaces/:id/cashbook/summary | Yes | query: month, year |
| GET | /workspaces/:id/invoices | Yes | query: status=open |

**Acceptance criteria**  
- Given a Cash Book with entries in the current month, dashboard cards show correct totals on load
- Given no invoices created, outstanding invoices card shows a prompt — not zero
- Activity feed shows the last 10 actions in reverse chronological order with actor names

---

### 5. Cash Book (M-05)

**Description**  
A chronological ledger for income and expense entries. Each entry has a type, date, description, category, and amount. The module displays a running balance after each entry and monthly aggregates by category.

**Primary use case**  
A business owner receives payment for a job and opens Cash Book on their phone to log it in 30 seconds: Income, today's date, "Website project — Adewale Ltd", Revenue category, 150,000 NGN. The running balance and monthly summary update immediately.

**Screens**  
SCR-08 Cash Book — List View, SCR-09 Cash Book — Add/Edit Entry

**Flows**  
Flow 4 (Cash Book entry)

**Design notes**  
- List View: header with period filter (default: current month), category filter, type toggle (All/Income/Expense). Add Entry CTA (Editor and above only — hide for Viewers).
- Entry list: each row shows date, description, category badge, amount (income in green, expense in red), running balance. Most recent entry at top.
- Monthly summary panel (sidebar on desktop, collapsible on mobile): total income, total expenses, net position, category breakdown bar chart.
- Add/Edit Entry form: Income/Expense type toggle (prominent, at top), date picker, description text field, category dropdown, amount field, optional notes field, recurring toggle. Save and Cancel CTAs.
- Recurring toggle: reveals frequency selector (Daily/Weekly/Monthly).
- Empty state: illustration or icon with "No entries yet. Add your first transaction." CTA.

**Dev notes**  
- Running balance: computed as cumulative sum of all entries ordered by date ascending. Recalculate on every entry add, edit, or delete.
- Categories: seed default categories at workspace creation (Revenue, Cost of Sales, Operating Expenses, Other Income, Other Expenses). Custom categories added via Settings (Should Have S-06).
- Amount stored as integer (smallest currency unit) or decimal — pick one and apply consistently across all modules.
- Recurring entries: create a scheduled job that generates the next entry on the recurrence date. On delete of a recurring entry, ask user: this occurrence only, or all future occurrences.
- Future-dated entries are permitted with a soft warning indicator on the row.
- Entry deletion from within a finalized P&L snapshot: show a warning modal before allowing deletion.

**API endpoints**  
| Method | Endpoint | Auth | Notes |
|--------|----------|------|-------|
| GET | /workspaces/:id/cashbook/entries | Yes | query: startDate, endDate, category, type, page, limit |
| POST | /workspaces/:id/cashbook/entries | Yes (Editor+) | type, date, description, categoryId, amount, notes, isRecurring, recurrenceFrequency |
| PUT | /workspaces/:id/cashbook/entries/:eid | Yes (Editor+) | Any entry fields |
| DELETE | /workspaces/:id/cashbook/entries/:eid | Yes (Editor+) | query: deleteScope (single or all) |
| GET | /workspaces/:id/cashbook/summary | Yes | query: month, year |
| GET | /workspaces/:id/cashbook/categories | Yes | Returns categories list |
| POST | /workspaces/:id/cashbook/categories | Yes (Editor+) | name, type |

**Acceptance criteria**  
- Given an entry saved, the running balance updates on the list view without a page reload
- Given an amount of zero submitted, save is blocked with an inline validation error
- Given a Viewer clicking the Add Entry button, the button is not rendered; a direct POST to the endpoint returns 403

---

### 6. Invoice Generator (M-06)

**Description**  
Create, edit, preview, and track invoices. Business profile data auto-fills the header. Line items calculate subtotal, tax, and discount in real time. Invoices move through a status lifecycle: Draft, Sent, Paid, Overdue.

**Primary use case**  
A freelance designer finishes a project and needs to invoice a client before end of day. They open Invoice Generator, confirm their pre-filled business details, enter the client's name, add three line items, apply 7.5% VAT, preview the formatted invoice, export it as a PDF (paying USD 5), and mark it Sent.

**Screens**  
SCR-10 Invoice — List View, SCR-11 Invoice — Create/Edit Form, SCR-12 Invoice — Preview, SCR-13 PDF Export Payment Gate

**Flows**  
Flow 5 (Invoice creation and PDF export)

**Design notes**  
- List View: invoice rows with invoice number, client name, issue date, due date, amount, status badge (Draft/gray, Sent/blue, Paid/green, Overdue/red). Filter by status. Search by client name. Outstanding total summary at top. New Invoice CTA.
- Create/Edit Form: two-column layout on desktop (form left, running total panel right). Form sections: Business header (read-only from profile), Client Details (name, email, address), Invoice Details (number, issue date, due date), Line Items table (description, quantity, unit price per row with add/remove row controls), Tax Rate %, Discount %, Notes textarea.
- Running total panel: Subtotal, Tax Amount, Discount Amount, Total — all update in real time as user types.
- Line item rows: add new row button at bottom of table, delete row icon per row.
- Preview screen: formatted invoice layout exactly matching what will be exported. Back to Edit, Export PDF, and Mark as Sent CTAs.
- Export PDF: triggers payment modal (SCR-13). Two options: Pay USD 5 for this export, or USD 10/month for unlimited. Stripe card widget (or Paystack for NGN workspaces). Cancel button.
- Invoice number: auto-generates as INV-001, INV-002 etc. User can override. Show warning if duplicate.

**Dev notes**  
- Calculations: `subtotal = sum(qty * unitPrice per line item)`, `taxAmount = subtotal * (taxRate/100)`, `discountAmount = subtotal * (discountRate/100)`, `total = subtotal + taxAmount - discountAmount`. All computed server-side on save; client-side preview for real-time feedback only.
- Auto-save as Draft: trigger on navigation away if the form has unsaved changes.
- Status lifecycle: Draft → Sent (manual) → Paid (manual) or Overdue (automated). Overdue triggered by daily job: if `dueDate < today AND status == Sent`, update status to Overdue.
- PDF generation: generate server-side (Puppeteer or equivalent) using a styled HTML template. Release download URL only after payment webhook confirmation.
- Only Draft invoices can be deleted. Sent/Paid/Overdue invoices get a Void option instead.
- Invoice number auto-increment: track `lastInvoiceNumber` per workspace. Increment on each new invoice.

**API endpoints**  
| Method | Endpoint | Auth | Notes |
|--------|----------|------|-------|
| GET | /workspaces/:id/invoices | Yes | query: status, page, limit |
| POST | /workspaces/:id/invoices | Yes (Editor+) | clientName, clientEmail, clientAddress, dueDate, lineItems[], taxRate, discount, notes |
| GET | /workspaces/:id/invoices/:inv | Yes | Full invoice object |
| PUT | /workspaces/:id/invoices/:inv | Yes (Editor+) | Any invoice fields |
| DELETE | /workspaces/:id/invoices/:inv | Yes (Editor+) | Draft only |
| POST | /workspaces/:id/invoices/:inv/export | Yes (Editor+) | Returns pdfUrl or 402 paymentRequired |
| POST | /workspaces/:id/invoices/:inv/send | Yes (Editor+) | Sets status to Sent |

**Acceptance criteria**  
- Given line items with quantities and prices, subtotal, tax, and total update in real time as values are typed
- Given no line items, the Export PDF button is disabled with a tooltip
- Given a successful payment webhook, the PDF URL is returned and the invoice status updates to Sent

---

### 7. P&L Statement (M-07)

**Description**  
Generates a formatted Profit and Loss statement for a user-selected period. Can auto-populate from Cash Book data or accept manual entry. Output shows revenue by category, COGS, gross profit, operating expenses by category, operating income, and net income. Saves named snapshots for historical reference.

**Primary use case**  
A business owner applies for a bank loan and needs to show six months of P&L. They open the P&L module, select the date range, confirm the auto-populated revenue and expense figures from Cash Book, add COGS manually, generate the report, and export it as a PDF to attach to the application.

**Screens**  
SCR-14 P&L — Period Selector, SCR-15 P&L — Report View

**Flows**  
Flow 6 (P&L statement generation)

**Design notes**  
- Period Selector: prominent period toggle (Month/Quarter/Custom). Month selector or custom date range pickers. Auto-populate from Cash Book toggle (on by default). Manual Entry fallback if no Cash Book data. Generate Report CTA.
- Report View: structured table with P&L sections (Revenue, COGS, Gross Profit, Operating Expenses, Operating Income, Net Income). Each section has category rows and a subtotal row. Figures in workspace currency.
- Save Snapshot CTA: opens a name input modal. Saved snapshots accessible from a list above the period selector.
- Export PDF: same payment gate as Invoice export (SCR-13).
- Viewer role: can view saved snapshots, cannot generate new reports.
- Empty state for a period with no Cash Book data: clear explanation with option to switch to manual entry.

**Dev notes**  
- Auto-populate logic: pull Cash Book entries for the selected date range. Map categories to P&L sections (Revenue categories → Revenue section; expense categories → Operating Expenses section).
- Manual entry mode: user inputs figures directly into the P&L table cells.
- Snapshot: store the complete P&L data object with the label and period. Snapshots are immutable once saved.
- Period comparison (Should Have S-08): design the report view to accommodate a prior period column from the start — render dashes if no data.

**API endpoints**  
| Method | Endpoint | Auth | Notes |
|--------|----------|------|-------|
| POST | /workspaces/:id/pl/generate | Yes (Editor+) | startDate, endDate, mode (auto or manual), manualEntries[] |
| GET | /workspaces/:id/pl/snapshots | Yes | Returns snapshots list |
| POST | /workspaces/:id/pl/snapshots | Yes (Editor+) | label, reportData |
| GET | /workspaces/:id/pl/snapshots/:sid | Yes | Full snapshot |
| DELETE | /workspaces/:id/pl/snapshots/:sid | Yes (Admin+) | |

**Acceptance criteria**  
- Given Cash Book data for the selected period, all revenue and expense categories populate the correct P&L sections
- Given no Cash Book data for the period, manual entry mode is offered without attempting auto-population
- Given a saved snapshot, it is accessible from the list and renders correctly when reopened

---

### 8. Break Even Analysis (M-08)

**Description**  
Calculates the revenue threshold at which the business covers all costs. User inputs fixed costs, variable cost per unit, and selling price per unit. Output includes break-even units, break-even revenue, contribution margin, and margin of safety — with a visual chart. Analysis runs can be labeled and saved.

**Primary use case**  
A product business owner is deciding whether to expand a product line. They enter their fixed overheads (rent, salaries: 400,000 NGN), variable cost per unit (1,200 NGN), and selling price (3,000 NGN). The calculation shows they need to sell 223 units per month to break even, and their current sales of 310 units give them a 28% margin of safety.

**Screens**  
SCR-16 Break Even — Input Form, SCR-17 Break Even — Results View

**Flows**  
Flow 7 (Break even analysis)

**Design notes**  
- Input Form: three clearly labeled input fields — Total Fixed Costs, Variable Cost Per Unit, Selling Price Per Unit. Optional label field. Calculate CTA (disabled until all three required fields are populated). Clear helper text under each field explaining what to enter.
- Results View: four result cards (Break Even Units, Break Even Revenue, Contribution Margin Per Unit, Margin of Safety). Below the cards: break-even chart — x-axis is units, y-axis is currency. Three lines: fixed cost (horizontal), total cost (diagonal), revenue (steeper diagonal). The intersection point is marked with a dot and a label.
- Save Run CTA and Export PDF CTA. Recalculate button returns to Input Form pre-filled with previous values.
- Error state: if selling price equals or is less than variable cost, the Calculate button stays disabled and an inline error explains that selling price must exceed variable cost.

**Dev notes**  
- Formulas:
  - `contributionMarginPerUnit = sellingPricePerUnit - variableCostPerUnit`
  - `breakevenUnits = fixedCosts / contributionMarginPerUnit` (round up to nearest whole unit)
  - `breakevenRevenue = breakevenUnits * sellingPricePerUnit`
  - `marginOfSafety` requires the user to have a current sales volume — at MVP this is informational; show the formula and result if user optionally provides current units sold.
- Chart: render client-side using a charting library (Recharts or Chart.js recommended).
- Saved runs: store inputs and computed outputs together. Label is optional; default to the date if blank.

**API endpoints**  
| Method | Endpoint | Auth | Notes |
|--------|----------|------|-------|
| POST | /workspaces/:id/breakeven/calculate | Yes (Editor+) | fixedCosts, variableCostPerUnit, sellingPricePerUnit |
| GET | /workspaces/:id/breakeven/runs | Yes | Returns saved runs |
| POST | /workspaces/:id/breakeven/runs | Yes (Editor+) | label, inputs and results |
| DELETE | /workspaces/:id/breakeven/runs/:rid | Yes (Editor+) | |

**Acceptance criteria**  
- Given fixed costs 10,000, variable cost per unit 20, selling price 50: break-even units = 333, break-even revenue = 16,667
- Given variable cost equal to or greater than selling price, the Calculate button is disabled and an error is shown
- Given all required fields empty, the Calculate button is disabled

---

### 9. Payments and PDF Export (M-11)

**Description**  
PDF export from Invoice Generator, P&L, and Break Even Analysis is gated behind a payment. Options: USD 5 per export (single) or USD 10/month for unlimited exports. Stripe is the primary processor; Paystack is available for NGN-currency workspaces. Users with an active subscription bypass the payment gate.

**Primary use case**  
A user previews their invoice and clicks Export PDF. A modal appears with two pricing options. They select the per-export option, complete Stripe checkout, return to the app, and the PDF downloads automatically.

**Screens**  
SCR-13 PDF Export — Payment Gate (modal), SCR-20 Settings — Billing

**Design notes**  
- Payment gate: modal overlay (not a new page). Plan selector: two cards side by side — "One-time export: USD 5" and "Unlimited exports: USD 10/month". Selected card gets a highlight border.
- Stripe card widget or Paystack widget renders inline in the modal below the plan selector.
- Cancel button top right of modal. Processing state while payment is in flight.
- After successful payment: modal closes, PDF download triggers automatically.
- Settings — Billing: current plan, billing cycle, next renewal date, export count this billing period, payment history table (date, amount, type), Upgrade and Cancel Subscription CTAs.
- For NGN workspaces: show Paystack as the payment option. For all others: Stripe.

**Dev notes**  
- Never pass card data through the application server. Use Stripe.js (client-side) and Paystack.js (client-side) to tokenize payments.
- Create payment session server-side (Stripe: create checkout session; Paystack: initiate transaction). Return the redirect URL or widget config to the client.
- Webhook handlers for both processors confirm payment before PDF is generated or subscription is activated.
- PDF generation is server-side only. URL is time-limited (expires in 15 minutes after generation).
- Subscription status check: on every PDF export request, check the workspace's active subscription before triggering the payment gate.

**API endpoints**  
| Method | Endpoint | Auth | Notes |
|--------|----------|------|-------|
| POST | /payments/stripe/session | Yes (Editor+) | type (per-export or subscription), resourceId |
| POST | /payments/paystack/initiate | Yes (Editor+) | type, resourceId |
| POST | /payments/webhook/stripe | No (HMAC signed) | Stripe event payload |
| POST | /payments/webhook/paystack | No (HMAC signed) | Paystack event payload |
| GET | /payments/subscription | Yes (Owner) | Returns plan, status, renewsAt, exportsUsed |

**Acceptance criteria**  
- Given a failed payment, no PDF is generated and the user sees an inline retry option
- Given a successful payment webhook, PDF download triggers automatically within 10 seconds
- Given a user with an active monthly subscription, the payment gate is bypassed on export

---

### 10. Data Export and Compliance (M-12, M-13, M-14, M-15)

**Description**  
Owners and Admins can export all workspace data as a JSON file at any time. The app displays a cookie consent banner for users in the EU, EEA, Nigeria, and California. Users can request permanent deletion of all their data. Privacy Policy and Terms of Service are accessible from the footer of every page.

**Primary use case**  
An Owner decides to migrate to a different tool. They go to Settings → Privacy and Data, download their full workspace as JSON, and submit a deletion request. They receive a confirmation email and their account is scheduled for permanent deletion within 30 days.

**Screens**  
SCR-21 Settings — Privacy and Data, SCR-26 Privacy Policy, SCR-27 Terms of Service, SCR-28 Cookie Consent Banner

**Design notes**  
- Settings — Privacy and Data: two clear CTAs — "Export All Data (JSON)" and "Delete Account and All Data". The delete CTA is styled as a destructive action (red border, warning icon). Clicking it opens a confirmation modal that requires the user to type their email address before the action is enabled.
- Cookie consent banner: appears at the bottom of the page on first visit for users in covered jurisdictions. Three options: Accept All, Reject Non-Essential, Manage Preferences. Plain language copy. Link to Privacy Policy.
- Privacy Policy and Terms of Service: static pages, linked from the footer on every screen including pre-authentication pages.
- Footer on all screens: at minimum, links to Privacy Policy and Terms of Service.

**Dev notes**  
- JSON export: include all Cash Book entries, invoices, P&L snapshots, Break Even runs, business profile, team member list (names and roles, no other users' PII), and consent records.
- Cookie banner: detect IP jurisdiction via server-side geolocation or a cookie consent SaaS (e.g., Cookiebot, Osano). Do not set non-essential cookies before consent.
- Deletion: soft-delete immediately (user cannot log in). Schedule permanent deletion job for 30 days later. Send confirmation email within 24 hours of request with completion date.
- Deletion is blocked if the user is the sole Owner and the workspace has other active members. Error message lists required steps.
- Privacy Policy and ToS must be live before launch. Draft content must cover GDPR, NDPR, and CCPA rights and practices.

**API endpoints**  
| Method | Endpoint | Auth | Notes |
|--------|----------|------|-------|
| GET | /workspaces/:id/export | Yes (Admin+) | Returns full workspace JSON |
| DELETE | /users/me | Yes (Owner) | Schedules account and data deletion |

**Acceptance criteria**  
- Given an Owner clicking Export All Data, a JSON file downloads containing all workspace records
- Given a user in the EU on first visit, no analytics cookies are set before the consent banner is acknowledged
- Given a deletion request confirmed with the correct email, a confirmation email is sent within 24 hours and the account is inaccessible immediately

---

### 11. Multi-Currency Support (M-10)

**Description**  
Every workspace has a base currency set during onboarding. The currency symbol and number format apply to all monetary fields across all modules — Cash Book amounts, invoice totals, P&L figures, Break Even inputs and outputs, and Dashboard summary cards.

**Primary use case**  
A Nigerian business owner sets NGN as their base currency. Every monetary field across the entire app displays the NGN symbol with appropriate formatting. No manual formatting required.

**Design notes**  
- Currency symbol and code both visible on monetary inputs (e.g., "NGN" label and "₦" symbol)
- Currency formatting: use the Intl.NumberFormat API for locale-aware number display
- No currency conversion at MVP — single base currency per workspace

**Dev notes**  
- Store currency as ISO 4217 code (NGN, USD, GBP, EUR, CAD, AUD, KES, GHS, ZAR)
- All monetary storage is in the smallest unit of the currency (kobo for NGN, cents for USD, etc.) or use consistent decimal precision — decide and document before building
- Currency symbol lookup table required for display (NGN: ₦, USD: $, GBP: £, EUR: €, etc.)
- Currency change by Owner in Settings propagates to all displays without a data migration — the stored amounts are currency-agnostic integers

**Acceptance criteria**  
- Given NGN selected as base currency, all monetary displays across all modules show the NGN symbol
- Given an Owner changing currency from USD to GBP in Settings, all displays update to GBP on next page load

---

### 12. Mobile-Responsive Design (M-16)

**Description**  
The full application must be functional on mobile browsers from 320px viewport width upward. All modules, forms, tables, and flows must work on a phone.

**Design notes**  
- Breakpoints (recommended): mobile 320–767px, tablet 768–1023px, desktop 1024px+
- Navigation: hamburger menu or bottom tab bar on mobile
- Tables (Cash Book list, Invoice list): horizontal scroll on mobile or convert to card-based layout
- Forms: single column on mobile, full-width inputs
- Charts (Break Even): must be readable at 320px — reduce detail, increase touch targets
- Dashboard: summary cards stack vertically on mobile (2 per row minimum), module launcher tiles in 2-column grid
- PDF export gate modal: full-screen sheet on mobile (not a centered modal)
- Minimum touch target size: 44x44px for all interactive elements

**Dev notes**  
- Use CSS media queries and a mobile-first approach
- Test all flows at 320px, 375px (iPhone SE), 390px (iPhone 14), and 768px (tablet)
- No horizontal scroll at any standard breakpoint (except tables with explicit scroll container)

**Acceptance criteria**  
- Given a 320px viewport, all module forms are fully usable without horizontal scrolling
- Given a mobile browser, all navigation is accessible via touch

---

## Should Have Features

These are important but not launch-blocking. Design and develop these in the sprint immediately following MVP launch, using the same screens and API surface established above.

---

### S-01 — Email Invoice to Client

**Description:** Send an invoice directly from the app to a client's email address.  
**Primary use case:** Owner sends a finalized invoice to a client without leaving the app or copying a PDF to email.  
**Screen:** SCR-12 Invoice Preview — add a Send by Email CTA alongside Export PDF  
**Dev notes:** Send via transactional email service (SendGrid, Resend, or equivalent). Email includes PDF attachment (triggers payment gate if unpaid) and a plain-text link to view. Sets invoice status to Sent on delivery. Requires client email to be set on the invoice.

---

### S-02 — Auto Overdue Invoice Status

**Description:** Invoices automatically move to Overdue status when the due date passes and status is not Paid.  
**Primary use case:** Owner sees an accurate outstanding invoice count on the Dashboard without manually tracking due dates.  
**Dev notes:** Daily cron job. Query: `WHERE dueDate < today AND status = 'Sent'`. Update status to Overdue. Write activity log entry. No notification at MVP — that is C-01.

---

### S-03 — Recurring Cash Book Entries

**Description:** Cash Book entries can be set as recurring (daily, weekly, or monthly). The system auto-creates new entries on each recurrence.  
**Primary use case:** Owner sets monthly rent as a recurring expense. It auto-logs on the 1st of each month without any action required.  
**Screen:** SCR-09 Cash Book Add/Edit Entry — recurring toggle and frequency selector  
**Dev notes:** Store recurrence schedule alongside the entry. Scheduled job creates the next entry on the recurrence date. On delete: modal asks whether to delete this occurrence only or all future occurrences.

---

### S-04 — Cash Book Auto-Populates P&L

**Description:** P&L generation mode that pulls directly from Cash Book entries for the selected period.  
**Primary use case:** Owner generates a P&L for Q1 and all their logged income and expense categories appear pre-filled.  
**Dev notes:** Map Cash Book category types to P&L sections at workspace setup: revenue categories → Revenue section, expense categories → Operating Expenses section. COGS entries must be manually confirmed or added by the user.

---

### S-05 — JSON Import

**Description:** Owners and Admins can re-import a previously exported JSON file to restore workspace data.  
**Primary use case:** Owner accidentally deleted entries and has a recent backup JSON. They re-import to restore the data.  
**Screen:** SCR-21 Settings — Privacy and Data — add Import Data CTA  
**Dev notes:** Validate JSON schema matches export format. If import would overwrite existing records, show a warning modal with record counts before proceeding. On success, show a summary of records created, skipped, and overwritten. On malformed JSON, reject entirely with an error — no partial imports.

---

### S-06 — Custom Income and Expense Categories

**Description:** Owners and Admins can create custom categories for Cash Book entries beyond the default set.  
**Primary use case:** A photography studio creates a "Studio Hire" income category and a "Equipment" expense category specific to their business.  
**Dev notes:** Custom categories are workspace-scoped and available alongside defaults in the Cash Book category dropdown. Categories marked `isSystem = true` cannot be deleted.

---

### S-07 — Invoice Number Auto-Increment

**Description:** Invoice numbers auto-generate in sequence per workspace (INV-001, INV-002, etc.) and can be manually overridden.  
**Primary use case:** Owner never has to think about invoice numbering — the system handles it.  
**Dev notes:** Store `lastInvoiceNumber` as an integer on the workspace. Increment atomically on each new invoice. Format as INV-XXX (zero-padded to 3 digits; extend as needed). If user overrides with a duplicate: inline warning with a suggestion of the next available number.

---

### S-08 — Period-over-Period P&L Comparison

**Description:** P&L report view shows the prior period alongside the current period for comparison.  
**Primary use case:** Owner reviewing Q2 P&L sees Q1 figures alongside to spot trends.  
**Screen:** SCR-15 P&L Report View — add comparison toggle and prior period column  
**Dev notes:** When comparison is toggled on, fetch the equivalent prior period (e.g., previous month for a monthly view, previous quarter for a quarterly view). If no data exists for the prior period, show dashes and the label "No data for this period".

---

## Screen Inventory Reference

| SCR-ID | Screen Name | Module | Min Access |
|--------|-------------|--------|------------|
| SCR-02 | Registration | Auth | None |
| SCR-03 | Email Verification Pending | Auth | None |
| SCR-04 | Onboarding Step 1 of 3 | Onboarding | Owner |
| SCR-05 | Onboarding Step 2 of 3 | Onboarding | Owner |
| SCR-06 | Onboarding Step 3 of 3 | Onboarding | Owner |
| SCR-07 | Dashboard | Core | All roles |
| SCR-08 | Cash Book — List View | Cash Book | All roles |
| SCR-09 | Cash Book — Add/Edit Entry | Cash Book | Editor+ |
| SCR-10 | Invoice — List View | Invoice | All roles |
| SCR-11 | Invoice — Create/Edit Form | Invoice | Editor+ |
| SCR-12 | Invoice — Preview | Invoice | Editor+ |
| SCR-13 | PDF Export — Payment Gate | Payments | Editor+ |
| SCR-14 | P&L — Period Selector | P&L | Editor+ |
| SCR-15 | P&L — Report View | P&L | All roles |
| SCR-16 | Break Even — Input Form | Break Even | Editor+ |
| SCR-17 | Break Even — Results View | Break Even | All roles |
| SCR-18 | Settings — Business Profile | Settings | Admin+ |
| SCR-19 | Settings — Team Members | Settings | Admin+ |
| SCR-20 | Settings — Billing | Settings | Owner only |
| SCR-21 | Settings — Privacy and Data | Settings | Owner |
| SCR-22 | Login | Auth | None |
| SCR-23 | Forgot Password | Auth | None |
| SCR-24 | Reset Password | Auth | None |
| SCR-25 | Accept Invitation | Auth | None |
| SCR-26 | Privacy Policy | Static | None |
| SCR-27 | Terms of Service | Static | None |
| SCR-28 | Cookie Consent Banner | Overlay | None |

---

## Non-Functional Requirements Summary

| Category | Requirement |
|----------|-------------|
| Performance | Dashboard loads in under 3 seconds on standard broadband |
| Performance | PDF generates and downloads within 10 seconds of payment confirmation |
| Performance | Cash Book balance recalculates in under 500ms after entry save |
| Security | All data in transit encrypted via HTTPS, TLS 1.2 minimum |
| Security | Passwords stored as bcrypt hashes, minimum cost factor 12 |
| Security | CSRF protection on all state-changing operations |
| Security | Card data never passes through the application server |
| Accessibility | WCAG 2.1 Level AA |
| Accessibility | All form fields have programmatically associated labels |
| Accessibility | Minimum 4.5:1 color contrast ratio for normal text |
| Accessibility | All interactive elements are keyboard-navigable |
| Browser support | Chrome, Firefox, Safari, Edge — latest 2 major versions each |
| PWA | Include PWA manifest and service worker from launch |

---

## Compliance Summary

| Regulation | Requirement |
|------------|-------------|
| GDPR | Cookie consent banner for EU/EEA users; right to erasure within 30 days; right to data portability (JSON) |
| NDPR | Cookie consent banner for Nigerian users; same erasure and portability rights |
| CCPA | Cookie consent banner for California users; right to erasure within 45 days |
| All three | Privacy Policy and ToS live before launch; sub-processors listed in Privacy Policy; breach notification within 72 hours |

---

## Open Questions Blocking Build Decisions

| ID | Question | Blocks |
|----|----------|--------|
| OQ-02 | What is the soft-delete grace period before permanent deletion executes? | Backend deletion job design |
| OQ-03 | Is there a GDPR data residency requirement (EU-region hosting for EU user data)? | Infrastructure provider selection |
| OQ-04 | What is the overdue invoice trigger window — 1 day before due, on due date, or after? | S-02 cron job logic |
| OQ-05 | Will subscriptions auto-renew? What is the cancellation and refund policy? | Billing UX and Terms of Service |
| OQ-06 | Is the product name confirmed? | All copy, meta tags, payment descriptor |

