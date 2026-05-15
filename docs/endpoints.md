# CredFlow API Endpoint Requirements

## Diagnosis

This workspace is a frontend-first Next.js app. The dashboard, auth, and onboarding screens are present, but they are powered by mocked arrays and client-side navigation. There are no `app/api` route handlers in the repository, and none of the list views are wired to real data mutations.

That means the app currently has a product shell, not a working debt-management system. The UI implies a backend contract for authentication, customer/debt/payment CRUD, analytics, notifications, settings, audit logging, and export/reporting.

## API Design Notes

Use a versioned JSON REST API with consistent envelopes:

```json
{
  "data": {},
  "meta": {
    "page": 1,
    "pageSize": 20,
    "total": 0
  },
  "error": null
}
```

Recommended cross-cutting requirements:

- JWT or session-cookie auth.
- Tenant-aware scoping for the logged-in business.
- Pagination, sorting, and filtering on every list endpoint.
- Idempotency keys for payment creation and any money-moving action.
- Audit logging for destructive or financial actions.
- Webhook or background job support for reminders, receipts, and scheduled notifications.

## Endpoint Catalog

### Authentication and account access

| Method | Path | Purpose | Notes |
| --- | --- | --- | --- |
| POST | `/api/auth/register` | Create a business account | Used by the register screen. Should create the user, business, and initial session. |
| POST | `/api/auth/login` | Authenticate a user | Used by the login screen. Return session tokens plus onboarding status. |
| POST | `/api/auth/logout` | End the active session | Needed for sidebar/profile logout. |
| POST | `/api/auth/refresh` | Refresh access credentials | Required for long-lived sessions. |
| POST | `/api/auth/forgot-password` | Start password recovery | Used by the forgot-password page. |
| POST | `/api/auth/reset-password` | Finish password recovery | Used by the reset-password page. |
| GET | `/api/auth/me` | Return the current user | Needed by the header, settings, and route guards. |
| PATCH | `/api/auth/me` | Update current user profile | Supports name, phone, avatar, and preferences. |
| GET | `/api/auth/sessions` | List active sessions | Required by the security settings page. |
| DELETE | `/api/auth/sessions/:sessionId` | Revoke a session | Needed for device/session management. |
| POST | `/api/auth/2fa/enable` | Enable 2FA | The settings screen exposes this capability already. |
| POST | `/api/auth/2fa/disable` | Disable 2FA | Pair with verification codes or recovery codes. |

### Business and onboarding

| Method | Path | Purpose | Notes |
| --- | --- | --- | --- |
| GET | `/api/businesses/current` | Load the active business profile | Needed after login and in onboarding. |
| PATCH | `/api/businesses/current` | Update business profile | Business name, industry, size, address, branding. |
| POST | `/api/onboarding/complete` | Save onboarding progress | The onboarding flow currently exists only in local state. |
| GET | `/api/onboarding/status` | Return onboarding completion state | Used to redirect users into or out of onboarding. |
| POST | `/api/onboarding/seed` | Create first customer/debt bootstrap data | Optional, but useful if onboarding auto-creates starter records. |

### Customers

| Method | Path | Purpose | Notes |
| --- | --- | --- | --- |
| GET | `/api/customers` | List customers | Must support `search`, `riskLevel`, `hasOutstandingDebt`, `page`, and `sort`. |
| POST | `/api/customers` | Create a customer | Used by the Add Customer action. |
| GET | `/api/customers/:customerId` | Customer detail record | Required for a proper customer details page. |
| PATCH | `/api/customers/:customerId` | Update customer profile | Contacts, business metadata, risk tags, credit limits. |
| DELETE | `/api/customers/:customerId` | Archive or delete a customer | Prefer soft delete if debts exist. |
| GET | `/api/customers/:customerId/debts` | Debts for one customer | Required for the customer detail view. |
| GET | `/api/customers/:customerId/payments` | Payments for one customer | Required for the customer detail view. |
| GET | `/api/customers/:customerId/notes` | Internal notes and communication history | Needed for follow-up context. |
| POST | `/api/customers/:customerId/notes` | Add a note or call log | Should be audit logged. |
| GET | `/api/customers/:customerId/activity` | Timeline of actions | Useful for detail page traceability. |

### Debts

| Method | Path | Purpose | Notes |
| --- | --- | --- | --- |
| GET | `/api/debts` | List debts | Must support status, risk, customer, and due-date filtering. |
| POST | `/api/debts` | Record a new debt | Used by the Record Debt action and onboarding. |
| GET | `/api/debts/:debtId` | Debt detail record | Required for a debt details page. |
| PATCH | `/api/debts/:debtId` | Update debt terms | Amount, due date, interest, status, notes. |
| DELETE | `/api/debts/:debtId` | Archive or delete debt | Should be restricted and audited. |
| POST | `/api/debts/:debtId/mark-paid` | Close a debt as paid | Often called after a payment is recorded. |
| POST | `/api/debts/:debtId/payments` | Record a payment against a debt | Core money-movement action. |
| POST | `/api/debts/:debtId/reschedule` | Extend or modify due date | Needed for collection operations. |
| POST | `/api/debts/:debtId/waive` | Waive all or part of a debt | Requires permission checks and audit logs. |
| GET | `/api/debts/:debtId/schedule` | Repayment or amortization schedule | Useful if installment support is planned. |
| GET | `/api/debts/:debtId/activity` | Debt event timeline | Needed for the detail page. |

