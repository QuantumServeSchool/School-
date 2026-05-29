# QuantumServe School Portal
### East Africa School Management System — Uganda · Kenya · Tanzania · Rwanda

---

## What This Is

The staff-facing school management portal. Handles students, fees, exams, staff HR, payroll, inventory, notices, finance, and reporting. Role-gated — each of the 11 staff roles sees only what their position requires.

**This repo is separate from the Parent Portal.** They share the same Supabase backend but are deployed independently.

---

## Repos

| App | Repo | Audience |
|-----|------|----------|
| **School Portal** | `github.com/yourorg/quantumserve-school` | Staff (principal, teachers, bursar, etc.) |
| **Parent Portal** | `github.com/yourorg/quantumserve-parent` | Parents only |

---

## Licence Model

### How it works

1. **QuantumServe provisions a school** by running one SQL command in Supabase:
   ```sql
   SELECT * FROM fn_provision_school(
     'St. Mary''s College Kampala',
     'stmarys-kampala',
     'Uganda',
     'standard'
   );
   ```
   This returns a licence key: `QS-UG-STMARYS-A3F9C2D1`

2. **That key is shared with the school.** The school administrator opens the School Portal and enters it in the onboarding wizard.

3. **Onboarding wizard** (4 steps):
   - Step 1: Enter licence key → verified against `school_licences` table
   - Step 2: Confirm school profile (name, country, currency, exam board)
   - Step 3: Create the first admin user (Principal or System Admin)
   - Step 4: Done — portal unlocks

4. **After activation**, all subsequent logins go directly to the sign-in screen. The licence wizard is never shown again.

### What the licence controls

| Plan | Max Students | Max Staff | Features |
|------|-------------|-----------|----------|
| trial | 100 | 15 | All modules, 30 days |
| standard | 500 | 60 | All modules |
| professional | 2,000 | 200 | All modules + analytics |
| enterprise | Unlimited | Unlimited | All modules + API access |

### Parent portal access is derived from the school licence

- Parents do **not** need a licence
- Parents do **not** go through this portal
- When a school is licensed and a student is admitted into the system, **that student's parent automatically gains parent portal access** using their registered phone number
- No separate parent onboarding step exists
- Revoking a school licence (`is_active = false`) immediately blocks both staff and parent access to that school's data

---

## Roles & Access

| Role | Finance | Exams | Staff HR | Payroll | Settings |
|------|---------|-------|----------|---------|----------|
| System Admin | ✅ Full | ✅ Full | ✅ Full | ✅ Full | ✅ Full |
| Principal | ✅ Full | ✅ Full | ✅ Full | ✅ Full | ✅ Full |
| Deputy Principal | 👁 View only | ✅ Full | ✅ Full | ❌ | ❌ |
| Director of Studies | ❌ | ✅ Full | 👁 View | ❌ | ❌ |
| Class Teacher | ❌ | ✅ Own class | ❌ | ❌ | ❌ |
| Subject Teacher | ❌ | ✅ Own subjects | ❌ | ❌ | ❌ |
| Bursar / Accountant | ✅ Full | ❌ | ❌ | ✅ Full | ❌ |
| Registrar | ❌ | ✅ Full | ❌ | ❌ | ❌ |
| Store Keeper | ❌ | ❌ | ❌ | ❌ | ❌ |
| Librarian | ❌ | ❌ | ❌ | ❌ | ❌ |

Access is enforced at two levels:
1. Navigation — blocked roles never see the menu item
2. Page render — `canAccess(page)` checks role before rendering anything

---

## Supabase Setup

### Step 1 — Run migrations (fresh install only)

In Supabase SQL Editor (`https://supabase.com/dashboard/project/YOUR_PROJECT_ID/sql/new`):

1. Run `QuantumServe_School_Migration_v4.sql` — creates all 39 tables, triggers, RLS, tax functions, BI warehouse
2. Run `QuantumServe_Licence_Onboarding.sql` — creates licence table and provisioning function

### Step 2 — Supabase Auth settings

