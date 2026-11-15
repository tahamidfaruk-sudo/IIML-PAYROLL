# BPO PAYROLL WEB APP — FULL BUILD PROMPT (for any AI agent / app builder)

Instructions for the AI agent receiving this prompt: build the COMPLETE application described below as one project. Do not ask clarifying questions and do not wait for confirmation between steps — every business decision has already been made by the owner and is written here. Where a value is described as configurable or dynamic, store it in the database / Settings with the default given; never hardcode it. Build the full working app — database schema, authentication, every screen listed, the calculation engine, and seed demo data — not a prototype, mockup or landing page. Follow the build order at the end, and finish by running the acceptance check in the final section and reporting its result.

---

Build a complete, production-quality **BPO Payroll Management Web App** with a **Supabase (PostgreSQL)** backend, **Supabase Auth** login, and **strict Row Level Security**. (Platform note: if the environment you build in provides an equivalent managed PostgreSQL + authentication service instead of Supabase, use that instead — the data model, roles and access rules below must stay identical.) The app manages monthly payroll for a data-entry BPO company whose employees are paid mainly per production quantity. Build the full app, not a landing page: database schema, seed data, authentication, every screen listed below, and a working salary calculation engine. Nothing in the business rules may be hardcoded — every rate, threshold, percentage, country and category lives in the database and is editable from Settings/master-data screens.

## 1. Business overview

- The company runs ~30 client **projects** (demo projects: NA0390, NA0401, NA0423, NA0424). Each project has its own rate card, its own set of record types, and a **payment base** (see §4).
- Work is keyed by operators in several locations. **Country/location codes are dynamic master data** — seed with **BD, KOL, KTM, IDR**; an admin must be able to add a new country code and have rate cards, employees and reports pick it up with no code change.
- Employees have a **contract type**: FREELANCER (paid per production), FIXED (fixed monthly salary), TRAINEE (tenure incentive), plus staff who do **special work** paid by the day (§10).
- Monthly flow: preparation tracking → production CSV imports → batch/transmission registry → proof & doctype data → attendance import → reconciliation → salary calculation → Manager approval → Accounts approval → payment generation → period lock → payslips and branch salary sheets (WFO / WFH).

## 2. Users and roles

- **Admin** — everything, including users, projects, settings, master data.
- **Manager** — uploads, calculation, first approval; owns the projects assigned to them.
- **Accounts** — second approval, payment generation and exports.
- **Auditor** — read-only everywhere.
- Login with email + password via Supabase Auth. Demo seed users: admin@demo.local, manager@demo.local, accounts@demo.local, auditor@demo.local (password: role name + "123", e.g. admin123), clearly marked as demo data.
- Strict RLS on every table; the app must also run end-to-end in demo mode on the seed data.

## 3. Employees — two IDs, transfers, master data

- Every employee has a **current employee ID** and an **old employee ID** (IDs change over time; both are kept forever). Employee master fields: name, current ID, old ID, joining date, designation, department, **branch (WFO / WFH)**, **country code**, team, manager, contract type, pay method (bank / bKash / Nagad) with account or mobile number.
- Uploads may reference EITHER ID and must resolve to the same person — quantities and amounts consolidate onto the current ID (Excel equivalent: SUMIF by current ID + SUMIF by old ID). An employee upload keyed by an old ID updates that same employee; never create a duplicate, never overwrite the current ID.
- **Transfers:** employees transfer between companies/branches (e.g. from a sister concern). Keep a transfer record (name, ID, joining date, transfer date, from/to) — a transferred employee's tenure for trainee incentive counts from their original joining date where the transfer sheet says so.
- Both IDs appear in employee lists, the calculation screen, an ID register, the master salary sheet, and payslips (payslip search must work by old ID).

## 4. Projects — feature toggles and payment base

At project creation/editing:

