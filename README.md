# MASTER PROMPT — SP Task & Delegation System

Paste this whole prompt to rebuild or extend the app from scratch.

---

## ROLE & GOAL
You are building **SP Task & Delegation System** — a single-owner task delegation
web app. Author credit: **Somnath Karmakar**. Deliver **two complete files only**:

1. `Code.gs` — Google Apps Script backend (JSON API)
2. `index.html` — single-file frontend (HTML+CSS+JS)

Apply the **glass-ui** design system (dark glassmorphism, Jost font, purple→blue
accent, shimmer "DELEGATION" loading screen, glass modal popups, toast). UI text in
professional English. Mobile-first. No external JS libraries.

## ARCHITECTURE
* DB = Google Sheet (container-bound; use `getActiveSpreadsheet()`).
* Backend = Apps Script, single `doGet`/`doPost` router returning JSON.
* Frontend hosted on GitHub Pages; calls API via `fetch` POST with
  `Content-Type:text/plain` (avoids CORS preflight).
* Auth = token in `localStorage`.

## CORE MODEL (single-owner)
* **One-time signup** → first user becomes the **owner (Super Admin)**. After that
  signup is permanently locked. No second account.
* **Employees** are records (name/email/department) — they do NOT log in. The owner
  assigns tasks to them; they receive **email** notifications.
* Owner updates task status, sends follow-up emails, views reports.

---

# PART A — BACKEND `Code.gs` (step by step)

### A0. Config + constants
* `SALT`, `SESSION_HOURS=8`, `COMPANY_NAME`, `EMAIL_FROM`.
* `SHEETS = {EMP,TASK,LOG,SESSION,SETTINGS}`.
* `HEADERS` for each sheet:
  * **Employees**: EmpID, Name, Email, PasswordHash, Role, Department, PCName, Status, CreatedAt, LastLogin
  * **Tasks**: TaskID, Title, Description, AssignedTo, AssignedBy, Priority, AssignDate, DueDate, Status, CompletedAt, VerifiedBy, Recurrence, Attachment, CreatedAt
  * **TaskLog**: LogID, TaskID, Action, By, Timestamp, Detail
  * **Sessions**: Token, Email, Role, Name, Device, CreatedAt, ExpiresAt
  * **Settings**: Key, Value

### A1. Setup functions
* `ensureSheets_()` — create any missing sheet, add bold dark header row, freeze row 1, seed default Settings (CompanyName, EmailFromName, SessionHours). Return spreadsheet.
* `setup()` — call `ensureSheets_()`, set Jost font on all sheets, show alert "sign up first".
* `onOpen()` — add custom menu "SP Delegation" → Setup Sheets, Clear Expired Sessions.

### A2. Router
* `doGet(e)` — if action `ping` return `{ok,system,version}`, else `route_`.
* `doPost(e)` — JSON.parse `e.postData.contents`, then `route_`.
* `route_(action,p)` — try/catch switch dispatching to:
  `ping, signupStatus, signup, login, logout, checkSession, changePassword,
   forgotPassword, resetPassword, listAll, addEmployee, setEmployeeStatus,
   createTask, updateTaskStatus, followupTask, deleteTask`. Default → unknown action.

### A3. Signup + auth
* `signupStatus_()` — `ensureSheets_()`; return `{open: !superAdminExists_()}`.
* `signup_(p)` — block if owner exists; validate name/email/password (email regex, pwd≥6); append owner row (EMP-0001, Super Admin, Active, hashed pwd); create session; return token + user.
* `superAdminExists_()` — true if any Role == 'Super Admin'.
* `login_(email,password,device)` — find row by email; reject if inactive or hash mismatch; create session; update LastLogin; return token + user.
* `checkSession_(token)` — valid → `{user}` else error.
* `logout_(token)` — delete that session row.
* `changePassword_(token,old,new)` — verify session + old hash; update; pwd≥6.

### A4. Forgot / reset password (email OTP)
* `forgotPassword_(email)` — if email exists, generate 6-digit OTP, store via `setReset_`, email it with `sendResetMail_`; always return generic "code sent" (no leak).
* `resetPassword_(email,otp,newPw)` — load OTP via `getReset_`; check existence, expiry (10 min), match; update password hash; `clearReset_`; `clearSessionsForEmail_` (logout everywhere).
* `setReset_/getReset_/clearReset_` — store `{otp,exp}` in `PropertiesService.getScriptProperties()` keyed `reset_<email>`.
* `clearSessionsForEmail_(email)` — delete all session rows for that email.
* `sendResetMail_(to,name,otp)` — branded HTML email with large OTP, 10-min note.

### A5. Data read
* `listAll_(token)` — validate session; return `{me, employees[], tasks[]}` mapped from sheets (dates formatted via `fmtDate_`). Frontend computes stats/filters/reports from this.

### A6. Employees
* `addEmployee_(p)` — validate; reject duplicate email; append `EMP-000N`, Role 'Employee', empty password, Status Active.
* `setEmployeeStatus_(p)` — set Active/Inactive by EmpID; block deactivating the Super Admin.

### A7. Tasks
* `createTask_(p)` — validate title + assignedTo; append `TASK-000N` (AssignDate=today, Status 'Pending', AssignedBy=owner email); `log_('Created')`; send assign email via `sendTaskMail_`; return success (note if email failed).
* `updateTaskStatus_(p)` — set status from [Pending, In Progress, Done, Verified]; if Done set CompletedAt; if Verified set VerifiedBy; `log_`.
* `followupTask_(p)` — load task; send reminder email via `sendTaskMail_('followup')`; `log_('Follow-up sent')`.
* `deleteTask_(p)` — delete row by TaskID; `log_('Deleted')`.

