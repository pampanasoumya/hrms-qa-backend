# Test Strategy — Employee Management System
# Codebase: fenil29/employee-management-system-backend-node

## This strategy is based on reading app.js directly.
## Every flow named here maps to a real route in the codebase.

---

## The 5 Most Critical Flows — Ranked by Business Impact

### 1. Salary Assignment — POST /api/salary/:id
**What it does:** Assigns BasicSalary, BankName, AccountNo, 
AccountHolderName, IFSCcode, TaxDeduction to an employee.

**Critical bug found in code:**
BasicSalary is defined as `{ type: String }` in the Salary schema.
The API accepts "abc" as a valid salary. No numeric validation exists.
A site manager could enter "TBD" as salary and it saves successfully.
The worker's payslip shows a string, not a number.

**Worst case:** Construction worker's salary record contains "abc" 
or "0" as BasicSalary. Payroll operator doesn't notice during 
bulk processing. Worker receives wrong or zero pay.

**I will automate:** POST /api/salary/:id with string salary value,
negative salary, zero salary — all should return 400, currently don't.

**I will test manually:** The salary entry form UI — does it show 
a number keyboard on mobile? Does it prevent letter input visually?

---

### 2. Employee Creation — POST /api/employee
**What it does:** Creates employee with FirstName, MiddleName, 
LastName, Email, Password, Gender, DOB, DateOfJoining, 
ContactNo, EmployeeCode, Account.

**Critical issue found in code:**
Password is stored in plain text. `document.Password == req.body.password`
in the login route. No bcrypt, no hashing. If the database is 
compromised, every employee's password is exposed immediately.

**Worst case:** Database breach exposes all employee credentials.
For construction workers, this may be their only digital account.

**I will automate:** POST /api/employee with missing required fields,
duplicate email, invalid date formats.

**I will NOT automate:** Password security — this is an architectural 
decision that needs a conversation with the dev lead, not a test.
A test that checks "password is hashed" would require access to 
the database layer. I'll document it as a security finding instead.

---

### 3. Employee Deletion — DELETE /api/employee/:id
**What it does:** Nothing. The route exists but returns "error" 
for every request. This is in the code:
`res.send("error"); // deletion commented out`

**Worst case:** HR tries to remove a terminated worker from the system.
The worker remains active in payroll. They continue receiving 
salary calculations after their last working day.
For a construction company with high turnover, this is critical.

**I will automate:** DELETE /api/employee/:id — verify it returns 
a proper error response with status 501 (Not Implemented), not 
a 200 with the word "error" in the body.

**I will NOT automate:** The fix itself — that requires a product 
decision about soft delete vs hard delete and what happens to 
the deleted employee's salary records.

---

### 4. Login — POST /api/login
**What it does:** Takes email + password, returns JWT token.
Token payload contains _id, Account (role), FirstName, LastName.

**Critical issue found in code:**
When login fails (wrong password or user not found), the API 
returns `res.send("false")` — a 200 response with "false" in the body.
Not a 401. Not a 403. A 200 with the string "false".
The frontend must check the response body string, not the status code.
Any frontend change that assumes HTTP conventions will break auth silently.

**Worst case:** A developer updates the frontend to check `response.ok`
(standard HTTP check). Login failures look like successes. 
Anyone can access the system.

**I will automate:** POST /api/login with wrong password — 
assert response is NOT 200, or assert body is not "false".
Document the non-standard response as a bug.

**I will test manually:** What the login screen shows when 
credentials are wrong — is it a clear message or a blank screen?

---

### 5. Salary Data Integrity — GET /api/salary
**What it does:** Returns all employees who have exactly 1 salary record.
Code: `company.filter(data => data["salary"].length == 1)`

**Critical issue found in code:**
Employees with 0 salary records are silently excluded from this list.
An HR admin viewing the salary list has no indication that some 
employees are missing. No warning. No count. Just absent.

**Worst case:** New construction worker is added to the system.
HR forgets to assign salary. Worker doesn't appear in salary list.
Month-end payroll is run. Worker is not included.
Worker receives no pay. Nobody knows why.

**I will automate:** POST /api/employee (create worker) → 
GET /api/salary → verify the new worker appears with a 
"no salary assigned" indicator, not silently absent.

**I will test manually:** The salary list screen — is there any 
visual indicator of employees without salary? Or are they just gone?

---

## What I Deliberately Will NOT Test

**CSS and UI styling:** A misaligned button on the dashboard 
does not affect a worker's pay. Not worth automating.

**Admin portal features** (GET/POST /api/admin/portal, 
/api/admin/project-bid): These are project management features,
not payroll. A broken portal doesn't affect salary calculations.
Manual smoke test is sufficient.

**Country/State/City CRUD** (/api/country, /api/state, /api/city):
Reference data that changes rarely. Manual verification 
during setup is enough. CI automation would be brittle.

**Why this judgment matters:** The team is small, moving fast,
and the Dev Lead doesn't want a 20-minute test suite nobody runs.
If I automate everything, nobody runs the tests.
If I automate only what protects worker pay, 
the tests run in 2 minutes and everyone trusts them.

---

## Real Bug Found During Setup — Before Tests Even Ran

**Bug:** Server crashes with raw MongooseError when DATABASEURL 
is missing from .env file.

**Exact error:** "The uri parameter to openUri() must be a string, 
got undefined."

**This is Incident 1 from the assignment brief** — reproduced locally 
in under 5 minutes. The fix is a startup check:
```javascript
if (!process.env.DATABASEURL) {
  console.error('FATAL: DATABASEURL not set in .env');
  process.exit(1);
}
```
Currently the app starts successfully and crashes only on 
first database call — giving no guidance to the developer.

---

## Summary

| Flow | Automate | Manual | Why |
|---|---|---|---|
| POST /api/salary/:id | ✅ | ✅ | Wrong salary = wrong pay |
| POST /api/employee | ✅ | Partial | Foundation of all data |
| DELETE /api/employee/:id | ✅ | — | Broken by design, needs flagging |
| POST /api/login | ✅ | ✅ | Non-standard response breaks auth |
| GET /api/salary | ✅ | ✅ | Silent omission = missed payroll |
| Admin/portal/project | — | Smoke only | Not payroll-critical |
| Country/State/City | — | Setup only | Reference data, rarely changes |
