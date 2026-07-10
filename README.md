# J.T. Reffell Memorial French Friendship Primary — SIMS

A real, complete, deployable school information system: a Node.js + Express
API on a real PostgreSQL database, serving a real frontend (`public/index.html`)
that talks to it over HTTP. One deployment, one URL — no Claude.ai required.

## What's real here
- **Real accounts**: passwords hashed with bcrypt, never stored in plain text
- **Real sessions**: JWT tokens, checked and role-enforced on every API request (admin / teacher / parent)
- **Real database**: PostgreSQL — students, fees, payments, grades, attendance, timetable, staff, salaries, payslips, SMS log, enquiries all persist for real
- **Real frontend**: `public/index.html` is served directly by this same server and talks to the API with `fetch()` — open the server's URL in a browser and it's a working app, not a demo
- **Real PDFs**: pay slips download as actual generated PDF files
- **SMS**: safe by default (simulated + logged), becomes real the moment you add a provider API key — see below

## 1. Local setup (fastest way to try it)

Requires Docker.

```bash
cp .env.example .env
docker compose up --build
```

Then, in a second terminal:

```bash
docker compose exec api npm run migrate
docker compose exec api npm run seed
```

The seed step prints a real admin email + password — **open `http://localhost:4000` in a browser and log in with it.** Change that password immediately (via `POST /api/auth/change-password`) in any deployment a real school will use.

## 2. Setup without Docker (if you already have Postgres)

```bash
npm install
cp .env.example .env   # fill in your real DATABASE_URL and JWT_SECRET
npm run migrate
npm run seed
npm start
```

Open `http://localhost:4000` — the app itself, fully working.

## 3. Deploying somewhere real

**Railway / Render (recommended for a school with no dedicated IT team)**
1. Push this folder to a GitHub repo
2. Create a new Postgres database on Railway/Render — copy the connection string into `DATABASE_URL`
3. Create a new Web Service pointing at the repo, set the env vars from `.env.example`
4. Run `npm run migrate` and `npm run seed` once against the deployed service
5. Your app is live at a public URL like `https://jt-reffell.up.railway.app` — that URL serves both the API and the frontend

**Your own server (VPS)**
1. Install Node 20+ and Postgres, or run `docker compose up -d --build` on the server itself
2. Put a reverse proxy (nginx/Caddy) in front with HTTPS (Let's Encrypt)
3. Run migrate + seed as above

## 4. Turning on real SMS

Every SMS is logged to the real `sms_log` table but not actually sent by default — `status` is `"simulated"`. To send real texts to parents:

1. Create an account at [Africa's Talking](https://africastalking.com) (works well for Sierra Leone numbers) and get an API key
2. `npm install africastalking`
3. In `.env`, set `SMS_PROVIDER=africastalking`, `AFRICASTALKING_API_KEY`, `AFRICASTALKING_USERNAME`
4. Restart the server — `POST /api/sms` now sends real texts and logs `status: "sent"`

## 5. Creating logins for teachers and parents

The admin account can create more logins:

```bash
curl -X POST http://localhost:4000/api/auth/register \
  -H "Authorization: Bearer <admin-token>" -H "Content-Type: application/json" \
  -d '{"name":"Mrs. Isatu Kamara","email":"i.kamara@jtreffell.edu.sl","password":"a-real-password","role":"teacher","cls":"Class 1"}'
```

Roles are `admin`, `teacher`, or `parent`. A teacher's `cls` scopes their default view to that class.

## 6. Known simplifications (be aware of these before relying on it)
- **Parent accounts aren't linked to a specific child yet** — a parent role currently sees a general dashboard and can submit/read enquiries, but fee/grade/attendance views are admin/teacher-only. Linking a parent login to their specific child (a `students.guardian_user_id` column) is a natural next step if you need parents to see their own child's records online.
- **No password reset flow** — if someone forgets their password, an admin needs to be added to reset it directly in the database or via a new endpoint.
- **PAYE tax is entered manually**, not calculated from Sierra Leone's tax brackets.
- The auth token lives in memory in the browser tab (not localStorage), so refreshing the page requires logging in again. This was a deliberate choice so the same file still safely previews inside Claude.ai; `public/index.html` has a comment marking exactly where to add `localStorage` persistence for a real deployment.

## 7. API overview

All endpoints except `/api/health` and `/api/auth/login` require `Authorization: Bearer <token>`.

| Area | Endpoints |
|---|---|
| Auth | `POST /api/auth/login`, `GET /api/auth/me`, `POST /api/auth/register` (admin), `POST /api/auth/change-password` |
| Students | `GET/POST /api/students`, `PUT/DELETE /api/students/:id` |
| Fees | `GET /api/fees` (admin), `GET/POST /api/fees/:studentId/payments`, `PUT /api/fees/:studentId` |
| Grades | `GET/POST /api/grades` |
| Attendance | `GET/POST /api/attendance`, `POST /api/attendance/bulk` |
| Timetable | `GET/PUT /api/timetable` |
| Staff | `GET/POST/DELETE /api/staff` (admin) |
| Payroll | `PUT /api/payroll/salary/:staffId`, `POST/GET /api/payroll/payslips`, `GET /api/payroll/payslips/:id/pdf` |
| SMS | `GET/POST /api/sms` |
| Enquiries | `GET/POST /api/enquiries`, `GET/POST /api/enquiries/:id/messages` |
