# Site Ledger — Site Accounts Register

A single-page web app for a civil engineer to track money and materials across
every client's project — Labour, Material, Supplier and Client accounts, all
linked together, with real Google login and live cloud sync.

No install, no build step, no server to maintain. One HTML file, hosted for
free on GitHub Pages, backed by a free Firebase project for login + storage.

---

## What it does

- **Daily Entry** — one page, five sections (Client, Labour, Material, Supplier,
  Engineer). Pick a client at the top and everything else you fill in that
  same save automatically links to that client's project — no repeated
  selection needed.
- **Every section keeps its own ledger** — Labour (Mason, Centring, Concrete…),
  Material (Cement, Steel, Sand…), Supplier (named suppliers), Client (each
  client gets two accounts — see naming note below), and Engineer (the
  engineer's own running total of everything paid out, plus personal expenses).
- **Automatic linking** — any Labour/Material/Supplier entry automatically
  mirrors into the Engineer account (since that money came out of the
  engineer's own pocket), and optionally into a client's project cost, and
  (for Material) optionally into a specific supplier's ledger.
- **Client account naming** — for a client named "Sampath":
  - **"Sampath"** = Engineer Selavu (project cost — what the engineer spent
    on that client's project, mostly filled in automatically via linking)
  - **"Sampath 2"** = Client Selavu (direct payments to/from the client)
- **Charts on every page** — Credit vs Debit donut chart, plus a "where the
  money went" category-breakdown chart on Engineer and client-project pages.
- **Date filter everywhere** — defaults to today's entries; switch to any
  other date, or "All dates" to see the full chronological history.
- **Print & PDF** — every page has a Print button that produces a clean,
  letterheaded statement (business name/contact you set once in Profile).
- **Excel/CSV export** — full workbook export (one sheet per account) at any time.
- **Real login, real cloud sync** — sign in with Google, data lives in
  Firestore under your account, syncs across any device, works offline and
  catches up when back online.

## Setup (one-time, ~20 minutes)

This app needs two free accounts to run with real login:

1. **Firebase** (Google's backend) — for login and the database.
2. **GitHub Pages** — to host the file, since Google Sign-In requires a real
   web address (it won't work on a file you just double-click).

Full step-by-step instructions, including the exact security rules to paste
in, are in **`Firebase_Setup_Guide.txt`** in this repo. Follow it once; after
that, using the app is just opening the link and signing in.

## Files in this project

| File | Purpose |
|---|---|
| `Site_Ledger_App.html` | The entire app — this is the file you host |
| `Firebase_Setup_Guide.txt` | One-time setup steps for login + database |
| `PROJECT_NOTES.md` | Full technical write-up of how the app works (for future development, by a human or an AI) |
| `README.md` | This file |

## Making changes later

The whole app is one HTML file — open it in any text editor. If you (or an
AI assistant) need to understand how it's built before changing anything,
read **`PROJECT_NOTES.md`** first — it explains the data model, the linking
logic, and a few non-obvious design decisions that aren't visible just from
reading the code.

After any edit, remember: your Firebase config (the six values near the top
of the script) needs to already be in the file before you upload it — a
fresh copy of the file won't have it.
