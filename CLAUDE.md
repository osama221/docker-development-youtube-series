# ONE WAY Accounting — notes for Claude Code

The owner is Osama. He writes in Egyptian Arabic and wants short, structured replies in Arabic. He is not a developer: explain in business terms, not code.

## What this is
An online accounting web app for ONE WAY Real Estate (brokerage) plus an Android WebView app.
- Backend: one Vercel serverless function `api/index.js`. Data is JSON in private Vercel Blob storage: `db/main.json` (all tables and the change log) and `db/users.json`. Writes go through `mutate()` with ETag retries. A warm instance keeps an in-memory copy and re-reads with `ifNoneMatch`.
- Frontend: one file, `public/index.html` (vanilla JS, Arabic RTL, brand colors gray #818284 and green #94BC37, logo `public/logo.png`). Global state lives in `S`; `go(tab)` switches tabs; `shell()` renders; forms are in `FORMS`.
- Excel export: `lib/export.js` (ExcelJS).
- Android: `android/`, built with `android/build.sh`, which writes `public/app/OneWayAccounting.apk` and `public/app/version.json`. Current version is 1.5 (versionCode 6). Most changes need no new APK because the app loads the website. Rebuild only when native Java/manifest code changes.
- README.md has the full business rules: treasury types, commission payouts per person, receipts, permissions and presets, protection, backups, month books, data rules.

## Deploy
Pushing to `main` does NOT auto-deploy. After pushing, deploy production with the Vercel MCP tool `create_deployment`:
- teamId `team_BZdxdeS9JY5OFAq2egYZyh6a`
- body `{"name":"oneway-data","project":"prj_Gog1V0RnBEtvBN07xYDOZ0qiwqgN","target":"production","gitSource":{"type":"github","org":"osama221","repo":"oneway-data","ref":"main"}}`

Then poll `get_deployment` until it is READY. The live site is https://oneway-data.vercel.app. It only works from Egypt (country check), so it can't be tested live from outside Egypt.

The agent token can't create env vars, so ask Osama to add any he needs in the Vercel dashboard. `AUTH_SECRET` is already set. `CRON_SECRET` is optional; without it the daily backup cron (vercel.json, 23:00 UTC) accepts only Vercel's scheduler, once a day.

## Run and test locally
- Dev server: `LOCAL_STORE=1 AUTH_SECRET=0123456789abcdef0123456789abcdef PORT=3999 node scripts/dev.js`. Data goes to `.local-db/`; delete it for a clean start. Local admin is `osamag` / `12345678`.
- Full suite: `bash tests/run.sh [screenshotDir]`. It runs three passes, all of which must pass before deploying:
  - `tests/e2e.js` (Playwright, real UI) on the sheet data
  - `tests/fresh.js` on the fresh start (October import)
  - `tests/qa.js`, API regression checks
- Kill dev servers by their PID; never use `pkill -f scripts/dev.js`, because it kills the shell too.
- Data changes for production go in `data/migrations/*.json`. They are applied once and tracked in `db.imports`, and the order is set in `MIGRATIONS` in api/index.js.

## Rules Osama asked for (keep them)
- Treasury income types: تمويل جاري شريك، عمولة وحدة، حجز وحدة، مقدم وحدة.
- Treasury expense types: صرف عمولة، مرتب، سلفة، جاري شريك، بونص، نثريات، كامباين، صافي مقدم وحدة، استرداد حجز.
- `استرداد حجز` is offered only when a unit has a booking, and only up to what remains of it.
- Commissions are paid per person, each with its own voucher. Unpaid amounts stay in «العمولات المستحقة». The unit form's اتصرف / لم يُصرف asks who paid and when.
- Voucher numbers are automatic. Every movement has a printable receipt, and the preview matches the printed page.
- Four wrong passwords lock only the account; the device or office IP is never blocked. The admin's own lock lasts 30 minutes. The site is Egypt-only.
- Each month's closing balance is the next month's opening balance. The imported «رصيد مرحل» row counts as opening balance, not income.
- Excel export comes from the menu, limited to the sections the user can see. There is a daily backup, and the last 60 days are kept.
- When adding a feature, also add an entry to `CHANGES` in index.html (shown in the «التحديثات» screen).

## Open ideas Osama hasn't decided on
- A «رد سلفة» income type (cash returned by an employee lowers their advance balance).
- Moving the function and Blob storage from iad1 to fra1 for speed. This needs data migration at a quiet time.
- A «قفل الشهر» (lock month) button for the admin.

## Secrets
Keep this repo private: it contains the Android signing keystore (`android/oneway.keystore`), and its password is in `android/build.sh`. Never paste secrets into chats or commits.
