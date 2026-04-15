# expenseTrackerDraft
Build a Production‑Ready Expense Tracker (MERN + Nigerian Tax Compliance)
You are an expert full‑stack developer. Build a complete expense tracking web application for small‑scale Nigerian businesses. The app must be production‑ready, secure, scalable, and fully aligned with the Nigeria Tax Act 2025 (effective 2026). Use the MERN stack (MongoDB, Express.js, React, Node.js). Follow DRY (Don’t Repeat Yourself) principles, clean architecture, and modern best practices for security, performance, testing, and deployment.

1. Core Business Requirements
The app helps small businesses track expenses, automatically handle tax compliance, and generate audit‑ready reports. Key features:

User authentication (email/password) with role‑based access (Admin, Staff).

Expense management – add, edit, delete, view, filter (by date, category, amount).

Automatic tax calculations based on Nigerian laws:

VAT (7.5%) – track input VAT on purchases; compute VAT‑inclusive/exclusive amounts.

Company Income Tax (CIT) – small companies (turnover ≤ ₦100M) are exempt, but must file annual returns.

Withholding Tax (WHT) – exempt if annual turnover ≤ ₦100M.

Capital Gains Tax (CGT) – 30% on disposal of business assets.

Presumptive Tax – for turnover ≤ ₦12M, a simplified regime.

Turnover monitoring – track total income (from invoices) and compare against thresholds (₦50M for VAT, ₦100M for CIT). Show real‑time alerts when approaching limits.

E‑invoicing readiness – support for generating invoices compatible with FIRS Merchant Buyer Solution (MBS) format (sequential numbering, Tax ID, buyer/seller details).

Audit trail – every change to expenses or invoices is logged immutably (who, what, when, old value, new value).

Reporting – generate monthly/quarterly/annual tax reports (VAT returns, CIT returns, expense summaries) in PDF and Excel.

2. Technical Stack & Architecture
2.1 Backend (Node.js + Express)
Language: TypeScript (for type safety across the stack).

Structure (modular, layered):

text
src/
├── config/          # environment variables, tax constants, database config
├── models/          # Mongoose schemas (User, Expense, Invoice, AuditLog, CompanySettings)
├── services/        # business logic (taxService, reportService, auditService)
├── controllers/     # thin request handlers (call services)
├── routes/          # API endpoints (v1/expenses, v1/auth, v1/reports)
├── middleware/      # auth, validation, errorHandler, logger, rateLimiter
├── validations/     # Joi / Zod schemas (reused on frontend via shared package)
├── utils/           # helpers (pagination, date formatting, currency formatter)
└── server.ts
Database: MongoDB with Mongoose. Indexes on user, date, category, deletedAt. Soft delete expenses (field deletedAt: Date).

Authentication: JWT with refresh tokens stored in HTTP‑only cookies (not localStorage). Implement role‑based access (admin, staff). Use bcryptjs for password hashing.

Security:

Helmet.js for secure headers.

Rate limiting (express‑rate‑limit) – 100 requests per 15 minutes per IP.

CORS – restrict to frontend domain.

Input sanitisation (express‑validator, plus DOMPurify on frontend for any HTML).

No sensitive data in logs.

Logging: Winston – structured JSON logs, separate audit log file for expense changes.

Error handling: Centralised async error handler; custom AppError class.

2.2 Frontend (React)
Build tool: Vite (instead of Create React App).

Language: TypeScript.

State management: Redux Toolkit with RTK Query (for API calls, caching, loading/error states). Or Zustand for simpler cases – but RTK Query strongly preferred for DRY API logic.

Routing: React Router v6.

UI Components: Reusable, styled with Tailwind CSS (or Material‑UI if preferred). Create a components/ui/ folder with Button, Input, Select, Modal, Table, Toast.

Forms: React Hook Form + Yup (or Zod) validation – reuse the same validation schemas as backend (via a shared npm package).

