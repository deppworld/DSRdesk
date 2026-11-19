# Andromeda DSR – Daily Sales Record system

A web app that replaces the per-employee DSR Excel sheets. Data lives in one central PostgreSQL database. Management sees every update live, and employees only ever see their own records.

The full **Tutorial & Troubleshooting guide** is a separate document. This file is the short version.

## Quick start (Windows)

1. Install **Node.js LTS** (version 20 or newer) from https://nodejs.org. Keep the default options.
2. Double-click **`Start-DSR.bat`**. The first run downloads the components (it needs internet, 1–3 minutes) and creates the database.
3. The console window shows the **owner username `admin` and a temporary password**. The same details are saved in `data\FIRST_LOGIN.txt`.
4. Your browser opens http://localhost:8080. Sign in, choose your own password, then delete `data\FIRST_LOGIN.txt`.
5. Other computers on the same network open the address shown as **"Other computers on your network"**, for example `http://192.168.1.20:8080`.

Keep the black console window open. Closing it stops the system.

## Folder contents

| Path | What it is |
|---|---|
| `Start-DSR.bat` | Starts the system (and installs it on the first run) |
| `Backup-Now.bat` | Makes a backup immediately |
| `Reset-Admin-Password.bat` | Gives the `admin` account a new temporary password |
| `.env` | Settings and secret keys (created on the first run). **Keep it private and back it up.** |
| `data\pg` | The built-in PostgreSQL database files |
| `backups\` | Nightly backups (`dsr-backup-*.json.gz`). The last 30 are kept. |
| `server\` | Server code (Node.js, Express, Socket.IO) |
| `public\` | The web pages (Preact, no build step) |

## Commands

```
npm start                      start the server
npm run backup                 back up now
npm run restore -- <file>      restore a backup (replaces ALL data – asks for YES)
npm run reset-admin            new temporary password for "admin"
npm run reset-admin -- <user>  …or for any username
```

## Settings in `.env`

| Key | Meaning |
|---|---|
| `PORT` | Web port (default 8080) |
| `DATABASE_URL` | Empty = built-in database. Set it to use an external PostgreSQL, e.g. `postgres://dsr:pw@db-host:5432/dsr` |
| `APP_TIMEZONE` | Business time zone for dates and hourly slots (default `Asia/Kolkata`) |
| `COOKIE_SECURE` | `true` when the site is served over HTTPS |
| `TRUST_PROXY` | `1` behind nginx, Caddy or IIS, so the real client IPs are logged |
| `BACKUP_DIR`, `BACKUP_KEEP`, `BACKUP_TIME` | Where, how many, and when the nightly backups run |
| `JWT_*`, `DATA_ENCRYPTION_KEY` | Secrets, generated automatically. **Never share them. Losing `DATA_ENCRYPTION_KEY` makes stored PAN numbers unreadable.** |

## Security summary

- bcrypt password hashing (cost 12). Temporary passwords must be changed at first sign-in.
- 15-minute access tokens plus rotating 7-day refresh tokens in an httpOnly, SameSite=strict cookie. If a stolen refresh token is reused, all of that user's sessions end.
- 5 wrong passwords lock the account for 15 minutes. Admins are alerted, and sign-ins from a new IP are reported.
- Role checks on every API call. Employee queries are always filtered by `owner_id = current user`, so changing an id in the URL returns "Not found" and is written to the audit log as `access_denied`.
- Real-time events go only to the right people: admins get team activity, and employees only get events about themselves.
- PAN numbers are stored with AES-256-GCM encryption. Viewing a full PAN is written to the audit log.
- Every sign-in, change, export, role switch and blocked access is logged with IP address and device.
- Helmet security headers (Content Security Policy), request rate limiting on sign-in, and parameterised SQL everywhere.

## Architecture

```
Browsers (CSO / HRE / Admin) ──HTTPS/WebSocket──► Node.js (Express API + Socket.IO)
                                                    │  RBAC middleware, audit log, scheduler (every minute)
                                                    ▼
                                         PostgreSQL (built-in or external)
                                                    │
                                                    ▼
                                      backups\ (nightly JSON.gz, 30 kept)
```

To grow from 10 to 500+ users, the code stays the same. Move the database to a managed PostgreSQL (set `DATABASE_URL`), run the app on a cloud VPS behind HTTPS (Caddy or nginx), and add more app instances with the Socket.IO Redis adapter if needed.