### A8. Email
* `sendTaskMail_(to,name,type,t)` — branded HTML (company header, task table: Task/Details/Priority/Due Date/Assigned By, footer credit). `type` = 'assign' or 'followup' changes subject + intro. Uses `MailApp.sendEmail`.

### A9. Helpers
* `makeSession_(email,role,name,device)` — UUID token, append session with expiry = now + SESSION_HOURS.
* `getSession_(token)` — return session or null; delete + null if expired.
* `findEmpRow_(sh,email)` — row by email (col C); guard last<2 → null.
* `findRowById_(sh,id,col)` — row by id.
* `empName_(ss,email)` — name lookup.
* `readSheet_(sh)` — return array of objects keyed by header row.
* `log_(taskId,action,by,detail)` — append TaskLog row (`LOG-000N`).
* `nextId_(sh,prefix,col)` — next serial like PREFIX-0001.
* `hashPw_(pw)` — SHA-256 of `pw + SALT`, hex.
* `getSetting_(key)` / `getSessionHours_()` — read Settings.
* `fmtDate_(v)` — Date → 'yyyy-MM-dd', else string.
* `clearExpiredSessions()` — purge expired sessions.
* `json_(obj)` — `ContentService` JSON output.

---

# PART B — FRONTEND `index.html` (step by step)

### B0. Head + CSS
* Jost font link (weights 300–800).
* Apply full **glass-ui** CSS: color `:root` tokens, blobs, glass `.shell`, auth
  `.brand`/`.panel`, `.field` inputs (16px), `.btn` (primary/ghost/sm/danger),
  `.msg`, `.spin`, app `.topbar`/`.nav`/`.content`, `.cards`/`.stat`/`.box`,
  `.tcard`, `.badge` (status+priority colors), `.filters`/`.frow`, `table.rep`,
  `.emprow`, `.empty`, **`.toast` (z-index 10001)**, **loader `.faceLoader` +
  `.loading-text` shimmer**, **`.modal` popup**, responsive `@media(max-width:760px)`.

### B1. Body markup
* `<div class="blob b1"></div><div class="blob b2"></div>`
* `<div id="root"></div>`
* `<div class="toast" id="toast"></div>`
* Loader overlay: `<div class="faceLoader" id="faceLoader"><div class="loading-text">Delegation</div></div>`
* Modal: `#modal` with `#modalIc/#modalTitle/#modalMsg/#modalActs`, backdrop click closes.

### B2. Globals + utils
* `const API` = GAS URL; `const DEVICE` = userAgent+platform.
* `STATE = {token, me, employees, tasks, view:'dashboard', filters:{date,status,emp}}`.
* `api(p)` — wraps fetch; calls `showLoader()`/`hideLoader()` (busy counter) so the
  shimmer loader shows on EVERY server call.
* `tGet/tSet/tClr` — localStorage token.
* `esc, toast, todayStr, addDays, isDone, isOverdue, initials`.
* `modalConfirm(o)/modalAlert/closeModal` — replace native alert/confirm.

### B3. Boot
* IIFE: if token → `checkSession`; valid → `loadApp()`. Else `signupStatus` → if open
  `renderSignup()` else `renderLogin()`.

### B4. Auth views
* `authShell(inner)` — split-screen brand panel + form panel wrapper.
* `renderSignup()` / `doSignup()` — owner creation form → on ok store token, `loadApp`.
* `renderLogin()` / `doLogin()` — email+password (+Enter key) → on ok `loadApp`. Has
  "Forgot password?" link.
* `renderForgot()` / `doForgot()` — email → `forgotPassword` → `renderReset(email)`.
* `renderReset(email)` / `doReset(email)` — OTP + new password → `resetPassword` → back
  to login with success message.

### B5. App shell
* `loadApp()` — show loading then `refresh(true)`.
* `refresh(first)` — `listAll`; on fail logout→login; store data; if first `renderShell()`; `renderView()`.
* `renderShell()` — topbar (logo, user chip, Sign Out) + nav pills + `#content`.
* `navBtn(id,label)`, `go(v)` — switch active view.
* `renderView()` — render the active view into `#content`.

### B6. Views (function by function)
* `viewDashboard()` + `stat()` — 5 stat cards (Total, Pending, In Progress, Completed, Overdue) + recent tasks.
* `viewEmployees()` + `empRow()` + `addEmp()` + `toggleEmp()` — add form + team list + enable/disable.
* `viewAssign()` + `createTask()` — employee dropdown + title + description + priority + due date → create + email; then `go('tasks')`.
* `viewTasks()` + `setF()` + `applyFilters()` + `taskCard()` + `setStatus()` + `followup()` + `delTask()`:
  * Filters: date chips (All/Today/Tomorrow/Future/Previous) + status select (+Overdue computed) + employee select.
  * Each task card: status badge (Overdue computed), assignee, priority, due, id; actions: Start, Mark Done, Verify, Follow-up (email), Delete (modal confirm).
* `viewReports()` — per-employee table: Total, Done, Pending, Overdue, On-time % (color-coded).

### B7. Common
* `doLogout()` — `logout` + clear token + reload.
* `v(id)`, `showMsg(el,type,txt)`, `reset(b,txt)`.

---

# RULES
1. Output the **two full files** only — no snippets.
2. Follow **glass-ui** exactly (Jost, colors, shimmer loader, modal, toast).
3. Loader on every server action; instant UI (tab/filter) has no loader.
4. No native `alert()`/`confirm()` — use modal.
5. Toast z-index above all overlays (10001).
6. Inputs `font-size:16px` (no iOS zoom).
7. Single-owner; signup locks after first account.
8. Auto-create sheets/headers in `Code.gs`.
9. Professional English UI; footer "Built by Somnath Karmakar".
10. On code change: redeploy via **Manage Deployments → pencil → New version** (URL stays same), never "New deployment".