- Three Yes/No dropdowns: **Proof**, **Doctype**, **Split**. Selecting **No** removes that feature from EVERY later step for that project: its rates disappear from the rate card, uploads of that kind fail validation with a clear error, the calculation skips it, and payslips omit it. Selecting **Yes** lets it flow through all steps. Seed NA0423 with Doctype = No as a working demo.
- **Payment base** (per project, from a configurable list; seed values): **Record** (pay per record), **Character** (pay per character), **DocWise-SplitWise** (pay follows the batch registry's billable counts with doctype split), **Batch** (pay per batch at the batch's standard quantity). Every project also records: category (English / Non-English), software (IIM / Client), manager/owner, active countries, sub-projects.
- Record types are **per-project master data**: each project defines its own record-type groups (examples from real projects: Header, Person, Parents, Birth, Death, Marriage, Full Key, Review Key, Header PZ, BMD With Zoning, BMD Without Zoning, Prezone, Top Header, Inbound) and each record type has a **unit — records or characters**. New projects must be addable with their own record types without code changes.

## 5. Rate cards

- Per project, per country code, one rate per record type — plus a **per-image doctype rate** stored on the project's own rate card (never a global rate; rates are project-wise different by design).
- The engine's rate row for an employee is chosen by the employee's country code.
- All rates editable in the UI, stored in the database, effective-dated by payroll period.

## 6. Uploads (Upload Center)

CSV and XLSX uploads with a validation preview and a downloadable row-by-row error report before anything is committed.

1. **Production record details** — the real export format of the production software, one CSV per batch file, many files per batch: columns `Source file, Project Name, Client Batch, EMID, UserName, Date, InTime, Last OutTime, HrSpent`, then per-record-type `Record`/`Char` quantity pairs, `Skipped`, `RPH`. Import ALL files for a period together; aggregate quantities by employee (either ID) and record type. ⚠️ A repeated key (same employee + record type appearing in many rows/files, including main + remainder split rows) is NOT a duplicate — always SUM quantities, never dedupe rows.
2. **Proof records** — per project sheet: Project, Batch, Image from, Image to, **Proof User**, **Entry User**, Team, then per record-type group a Record count (see §8 for how the split is applied by the engine — the file carries counts, not money).
3. **Doctype / Doc ID records** — Batch, Images, Transmission Status, Operator EMID (or Operator Name + EMID + Team). The operator's image count for the period feeds the doctype pay in §8.
4. **Batch registry** (§7) — batch-level rows with transmission data.
5. **Shipdet** (§7) — the client shipment/acceptance file.
6. **Attendance (HRMS format)** — Branch, Department, Designation, Name, EMID, WFO days, WFH days, WD-Present, PL, UL, Adj.Off, Holiday, Absent, Late Days, Night, WE-Work, Festival Work, **Avg. Working Hr.** (a duration, may exceed 8:00). Avg. Working Hr. is the slab input in §8.
7. **Special Work register** (§10) — daily rows.
8. **Deductions** — kind (advance / loan / penalty / adjustment), amount, note, per employee per period.
9. **Employees** — master data incl. old ID, branch, country, contract type.
- Uploading a kind whose project feature is No (e.g. doctype for a Doctype=No project) must be rejected at validation. Unknown employee IDs or record types are validation errors, listed row by row. Imported data is never silently corrected.

## 7. Batch registry, transmission and Shipdet reconciliation

- **Batch registry** (one row per batch per project per period): client batch name, IIMI batch name, combined-batch mapping (several client batches may map to one IIMI batch), images, records, billable records, billable characters, **transmission status** (Transmitted / Non-Transmitted + reason), transmitted date, server location, sub-project, team name, payment base. A DOCID side-list per batch records which operator did the document tagging (batch → images → operator EMID).
- **Shipdet import** (the accepted/client stage): per batch — shipped month, records, **character count (client)**, **character count (system/IIMI)**, difference and difference %, Audit A sample count, Audit A date, invoiced-to-client flag, comments.
- **Reconciliation screen**: per project and period, three stages — **Produced** (production imports) → **Transmitted** (batch registry) → **Accepted** (Shipdet) — with per-batch and total differences highlighted. A per-project **payable quantity policy** decides which stage payroll follows; default = **Produced** (pay never silently follows transmitted/accepted — in practice salary is prepared even for non-transmitted batches). Mismatches create **reconciliation exceptions**: warn-and-proceed — the Approvals screen shows an alert banner with the open-exception count, approvals are never blocked, and the approval audit log records how many exceptions were open at approval time.
- **Preparation tracker**: per project per month — salary preparation status, batch information updated (Y/N), transmit status, per-country download status, supporting-file status, accounts confirmation/feedback status, assigned person, project manager. This is the monthly checklist the team works through before calculation.

