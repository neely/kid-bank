# 🏦 Kid Bank

A lightweight family banking app built with two static HTML files, a JSON ledger, and zero backend. Kids see their balances and transaction history. Parents add, edit, and delete transactions through a PIN-gated admin interface. Everything lives in a GitHub repo and is served via Cloudflare Pages.

---

## How It Works

### The Files

| File | Purpose |
|---|---|
| `index.html` | Kid-facing dashboard — read only, no auth |
| `bank.html` | Parent admin app — requires GitHub PAT to use |
| `ledger.json` | All transactions for all kids, single source of truth |
| `investments.json` | Optional investment balances per kid |

### The Data

`ledger.json` is a flat JSON array. Every transaction — for every kid — lives in this one file:

```json
[
  {
    "id": "adam-opening",
    "kid": "Adam",
    "date": "2024-12-15",
    "description": "Opening balance",
    "amount": 836.06,
    "type": "deposit",
    "seq": 0
  },
  {
    "id": "adam-1",
    "kid": "Adam",
    "date": "2025-01-03",
    "description": "ARC-170",
    "amount": 76.29,
    "type": "withdrawal",
    "seq": 1
  }
]
```

**Field notes:**
- `id` — unique string, format `kid-timestamp` for new entries
- `kid` — kid's name, case-insensitive (app normalizes on load)
- `date` — `YYYY-MM-DD` format
- `amount` — always a positive number regardless of type
- `type` — `"deposit"` or `"withdrawal"`
- `seq` — tiebreaker for same-day transactions, integers starting at 0

**Balances are never stored.** Every balance shown in the app is computed fresh on load by sorting each kid's transactions by `date` then `seq` and walking deposits and withdrawals. Opening balance entries (seq: 0) handle the starting point.

`investments.json` is a separate array for investment accounts that are tracked but not included in the spendable balance:

```json
[
  { "kid": "Adam", "name": "SPY", "value": 142.30, "last_checked": "2025-04-20" },
  { "kid": "William", "name": "Roth IRA", "value": 312.00, "last_checked": "2025-04-18" }
]
```

The `name` field is freeform — SPY, Fidelity, Roth IRA, whatever fits.

---

## The Two Apps

### `index.html` — Kid Dashboard

- Fetches `ledger.json` and `investments.json` directly from the GitHub raw URL (no auth required since the repo is public)
- Landing screen shows all kids as large stacked cards with their balance and investment line
- Tap a card to slide into that kid's transaction history
- Transactions shown newest first with relative dates ("3 days ago", "Last week")
- Filter by All Time / This Year / 30 Days, plus live search
- Fully client-side — no server, no cookies, no login

### `bank.html` — Parent Admin

- **PAT-gated:** on every new browser session, a fullscreen prompt requires a GitHub Personal Access Token before anything loads. The token is validated against the GitHub API before the app renders. It is stored in `sessionStorage` only — it clears when the tab closes, never touches `localStorage` or any server.
- Same kid card landing as `index.html`, but tapping a card opens the management view
- **Add transaction:** tap the `+ Add` button for a bottom sheet with deposit/withdrawal toggle, amount, description, and date (defaults to today)
- **Edit transaction:** tap any transaction row to open the same sheet pre-filled with existing values
- **Delete transaction:** hard delete from the edit sheet with a confirmation prompt
- **Edit investment:** small button in the kid header opens a sheet to update name, value, and last-checked date
- All writes go through the GitHub Contents API — the app fetches the current file SHA, patches the in-memory ledger, and PUTs it back. The UI updates immediately from memory rather than re-fetching from the CDN, which avoids propagation delay issues.

---

## Security Notes

**This repo is public.** That means:

- Anyone can read `ledger.json` and `investments.json` directly from GitHub or the raw URL
- `index.html` is intentionally public — it's designed for kids to bookmark
- `bank.html` is protected only by the PAT gate — someone who finds the URL can reach the login screen, but cannot do anything without a valid GitHub token that has write access to the repo
- The PAT itself is never stored persistently — only in `sessionStorage` for the duration of the browser tab
- On a phone, the keychain entry for the PAT (if saved) is protected by Face ID or device passcode, which serves as the practical "password" for the admin app

**What this is not:** a bank. There is no encryption of transaction data, no audit log beyond Git history, and no access control beyond the PAT. It is a fun family ledger that happens to look nice.

**Git history is your audit log.** Every write to `ledger.json` and `investments.json` creates a GitHub commit with a descriptive message (e.g. `Add transaction for Adam`, `Delete transaction adam-1234 for William`). The full history of every change is preserved in the repo.