Tax & formatting helpers: Centralised src/utils/taxHelpers.ts and formatHelpers.ts (e.g., formatNaira, calculateVATOnExpense).

Feature‑based structure (not by file type):

text
src/features/
  auth/         (Login, Register, useAuth hook)
  expenses/     (ExpenseForm, ExpenseList, useExpenses hook)
  invoices/     (InvoiceForm, InvoiceList)
  reports/      (ReportGenerator, TaxSummary)
  dashboard/    (Dashboard page with turnover alerts)
Custom hooks: useApi to abstract loading/error states; useExpenses that uses useApi or RTK Query queries.

Performance:

Code splitting + lazy loading for routes.

React.memo for expensive lists.

Virtualised tables for >100 rows (react‑virtual).

2.3 Shared Code (DRY across frontend/backend)
Create a separate shared/ package (or monorepo using npm workspaces) that exports:

Tax constants (rates, thresholds).

Validation schemas (Yup/Zod).

TypeScript interfaces (IExpense, ITaxCalculationResult).

Utility functions (date formatting, currency conversion).

Both backend and frontend import from this shared package – ensuring consistency.

3. Nigerian Tax Logic – Detailed Implementation
3.1 Tax Configuration (shared/taxConfig.ts)
ts
export const TAX_CONFIG = {
  VAT_RATE: 0.075,
  VAT_THRESHOLD: 50_000_000,      // ₦50M annual turnover – exemption
  CIT_THRESHOLD: 100_000_000,     // ₦100M – small company exemption
  WHT_THRESHOLD: 100_000_000,
  PRESUMPTIVE_THRESHOLD: 12_000_000,
  CGT_RATE: 0.30,
  // Filing deadlines (days after year end)
  CIT_FILING_DEADLINE_DAYS: 180,
  VAT_FILING_DEADLINE_DAYS: 21,
};
3.2 Tax Service (backend/src/services/taxService.ts)
Provide functions:

computeInputVAT(amountExcludingVAT: number): number

isCITExempt(annualTurnover: number): boolean

isVATExempt(annualTurnover: number): boolean

calculatePresumptiveTax(turnover: number): number (e.g., 0.5% of turnover)

estimateCIT(profit: number, turnover: number): number (if not exempt)

generateVATReturn(expenses: IExpense[], period: {start, end}) – sum input VAT on eligible expenses.

3.3 Turnover Monitoring Service
A background job (node‑cron) runs daily to aggregate all invoice totals (paid invoices) per company.

Compare against thresholds. If turnover >= 0.9 * THRESHOLD, create an alert in the database and send email notification to admin.

On the dashboard, display a progress bar and warning message.

3.4 Audit Trail
Use Mongoose middleware (post‑save, post‑findOneAndUpdate) to automatically write to an AuditLog collection:

ts
{
  action: 'CREATE' | 'UPDATE' | 'DELETE',
  collection: 'expenses',
  documentId: ObjectId,
  userId: ObjectId,
  changes: object,   // for updates, store old vs new
  timestamp: Date
}
No manual logging in controllers.

4. API Endpoints (RESTful)
Method	Endpoint	Description	Access
POST	/api/v1/auth/register	Register user	Public
POST	/api/v1/auth/login	Login (returns http‑only cookie)	Public
POST	/api/v1/auth/logout	Clear cookie	Auth
GET	/api/v1/auth/me	Get current user	Auth
GET	/api/v1/expenses	List user's expenses (paginated)	Auth
POST	/api/v1/expenses	Create expense	Auth
PUT	/api/v1/expenses/:id	Update expense	Auth (owner)
DELETE	/api/v1/expenses/:id	Soft delete expense	Auth (owner)
GET	/api/v1/reports/tax-summary	Get tax summary for a period	Admin
GET	/api/v1/reports/vat-return	Download VAT return (PDF/Excel)	Admin
GET	/api/v1/alerts	Get turnover / filing alerts	Admin
POST	/api/v1/invoices	Create invoice (MBS‑compatible)	Auth
All endpoints return JSON with standard structure:

