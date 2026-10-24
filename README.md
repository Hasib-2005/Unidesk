# UniDesk

**One place for a university class: routine, exam schedule, deadlines, notices, notes, results and previous-batch material.**

UniDesk is a self-hosted web app built for university batches. A class representative (CR) posts the
information once, and every classmate sees it on their phone or laptop. Each department or batch gets its own
private space, and each semester is archived when the next one begins.

Built with Node.js, Express and SQLite. There is no build step, no front-end framework and no external database to run.

---

## Contents

- [Features](#features)
- [How it works](#how-it-works)
- [Quick start](#quick-start)
- [Configuration](#configuration)
- [Deployment](#deployment)
- [Backups and recovery](#backups-and-recovery)
- [Security](#security)
- [Tech stack and project layout](#tech-stack-and-project-layout)
- [API overview](#api-overview)
- [Known limitations](#known-limitations)

---

## Features

### Classes (multi-tenant spaces)
- **Create a class** on the sign-up page (name, university, current semester). The creator becomes the class admin.
- **Join a class** with a 6-character class code or an invite link (`/?join=CODE`). Admins and CRs can copy the link from the sidebar.
- **Complete isolation:** every class has its own users, posts, files and Drive folder. People in one class can never read or change another class's data.
- Admins can rename the class and generate a new code if the old one leaks.

### Roles and permissions

| Role | Can do |
|---|---|
| **General** (student) | Read everything in their class, mark deadlines as done, keep private notes, change their password |
| **CR** | Everything a student can, plus post and manage notices, routines, exams, deadlines, notes, results, curriculum and books |
| **Admin** | Everything a CR can, plus approve or remove CRs, reset passwords, delete members, start a new semester, set the Drive folder |

- Separate **General / CR / Admin** login tabs. A CR account works only after the admin approves the request.
- Permissions are enforced on the server, not just hidden in the interface.

### Semesters and archive
- The class has a **current semester**. Admins start the next one from *Class space* with one action.
- Routine, exams, deadlines, notices, class notes and results of the previous semester are **kept in an archive** and can be opened from the *Semester* menu (read only). The new semester starts clean.
- Curriculum, books, the Drive folder and the member list carry over automatically.
- A mistaken switch can be undone with *Make this the current semester*.

### Batch pages (posted by CR/admin, visible to the whole class)

| Page | What it offers |
|---|---|
| **Notices** | Announcements with an *Important* flag, author and date |
| **Class routine** | Weekly timetable (today highlighted) plus dated schedule updates, with a *Show past* toggle |
| **Mid exams / Final exams** | Exam schedule grouped by date with countdown chips, and the official routine PDF or photo displayed on the page |
| **Deadlines** | Assignments, quizzes and projects with due date and time, instructions, a link, and **attached question PDFs/photos**. Each student marks items done for themselves |
| **Class notes** | Lecture notes with course, tags, search and an optional Drive/PDF link |
| **Results** | Result posts per semester and an uploaded result sheet (PDF or photo) shown on the page |
| **Curriculum** | Courses by semester with credits, outline and a syllabus link |
| **Course books** | Book list with author, course and download link |
| **Previous batches** | The batch's Google Drive folder, browsable inside the app |

### Fast data entry for CRs
- **Paste from WhatsApp.** Paste the message a CR would normally send to the group and UniDesk turns it into structured entries, shows a preview table, and lets you fix, remove or save rows. It understands:
  - dates such as `14/10/2026(Wednesday)`, `20 Oct 2026` or a bare weekday heading
  - section headings: `Section A:`, `Section B:`, `Both Section:`
  - times such as `10:00 am`, `1.45pm-3.00pm`, `8.00pm-9.30pm`
  - rooms after `@` and notes in `( )`, for example `DLD Theory@G2(2nd last class)`
  - noise like `#update` or `2/2`; lines it cannot read are listed as skipped
  - weekday mismatches are flagged
  - an *Update* option replaces what was already posted for those dates
- **Bulk entry grids:** add many rows at once, tick several weekdays to repeat a class, duplicate the previous row.
- **Upload an exam routine** as a PDF or screenshot. Optionally, UniDesk reads it and fills the schedule for you (see [Configuration](#configuration)). Every row is editable before saving.
- **Sections (A/B):** entries can be tagged with a section; students filter to their own.

### Files
- PDF, PNG, JPG, WebP and GIF up to 12 MB each, stored inside the database.
- Phone photos are resized in the browser (up to 2400 px) before upload.
- Files are served only to logged-in members of the same class.
- Images show inline and open full size on click; PDFs open in a viewer with a download button.

### Personal space
- **My notes:** a private notebook that nobody else can read, with search and tags.

### Google Drive integration
- Admin pastes a shared Drive folder link once.
- Without an API key, the folder is **embedded** directly.
- With `GOOGLE_API_KEY`, students get a **file browser** with folder navigation and search inside the folder.

### Accounts and recovery
- Sign-up, login, and *Change password* for everyone.
- **Admin password reset:** admins issue a temporary password; the member must choose a new one at next login.
- The site owner can reset any password, including an admin's, from the owner page or with a command-line script.
- Brute-force protection on login and on class-code guessing.

### Owner tools
- `/owner.html`, protected by `OWNER_SECRET`: server statistics, **download a full backup**, **restore from a backup file**, and reset any password.
- **Automatic daily backups** kept on the server disk.
- Upgrading from an older single-batch database is automatic; its data becomes the first class.

### Interface
- Responsive layout with a sidebar on desktop and a bottom navigation bar on phones.
- Automatic light and dark mode.
- Browser auto-translation is switched off so labels are not mistranslated.

---

## How it works

```
Browser (single-page app, plain JS)  --JSON over HTTPS-->  Express server  -->  SQLite (one file)
                                                               |
                                                               +-- Google Drive API   (optional)
                                                               +-- Anthropic API      (optional, exam routine reading)
```

- **Spaces:** every shared record carries a `space_id`. Semester-based records also carry a `term`. Every query is scoped by the logged-in user's space on the server.
- **Authentication:** passwords are hashed with bcrypt; sessions use signed JWTs valid for 30 days. Roles are re-read from the database on each request, so approvals and removals take effect immediately.
- **Storage:** one SQLite database (WAL mode) holds everything, including uploaded files as BLOBs. Backup and restore are therefore a single file.

---

## Quick start

Requires **Node.js 18 or newer**.

```bash
git clone <your-repo-url> unidesk
cd unidesk
npm install
npm start
```

Open <http://localhost:3000>, choose **Create an account**, then **Create a new class**. You become the admin. Share the class code or invite link with classmates.

To also use the owner tools locally:

```bash
# macOS / Linux
OWNER_SECRET=choose-a-password npm start

# Windows PowerShell
$env:OWNER_SECRET="choose-a-password"; npm start
```

then open <http://localhost:3000/owner.html>.

---

## Configuration

All settings are environment variables. None are required to try it locally.

| Variable | Default | Purpose |
|---|---|---|
| `PORT` | `3000` | HTTP port |
| `DB_PATH` | `./unidesk.db` | Database file. **On a host, point this at a persistent disk**, for example `/data/unidesk.db` |
| `OWNER_SECRET` | *(off)* | Enables `/owner.html` (backup, restore, password reset). Use a long random value |
| `JWT_SECRET` | auto-generated | Session signing key. If unset, one is created once and stored in the database |
| `GOOGLE_API_KEY` | *(off)* | Turns *Previous batches* into a searchable file browser (Google Drive API) |
| `ANTHROPIC_API_KEY` | *(off)* | Lets UniDesk read an uploaded exam-routine photo/PDF and propose the schedule |
| `ANTHROPIC_MODEL` | `claude-sonnet-5-5` | Model used for reading exam routines |
| `BACKUP_KEEP` | `3` | Daily backups kept next to the database (`0` disables) |
| `BACKUP_DIR` | `<db folder>/backups` | Where daily backups are written |
| `OWNER_UPLOAD_LIMIT` | `500mb` | Maximum size of a restore upload |

Automatic exam-routine reading is optional and always shows an editable preview before anything is saved.

---

## Deployment

UniDesk runs on any host that can run Node.js and keep a **persistent disk**: Render, Railway, a VPS, and similar.

1. Push the code to GitHub.
2. Create a web service (`npm install` to build, `npm start` to run).
3. Attach a persistent disk or volume mounted at `/data`.
4. Set `DB_PATH=/data/unidesk.db` and `OWNER_SECRET`.

The full step-by-step guide, including moving data from your own computer, is in **[DEPLOY.md](DEPLOY.md)**.
A `render.yaml` blueprint is included.

> Without a persistent disk, the database and every uploaded file are lost on each restart.

---

## Backups and recovery

- **Automatic:** one snapshot per day on the server disk, newest `BACKUP_KEEP` kept. This protects against mistakes, not against losing the disk.
- **Manual (recommended weekly):** `/owner.html` → *Download backup*, and store the file somewhere else.
- **Restore or migrate:** `/owner.html` → *Restore*. The file is validated, staged, and swapped in when the server restarts. Backups taken from older single-batch versions are upgraded automatically.
- **Forgotten admin password:** use the owner page, or run
  `node reset-password.js someone@example.com NewPassword123`.

Do not copy `unidesk.db` by hand while the server runs: recent data may still be in the `-wal` file. Use the backup download instead.

---

## Security

- bcrypt password hashing; signed, expiring session tokens.
- Server-side role checks on every write; every query scoped to the user's class.
- Login lockout per email and class-code guess limiting per IP.
- File uploads limited to an allow-list of types and 12 MB; downloads require login and class membership.
- User-supplied text is escaped before rendering.
- Owner tools are disabled unless `OWNER_SECRET` is set, compare the secret in constant time, and throttle failed attempts.
- Run behind HTTPS in production (Render and Railway provide this automatically).

---

## Tech stack and project layout

**Backend:** Node.js, Express, better-sqlite3, bcryptjs, jsonwebtoken
**Frontend:** a single HTML file with vanilla JavaScript and CSS, no build step

```
unidesk/
├── server.js            API, authentication, spaces, files, backups, owner tools
├── public/
│   ├── index.html       the whole application UI
│   └── owner.html       owner tools page (backup / restore / password reset)
├── reset-password.js    command-line password reset
├── render.yaml          optional Render blueprint
├── DEPLOY.md            hosting guide
└── package.json
```

---

## API overview

All endpoints are under `/api` and, except sign-up and login, require `Authorization: Bearer <token>`.

| Area | Endpoints |
|---|---|
| Account | `POST /register`, `POST /login`, `GET /me`, `POST /password` |
| Class space | `PUT /space`, `POST /space/new-code`, `POST /space/new-term`, `POST /space/set-term`, `PUT /settings` (Drive folder) |
| Members (admin) | `GET /admin/users`, `PUT /admin/users/:id`, `POST /admin/users/:id/reset` |
| Shared data | `GET/POST/PUT/DELETE /:table[/:id]` and `POST /:table/bulk` for `announcements`, `routine`, `exams`, `curriculum`, `books`, `deadlines`, `notes`, `result_posts`. Add `?term=` to read an archived semester |
| Personal data | the same routes for `mynotes` |
| Deadlines | `POST /deadlines/:id/done` (per user) |
| Files | `GET /files`, `POST /files?kind=…`, `GET /files/:id`, `DELETE /files/:id`, `POST /files/:id/extract` |
| Drive | `GET /drive` |
| Owner | `GET /owner/stats`, `GET /owner/backup`, `POST /owner/restore`, `POST /owner/reset` (header `x-owner-secret`) |

---

## Known limitations

- One account belongs to one class. Moving to another class needs a new account.
- No email verification and no "forgot password by email"; password recovery goes through the class admin or the site owner.
- SQLite on a single server suits many classes at a time but is not built for horizontal scaling. Uploaded files share that database, so plan disk space and backups.
- The interface is in English.
- No push notifications or calendar export yet.
- The repository has no automated test suite yet.