---

## Deployment

This app is deployed as two static HTML files on Cloudflare Pages, connected to this GitHub repo, with a custom domain managed through a separate DNS registrar.

### Stack

```
GitHub repo (source of truth)
    ↓  automatic deployment on push to main
Cloudflare Pages (hosts index.html and bank.html)
    ↓  custom domain via CNAME
Domain registrar (DNS, e.g. Squarespace, Namecheap, etc.)
```

### Step-by-step Cloudflare Pages setup

1. Go to [Cloudflare Pages](https://pages.cloudflare.com) and create a new project
2. Connect your GitHub account and select your forked repo
3. No build command needed — set the build output directory to `/` (root)
4. Deploy. Cloudflare will give you a `*.pages.dev` URL
5. In your Pages project → **Custom Domains** → add your domain (e.g. `kidbank.yourdomain.com`)
6. Cloudflare will tell you what CNAME record to add

### DNS setup at your registrar

Add a CNAME record at your domain registrar:

| Type | Name | Value |
|---|---|---|
| CNAME | `kidbank` | `your-project.pages.dev` |

> ⚠️ Both steps are required — the CNAME at your registrar AND adding the custom domain inside Cloudflare Pages. Missing either one causes a 1001 DNS error.

After DNS propagates (usually minutes if your registrar is fast, up to 48 hours worst case), your app will be live at your custom domain.

---

## Making Your Own

### 1. Fork the repo

Fork `github.com/neely/kid-bank` to your own GitHub account.

### 2. Update the kids

In both `index.html` and `bank.html`, find the `KIDS` config object near the top of the `<script>` block and update it:

```javascript
const KIDS = {
  alice:   { color: '#4a9eff', label: 'Alice',   init: 'A' },
  ben:     { color: '#c44aff', label: 'Ben',     init: 'B' },
  charlie: { color: '#ff8c4a', label: 'Charlie', init: 'C' },
};
```

- The key (e.g. `alice`) must be lowercase and match the `kid` field in your ledger (case-insensitive — the app normalizes on load)
- `color` is any hex color
- `label` is the display name
- `init` is the one-letter initial shown in the transaction dot

**How many kids can it hold?** The ledger supports any number of kids — it's just a flat array filtered by name. The landing screen layout is designed for three stacked cards and looks best with 2–4. With more than four you'd want to update the CSS to allow the cards section to scroll.

### 3. Update the repo owner in both HTML files

In both `index.html` and `bank.html`, find:

```javascript
const OWNER = 'neely';
const REPO  = 'kid-bank';
```

Change `OWNER` to your GitHub username and `REPO` to your fork's repo name.

### 4. Create your ledger files

Replace `ledger.json` with your starting data. At minimum, create an opening balance entry for each kid:

```json
[
  { "id": "alice-opening", "kid": "Alice", "date": "2025-01-01", "description": "Opening balance", "amount": 0, "type": "deposit", "seq": 0 },
  { "id": "ben-opening",   "kid": "Ben",   "date": "2025-01-01", "description": "Opening balance", "amount": 0, "type": "deposit", "seq": 0 }
]
```

Create `investments.json` as an empty array to start:

```json
[]
```

### 5. Create a GitHub PAT

In GitHub → Settings → Developer Settings → Personal Access Tokens → Fine-grained tokens:

- Set repository access to your fork only
- Grant **Contents: Read and Write** permission
- No other permissions needed

Save the token somewhere safe (your password manager). This is what you'll enter each session when opening `bank.html`.

### 6. Deploy to Cloudflare Pages

Follow the deployment steps above, pointing to your forked repo.

### 7. Optional: rename the app

Search both HTML files for `Neely Bank` and `Neely Family Bank` and replace with your family name.

---

## App Icon

Both HTML files include a base64-encoded home screen icon in the `<head>`:

```html
<link rel="apple-touch-icon" href="data:image/png;base64,...">
<link rel="icon" type="image/png" href="data:image/png;base64,...">
```

The `apple-touch-icon` is used when saving to the iOS home screen. To use a custom icon, replace the base64 string with your own 512×512 PNG encoded as base64.

---

## Local Development

Since everything is static HTML, you can open the files directly in a browser — but note:

- `index.html` fetches from the GitHub raw URL, so it works as-is as long as you have internet
- `bank.html` calls the GitHub API, so it also works directly — just enter your PAT when prompted
- No build step, no Node, no dependencies, no local server required

---

## License

Do whatever you want with this. It's a family project.