json
{ "success": true, "data": {...}, "message": "optional" }
5. Security & Compliance Checklist
Passwords hashed with bcrypt (salt rounds = 10).

JWT access token expires in 15 minutes; refresh token in 7 days (stored in DB, revoked on logout).

Refresh token rotation – issue new refresh token on each refresh.

All API endpoints (except login/register) protected by verifyJWT middleware.

Role middleware: requireRole(['admin', 'staff']).

Input validation: Joi schema for each endpoint; reject extra fields.

HTTPS enforced in production (via environment flag).

No CORS wildcard (origin: process.env.FRONTEND_URL).

Rate limiting per user (if authenticated, use user ID; else IP).

Helmet with default settings.

XSS protection: on frontend, sanitise any user‑generated HTML before rendering (use DOMPurify).

SQL injection not applicable (MongoDB), but use mongoose $regex safely (no user‑controlled regex injection).

6. Testing Strategy
Unit tests (Jest + supertest for backend):

Tax service functions (given input, expect correct output).

Validation schemas.

Audit log middleware.

Integration tests:

API endpoints with an in‑memory MongoDB (mongodb‑memory‑server).

Test full flow: register → login → create expense → update → soft delete → verify audit log.

Frontend tests:

Component tests (React Testing Library).

Hook tests (@testing-library/react-hooks).

E2E tests (Cypress or Playwright) for critical user journeys: login, add expense, generate report.

Coverage target: >80% for business logic (services, utils).

7. DevOps & Deployment
Containerisation: Docker + Docker Compose (backend, frontend, MongoDB, Redis for rate limiting optional).

CI/CD: GitHub Actions (or GitLab CI). Pipeline steps:

Lint (ESLint, Prettier).

Unit & integration tests.

Build (TypeScript compilation, Vite build).

Push Docker image to registry (GHCR / Docker Hub).

Deploy to cloud (AWS ECS / DigitalOcean / Render) using docker-compose or Kubernetes for larger scale.

Environment variables: Use .env files for development; in production, use secret manager (AWS Secrets Manager or environment variables on the host).

Monitoring:

Application logs aggregated to Datadog / Logtail / Sentry.

Uptime monitoring (UptimeRobot).

Performance monitoring (New Relic or OpenTelemetry).

8. Deliverables & Documentation
Provide:

Complete source code with clear folder structure and comments.

README.md containing:

Setup instructions (clone, install, environment variables, run with Docker).

API documentation (Postman collection or OpenAPI spec).

Explanation of tax logic and thresholds.

Database schema diagram (Mongoose models relationships).

Sample .env files for development and production.

Test report (output of npm test).

Deployment guide (how to deploy to a cloud provider).

9. Additional Quality Requirements
DRY enforcement:

Tax rates defined in one place (shared package).

Validation schemas defined once, used on backend and frontend.

API calls on frontend use RTK Query – no duplicate fetch logic.

Reusable React components (Button, Modal, Table) – no copy‑paste UI.

Performance:

MongoDB indexes on { user: 1, date: -1 }, { user: 1, category: 1 }.

Pagination (limit/offset or cursor) for /expenses.

Frontend bundle size <200KB (gzipped) for initial load.

Accessibility: WCAG 2.1 AA – proper labels, keyboard navigation, ARIA roles.

10. Example User Stories (Acceptance Criteria)
As a business owner, I can register, log in, and see a dashboard with my current turnover, tax alerts, and recent expenses.

As a staff member, I can add an expense with amount, description, category, and date. The system automatically calculates input VAT and stores it.

As an admin, I can view an audit log showing who changed which expense and when.

As a business owner, when my annual turnover reaches ₦45M, I see a warning that I am approaching the VAT exemption threshold.

As a user, I can generate a PDF “VAT Return” for a selected quarter, listing all purchases with input VAT.

As a user, I can soft‑delete an expense; it disappears from the list but remains in the audit trail and tax reports (flagged as deleted).