## 8. Salary calculation engine (exact rules)

For each employee and period, per project first, then consolidated in §9:

1. **Production base** = Σ over the project's record types of (quantity × rate), quantities aggregated in §6.1, rate from the project's rate card at the employee's country code. (Depending on record type, quantity is records or characters — the record type's unit decides.)
2. **Slab** on the employee's Avg. Working Hr. (from the HRMS import), applied to the production base:
   - avg hours < 3.99 → **60%**; 3.99 ≤ avg hours < 5.99 → **80%**; avg hours ≥ 5.99 → **100%** (gap-free boundaries; thresholds configurable in Settings).
3. **Proof** (only if project Proof = Yes): proof quantities are aggregated per record-type group for the proof user and the entry user. The split is defined by **two independent, configurable ratios** — a **credit ratio (default 0.35)** and a **charge ratio (default 0.65)** — settable globally in Settings and overridable per project + record-type group (a group may be 0/1 = no split):
   - proof user is **credited** Σ (proofed records × credit ratio × that record type's rate);
   - entry user is **charged** Σ (proofed records × charge ratio × that record type's rate), deducted from their pay.
   If the project's Split toggle is No, the proof amounts apply without splitting (full credit, no charge) using the group's no-split ratios. Proof rows imported with a ready money amount instead of counts are summed as given.
4. **Doctype** (only if Doctype = Yes): operator's total Doc ID images for the project × the project's per-image rate from its rate card.
5. **Project payable** = (production base × slab %) + proof credit − proof charge + doctype.
6. **Fixed contract type:** the fixed monthly amount **REPLACES** production pay entirely (no production, slab, proof or doctype for that employee).
7. **Trainee incentive** (§11) replaces production pay for trainees.
8. **Special-work pay** (§10) is added for employees with special-work days.

**Worked example the engine MUST reproduce exactly** (seed as demo data): employee EMP-001 (Freelancer, BD) in a month where production quantities × BD rates = **31.50**; proof gives **+2.10** credit and **−0.80** charge; doctype **40 images × 0.50 = 20.00**; subtotal **52.80**; avg hours ≥ 5.99 → slab **100%**; **project payable ৳53** after whole-taka rounding.

## 9. Master salary and final assembly (confirmed sequence)

1. **Master Salary sheet**: one row per employee — name, current ID, **old ID**, joining date, designation, contract type, team — one column per project with the employee's §8 project payable (consolidated across both IDs), a **total production salary** column, and per-project totals + grand total at the top. This mirrors the company's real combine sheet.
2. **Final net pay** per employee = total production salary (or fixed amount, or trainee incentive) **+ special-work pay − deductions (advance / loan / penalty) +/− adjustments − TDS**.
3. **TDS**: a percentage deduction on payable, enabled, rate configurable in Settings, **default 0%** until set. **PF is NOT applicable** — no PF anywhere.
4. **Rounding**: net pay rounded to **whole taka** (rounding mode is a Settings value).
5. **Branch salary outputs**: the final salary list is produced as two sheets — **WFO salary** and **WFO/WFH split by the employee's branch** (WFO list and WFH list) — plus the fixed-salary list; each with employee ID, old ID, name, designation, team, payable components and net pay.

## 10. Special Work (daily register)

- A daily register kept per team: Date, Employee Name, ProdID, EMID, Designation, Joining date, Project, **Category of special work** (configurable list; seed: Production, QC, Audit A, New project Trial, Other work, Work As A FT, Supervisor Task, Finalization work, Review Work, Tag Work, Linking), Details of work, Work start, Work end.
- **Total hours per day are capped at 7:30** (cap configurable); **In Days = hours ÷ 7.5, capped at 1.0 per day**. The register totals each employee's present days per period.
- **Special-work pay rule (owner-confirmed)**: daily rate = (**average of the employee's NET pay over the last 3 months**) ÷ (**this month's working days**); **special-work pay = daily rate × the employee's total special-work present days this month**. If the employee has fewer than 3 months of net-pay history, average over the months available; if none, the rate is 0 and the row is flagged for manual review.

## 11. Trainee incentive

- Trainees are paid a fixed training allowance by tenure phase — phases by completed months of service (seed: **months 1–3 → ৳3,000/month; months 4–6 → ৳5,000/month; month 6+ → ৳8,000/month**). Phase boundaries and amounts are **Settings values** — management changes them; nothing hardcoded.
- The payable incentive is **attendance-gated** (current company practice): it is computed from the phase amount together with the trainee's **actual present days vs the days they should be present** in the phase, and — for trainees older than 3 months — a **working-hour condition** (standard working hours vs actual hours from attendance). Keep the gate parameters (required-present basis, standard hours) configurable alongside the amounts.
- Transferred trainees use the transferred-operator variant: tenure counts from the original joining date, with month counting split by transfer date (Type 2 / Type 3 patterns in the current sheet) — model this with an `is_transferred` flag + original joining date on the employee.

## 12. Approvals, payments, period control

- Period status flow: **open → manager_approved → accounts_approved → locked**.
- **Payment generation unlocks at accounts_approved** — it must NOT wait for lock. Before Accounts approval, show a short status note instead of the generate action.
- Exports after generation: **bank payment file**, **bKash/Nagad file**, salary CSV. Default export columns (configurable template): SL | Employee ID | Old ID | Name | Pay Method | Account/Mobile No | Net Amount.
- A **locked** period blocks ALL writes everywhere (store-level guard, not just hidden buttons).
- Every create/update/delete writes an audit log entry (who, what, when). An **Audit Pack** export lists the period's source files, approvals and exception counts.

## 13. Screens / modules

Login; Dashboard (KPIs incl. open recon exceptions and preparation progress); Employees (both IDs, transfers); Projects (feature toggles + payment base + record types); Rate Card (per project/country, toggle-aware); Periods with a **cut-off calendar** (input cut-off, calculation, approval, payment dates + owners); Preparation Tracker; Upload Center (validation preview + error report); Batch Registry (with DOCID side-list); Reconciliation (3 stages, Shipdet differences, exceptions); Special Work Register; Calculation (per-employee breakdown incl. Employee ID register); Master Salary (employee × project matrix, dual-ID); Variance Review (this month vs last month, large changes flagged); Approvals (Manager → Accounts, exception alert banner); Payments; Payslips (printable/PDF, both IDs); Branch Salary Sheets (WFO / WFH / Fixed); Audit Pack; Settings / Rules Engine (slab bands, proof credit & charge ratios, trainee phases, special-work parameters, TDS, rounding, country codes, categories, accent/theme); Users (Admin only).

## 14. UI / UX requirements (mandatory)

- Completely fresh, professional **corporate LIGHT theme by default**; **Light / Dark / System** modes persisted per user; **user-selectable accent colour** in Settings, persisted.
- **NO placeholder text inside inputs** — labels only. **NO unnecessary instruction or helper text** in the app.
- Attractive, consistent colours, icons, buttons and input cells; clean data tables with pagination.
- **Fully responsive** — the primary reviewer uses a phone; every screen must work at mobile width with no horizontal overflow.

## 15. Seed data and verification

Seed: the four demo projects with their real record-type styles (NA0390 Full Key/Review Key char-based; NA0401 record-based with Header groups; NA0423 BMD zoning groups; NA0424 Header Full Key), demo employees of every contract type (each with current AND old ID, spread across BD/KOL/KTM/IDR and WFO/WFH), rate cards, HRMS-style attendance rows, two periods (one locked, one open), and the EMP-001 worked example. Acceptance: log in as Admin, open the open period — EMP-001's payslip must show base 31.50, proof +2.10 / −0.80, doctype 20.00, subtotal 52.80, slab 100%, net ৳53; the Master Salary total for a seeded project must equal the sum of its per-employee project payables.

## 16. Build order

1. Schema + RLS + seed. 2. Auth + roles. 3. Master data (countries, employees + transfers, projects + toggles + record types, rate cards). 4. Periods + preparation tracker. 5. Uploads + validation (production CSV set first). 6. Batch registry + Shipdet + reconciliation. 7. Attendance + Special Work register. 8. Calculation engine + EMP-001 example passing. 9. Trainee incentive. 10. Master Salary + final assembly + branch sheets. 11. Approvals + payments + payslips + audit pack. 12. Settings/Rules Engine + theme system. 13. Final pass: responsive check on every screen at phone width, zero input placeholders anywhere.
