# LEDGERLY API (Vercel + MongoDB)

One serverless function: `POST /api/rpc` with `{ "fn": "...", "args": [...] }`.
Same function names as the old `Code.gs`. Everything except `getMembers` and `verifyPin`
needs `Authorization: Bearer <token>` (returned by `verifyPin`).

```
api/rpc.js           HTTP layer + CORS
lib/handlers.js      all the functions (members, transactions, dashboard)
lib/dispatch.js      token check + routing
lib/auth.js          PIN hashing (scrypt) + signed tokens
lib/ctx.js           MongoDB connection
scripts/import.js    one-time import of the old sheet data
test/                npm test  (runs against an in-memory fake database)
seed/                your members + transactions for the import (never commit; git-ignored)
```

## Setup

1. **Put this folder in its own GitHub repo** (separate from the website repo).
   `seed/` is git-ignored, so it stays on your computer.
2. **Vercel** -> Add New -> Project -> import that repo -> Deploy (framework: Other).
3. **Connect MongoDB to this project**: Vercel project -> Storage -> connect your MongoDB database.
   Then Settings -> Environment Variables: you should see a connection string, usually `MONGODB_URI`.
   If it has a different name, add `MONGODB_URI` with the same value.
4. **Add these environment variables** (Production, Preview and Development):

   | Name | Value |
   |---|---|
   | `JWT_SECRET` | a long random string, 32+ characters (see below) |
   | `ALLOWED_ORIGIN` | your website origin, e.g. `https://yourname.github.io` (no path, no trailing slash) |
   | `APP_TIMEZONE` | optional, defaults to `Asia/Kolkata` |
   | `MONGODB_DB` | optional, defaults to `ledgerly` |

   Make a secret with:  `node -e "console.log(require('crypto').randomBytes(48).toString('base64url'))"`
5. **Redeploy** (Deployments -> ... -> Redeploy) so the variables take effect.
6. **Import your data (once)**, on your computer, inside this folder:
   ```
   npm install
   npx vercel link
   npx vercel env pull .env.local
   npm run import
   ```
   It prints `Imported 6 members and 475 transactions.` It refuses to run twice unless you add `--force`.
7. **Point the website at it**: in the website repo, edit `js/config.js`:
   `API_BASE_URL: 'https://<your-project>.vercel.app/api'`

## Security notes

- PINs are stored hashed. 5 wrong PINs lock that member for 15 minutes.
- Logins last 24 hours. Admin rights are checked on the server on every call.
- Only an Admin can add members. A member can edit/delete themselves; an Admin can edit/delete anyone
  (the last Admin cannot be deleted).
- In "Edit member", the "PIN" box is YOUR OWN PIN, as confirmation.
- `Banker` role (Sarath) is kept as is and currently has normal user permissions.
