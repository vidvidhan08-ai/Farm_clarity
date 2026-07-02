# Farm Profit Clarity — Private Portal

A private, login-only web app for multi-crop farm profitability analysis.
Nothing is public — every page and every API endpoint requires signing in
first. Built for a small team of **3 named users**.

## What this portal does

- 🔒 **Restricted login** — only 3 pre-created accounts can sign in. No
  public sign-up, no guest access. Every route except the login page
  itself requires an authenticated session.
- 📤 **Upload data** — upload source files (CSV, XLSX, PDF, etc.). File
  bytes are stored **directly in the database**, not on the filesystem,
  and are private to the account that uploaded them.
- 📊 **Generate reports** — the existing multi-crop financial analysis
  (break-even, margins, sensitivity tables, risk indicators, etc.).
- 💾 **Save reports** — saved to the database, scoped to your account.
  One user cannot see another user's reports or files.
- ⬇️ **Download PDFs** — every saved report can be exported as a
  formatted PDF, generated server-side.

## Requirements

- **Node.js 22.5 or newer** (uses the built-in `node:sqlite` module — no
  native compilation, no separate database server to install).

## Project structure

```
farm-profit-clarity/
├── package.json
├── .env.example        # copy to .env to customize accounts/secrets
├── server/
│   ├── index.js          # Express app, auth, all routes
│   ├── db.js              # SQLite layer: users, reports, uploads, sessions
│   ├── session-store.js   # SQLite-backed login sessions (survive restarts)
│   ├── seed-users.js      # creates the 3 accounts on first run
│   ├── set-password.js    # CLI to change a user's password
│   ├── compute.js         # crop math shared with the PDF export
│   └── pdf.js              # server-side PDF report generation
├── public/
│   ├── login.html          # the ONLY page reachable without a session
│   └── index.html           # the app (form / uploads / history / report)
└── data/                     # created automatically — the .db file lives here
```

## Setup & run

```bash
cd farm-profit-clarity
npm install
npm start
```

Open **http://localhost:3000** — you'll be redirected straight to
`/login.html` since nothing else is accessible while logged out.

### First run: your 3 accounts

On the very first start (empty database), the server creates 3 accounts
and prints their credentials to the console:

```
First run detected — created 3 login accounts:
  • username: admin   password: ChangeMe123!
  • username: user2   password: ChangeMe234!
  • username: user3   password: ChangeMe345!
```

**Change these before giving anyone real access:**

```bash
npm run set-password -- admin <new-strong-password>
npm run set-password -- user2 <new-strong-password>
npm run set-password -- user3 <new-strong-password>
```

Alternatively, copy `.env.example` to `.env` and set your own
`USER1_USERNAME` / `USER1_PASSWORD` / `USER1_NAME` (and `USER2_*`,
`USER3_*`) **before the very first start** — those values are only used
the first time the database is empty.

## Deploying it "online" for real use

- Set `SESSION_SECRET` in `.env` to a long random string.
- Set `NODE_ENV=production` once the site is behind HTTPS (this makes
  the login cookie `Secure`, so it's only sent over encrypted
  connections).
- Put the app behind a reverse proxy (Nginx, Caddy, or your host's own
  proxy) that terminates TLS and forwards to this Node process.
- Keep the `data/` folder on persistent storage — it holds the entire
  database (accounts, reports, and uploaded files).
- Nothing in `public/` is served until a session cookie is present, so
  there's no separate step to "hide" files — the server enforces it.

## How access control works

- Every request (page or API) passes through a session check **before**
  it can reach static files or API routes — `public/login.html` and the
  `/api/login` endpoint are the only two exceptions.
- Sessions are stored server-side in the `sessions` table (not just in
  the cookie), so a session can be invalidated at any time and survives
  server restarts.
- Reports and uploads are tagged with the owning `user_id` and every
  query filters on it — one account can never read, download, or delete
  another account's data.

## API reference (all require login except `/api/login`)

| Method | Endpoint                     | Description                        |
|--------|-------------------------------|--------------------------------------|
| POST   | `/api/login`                   | Sign in                             |
| POST   | `/api/logout`                  | Sign out                            |
| GET    | `/api/me`                      | Current logged-in user              |
| GET    | `/api/reports`                 | List your saved reports             |
| GET    | `/api/reports/:id`             | Get one report                      |
| POST   | `/api/reports`                 | Save a report                       |
| DELETE | `/api/reports/:id`             | Delete a report                     |
| GET    | `/api/reports/:id/pdf`         | Download a report as a PDF          |
| GET    | `/api/uploads`                 | List your uploaded files            |
| POST   | `/api/uploads`                 | Upload a file (multipart `file`)    |
| GET    | `/api/uploads/:id/download`    | Download an uploaded file           |
| DELETE | `/api/uploads/:id`             | Delete an uploaded file             |

## Notes

- All report math (break-even, margins, sensitivity, risk flags) runs
  identically on screen and in the PDF — the PDF uses the same formulas
  (`server/compute.js`), so the two never disagree.
- Uploaded files are capped at 15 MB each; adjust the limit in
  `server/index.js` (`multer({ limits: { fileSize: ... } })`) if needed.
- The SQLite database file lives at `data/farm_profit_clarity.db` and
  contains everything: accounts, reports, uploaded file bytes, and
  active login sessions. Back it up like any other file.