Go to **Authentication → Settings**:
- Enable **Email** provider
- Set **Site URL** to your deployed domain
- During development: disable email confirmation for instant login

### Step 3 — Provision first school

```sql
SELECT * FROM fn_provision_school(
  'Your School Name',
  'your-school-slug',   -- URL-safe, no spaces
  'Uganda',             -- or Kenya, Tanzania, Rwanda
  'standard'            -- or trial, professional, enterprise
);
```

Copy the `licence_key` from the result. Share it with the school administrator.

### Step 4 — Configure environment

Update the two constants at the top of `index.html`:

```javascript
const SUPABASE_URL  = 'https://YOUR_PROJECT_ID.supabase.co';
const SUPABASE_ANON = 'your-anon-key-here';
```

---

## Mobile Money Configuration (per school)

After a school is provisioned, update their payment details in Supabase:

```sql
UPDATE schools SET
  mtn_momo_paybill    = '123456',       -- Uganda: MTN MoMo business code
  airtel_money_code   = 'SCHOOLNAME',   -- Uganda: Airtel Money merchant
  mpesa_paybill       = '123456',       -- Kenya: M-Pesa paybill
  bank_name           = 'Stanbic Bank',
  bank_account_number = '9030012345678',
  bank_branch         = 'Kampala Road'
WHERE slug = 'your-school-slug';
```

These values appear in both the school portal fee recording screen and the parent portal payment instructions.

---

## Deploy

### Option A — Static hosting (simplest)

Upload `index.html` to any static host:
- Netlify: drag and drop
- GitHub Pages: push to `gh-pages` branch
- Vercel: connect repo, deploy automatically

### Option B — GitHub Pages

```bash
git init
git add index.html
git commit -m "Initial deploy"
git branch -M main
git remote add origin https://github.com/yourorg/quantumserve-school.git
git push -u origin main
```

Enable GitHub Pages in repo Settings → Pages → Source: main branch.

---

## Adding Staff Users

After the principal logs in:

1. Go to Settings
2. Invite staff member — creates a Supabase Auth user and a `school_users` row
3. Assign role (class_teacher, bursar, etc.)
4. Staff member receives email with login link
5. They log in — portal shows only what their role permits

Alternatively insert directly in Supabase:

```sql
-- First create auth user via Supabase dashboard or API
-- Then insert school_users row:
INSERT INTO school_users (school_id, auth_user_id, first_name, last_name, email, role)
VALUES (
  'your-school-uuid',
  'supabase-auth-user-uuid',
  'Robert', 'Kiggundu',
  'r.kiggundu@school.ac.ug',
  'class_teacher'
);
```

---

## Countries & Currencies Supported

| Country | Currency | Exam Board | Mobile Money | Tax |
|---------|----------|------------|-------------|-----|
| Uganda | UGX | UNEB | MTN MoMo + Airtel Money | URA PAYE + NSSF |
| Kenya | KES | KNEC | M-Pesa | KRA PAYE + NSSF + NHIF |
| Tanzania | TZS | NECTA | — | PAYE + NSSF |
| Rwanda | RWF | REB | — | PAYE + RSSB |

---

## Architecture

```
QuantumServe School Portal (this repo)
        │
        ▼
Supabase (ifsuhqnttjxbbmceibgn)
        │
        ├── schools (one row per licensed school)
        ├── school_users (staff, RLS-isolated per school)
        ├── students (RLS-isolated per school)
        ├── fee_payments (immutable, receipt-sequenced)
        ├── exam_results (immutable once released)
        ├── journal_entries (double-entry accounting)
        ├── domain_events (event sourcing)
        ├── bi_fee_daily / bi_enrollment_snapshot (BI warehouse)
        └── job_queue (async SMS, PDF, imports)

QuantumServe Parent Portal (separate repo)
        │
        └── Reads same Supabase backend
            No licence check — access derived from school.is_active
            + student.guardian_phone match
```

---

## File Structure

```
quantumserve-school/
├── index.html          ← entire app (single file)
└── README.md
```

Single-file architecture. No build step. No npm. No framework. Open in browser and it works.