### Payments

| Method | Path | Purpose | Notes |
| --- | --- | --- | --- |
| GET | `/api/payments` | List payments | Must support customer, method, date, and amount filters. |
| POST | `/api/payments` | Record a payment | Should accept idempotency keys and optional debt linkage. |
| GET | `/api/payments/:paymentId` | Payment detail record | Required for a payment details page. |
| PATCH | `/api/payments/:paymentId` | Correct a payment record | Useful for reconciliation and admin edits. |
| DELETE | `/api/payments/:paymentId` | Void or refund a payment | Should usually be a controlled admin action. |
| GET | `/api/payments/:paymentId/receipt` | Fetch a receipt payload or PDF | Needed for sharing and customer records. |
| POST | `/api/payments/:paymentId/send-receipt` | Email or message a receipt | Useful for operational workflows. |

### Dashboard and analytics

| Method | Path | Purpose | Notes |
| --- | --- | --- | --- |
| GET | `/api/dashboard/summary` | KPI cards for the dashboard | Supplies outstanding debt, overdue debt, customers, collections. |
| GET | `/api/dashboard/recent-debts` | Recent debt table | Replaces the hard-coded table on the dashboard page. |
| GET | `/api/dashboard/recent-payments` | Recent payment table | For dashboard shortcuts and activity feeds. |
| GET | `/api/dashboard/risk-distribution` | Risk pie data | Used by the dashboard risk chart. |
| GET | `/api/dashboard/collections-trend` | Collections trend series | Used by the dashboard line chart. |
| GET | `/api/analytics/collection-rate` | Collection rate over time | Used by the analytics KPI cards. |
| GET | `/api/analytics/risk-trend` | Risk distribution trend | Used by the analytics area chart. |
| GET | `/api/analytics/customer-segments` | Customer value distribution | Used by the analytics bar chart. |
| GET | `/api/analytics/export` | Export analytic data | Return CSV, XLSX, or PDF. |

### Notifications and communications

| Method | Path | Purpose | Notes |
| --- | --- | --- | --- |
| GET | `/api/notifications` | Notification inbox | Needed for the bell icon and a notification center. |
| POST | `/api/notifications/:notificationId/read` | Mark one notification read | Required for unread state management. |
| POST | `/api/notifications/read-all` | Mark all notifications read | Common inbox behavior. |
| GET | `/api/notification-preferences` | Load notification settings | Used by the notifications tab in Settings. |
| PATCH | `/api/notification-preferences` | Save notification preferences | Email, SMS, and in-app toggles. |
| POST | `/api/reminders/debts/:debtId` | Trigger a reminder | Useful for overdue collections. |
| GET | `/api/communications` | Message/reminder log | Important for auditing collections activity. |

### Files, reports, and operations

| Method | Path | Purpose | Notes |
| --- | --- | --- | --- |
| POST | `/api/uploads` | Upload receipts or attachments | Needed for proof-of-payment and supporting docs. |
| GET | `/api/reports` | List saved reports | For scheduled or saved exports. |
| POST | `/api/reports` | Create a report definition | Optional if custom reporting is planned. |
| POST | `/api/reports/export` | Export a report | PDF/CSV/XLSX output. |
| GET | `/api/audit-logs` | View audit trail | Essential for financial operations. |
| GET | `/api/search` | Global search across entities | Supports the header search bar. |
| GET | `/api/settings` | Aggregate settings payload | Useful if the settings screen is server-driven. |
| PATCH | `/api/settings` | Save settings | Store profile, security, appearance, and business preferences. |

## Minimum Backend Checklist

If this app were moved from prototype to production, the minimum viable backend would need to cover:

- Login, register, password recovery, and current-user retrieval.
- Business profile and onboarding persistence.
- Full CRUD for customers, debts, and payments.
- Dashboard and analytics aggregation endpoints.
- Notifications, reminders, and audit logs.
- Search, filter, pagination, export, and attachment support.

## Current Gap Summary

The following gaps are visible in the current codebase:

- The auth forms navigate directly to other routes without calling an API.
- The dashboard and analytics pages render static arrays instead of server data.
- The sidebar, header, and settings UI assume account/session data that does not exist yet.
- No dynamic detail routes exist for customers, debts, or payments.
- No API handlers exist under `app/api`, so every required endpoint above is still missing.
