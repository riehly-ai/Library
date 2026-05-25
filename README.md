# Work Library — Borrow Register

A small, single-page book-borrow tracker for an in-office library. Data lives
as `books.json` in a GitHub repo; every borrow/return is a commit. Hosted
on Netlify.

- **Frontend:** static HTML (`index.html` + `library.css` + a couple of JSX files).
- **Backend:** one Netlify Function that reads/writes `books.json` via the GitHub Contents API.
- **Persistence:** the JSON file in your repo. You get free version history and undo via git.
- **Auth:** none. Anyone with the URL can edit. Use a private/obscure subdomain or share it only inside your company.

## What's in here

```
index.html                       Production app (the page you deploy)
library.css                      All styles
library-shelf.jsx                The "Open Stacks" UI (book spines on shelves)
library-data.jsx                 Data hook — useRemoteLibrary() talks to the function
netlify.toml                     Netlify build/function config
netlify/functions/library.js     The serverless function (reads + writes books.json)
books.example.json               What an empty library looks like ([])

directions.html                  ARCHIVE: the original 3-direction design canvas
library-stickerbook.jsx          ARCHIVE: Direction A (not used in production)
library-ledger.jsx               ARCHIVE: Direction C (not used in production)
design-canvas.jsx                ARCHIVE: design-canvas wrapper (used by directions.html)
```

## One-time setup

### 1. Create a GitHub repo to hold the data

This can be the same repo as the app code, or a separate private repo. Either
way it just needs to contain a file called `books.json`. Create that file with
the contents `[]` (an empty JSON array) and commit it.

> Tip: it's cleaner to put `books.json` in the **same repo** as the app —
> one repo to deploy from, one to read/write. The instructions below assume that.

### 2. Make a GitHub fine-grained Personal Access Token

1. Go to <https://github.com/settings/tokens?type=beta> → **Generate new token**.
2. **Token name:** something like `library-register`.
3. **Expiration:** 1 year is reasonable (set a calendar reminder to rotate).
4. **Repository access:** "Only select repositories" → pick the repo above.
5. **Repository permissions:** `Contents` → **Read and write**. Leave everything else as "No access".
6. Generate and **copy the token** (you only see it once).

### 3. Deploy to Netlify

1. Push this folder to your GitHub repo.
2. In Netlify: **Add new site → Import an existing project →** select the repo.
3. **Build settings:** Netlify reads `netlify.toml`; you shouldn't need to change anything. Publish directory: `.`, Functions directory: `netlify/functions`.
4. **Site settings → Environment variables**, add:
   - `GITHUB_TOKEN` = the PAT from step 2
   - `GITHUB_REPO` = `your-username/your-repo-name`
   - `GITHUB_BRANCH` = `main` *(optional, defaults to `main`)*
   - `GITHUB_PATH` = `books.json` *(optional, defaults to `books.json`)*
5. Trigger a deploy (or push a commit). Once it's live, open the URL and the library should load.

That's it. The page now reads `books.json` from your repo on every load, and
every change you make in the UI gets committed back to that file as a new
commit on `main`.

### 4. (Optional) Restrict access

There's no built-in login — anyone with the URL can borrow/return books.
Options if you want a small fence:

- **Obscure URL:** rename the Netlify site to something not-guessable (`books-acme-internal-abc123.netlify.app`).
- **Netlify Password Protection:** Netlify Pro feature — Site settings → Visitor access → set a site password.
- **IP allow-list / Netlify Identity:** also Pro features.

For ~30 people in your team, an obscure URL shared in Slack is usually enough.

## How saving works

- The page does an optimistic update locally, then debounces 600ms and POSTs the full books array to the function.
- The function does a `PUT /repos/.../contents/books.json` with the new content and the `sha` of the version we last loaded.
- If two people change something at the same time, GitHub returns a 409. The function reloads the latest file and sends it back; the page reloads its state and shows a small "reloaded" badge.
- If the function is unreachable, the page falls back to localStorage so the user keeps working; it retries the save automatically.

A small status pill ("saved" / "saving…" / "offline" / "save failed") sits in
the header so you can always see what's happening.

## Local development

You can run this locally with the [Netlify CLI](https://docs.netlify.com/cli/get-started/):

```bash
npm install -g netlify-cli
netlify login
netlify link        # connect to your Netlify site
netlify dev         # serves the page at :8888 with functions wired up
```

Or just open `index.html` directly in a browser — the function call will fail,
the page goes "offline", and you'll edit a localStorage copy. Useful for trying
out UI changes without burning commits.

## Editing the design

- **Visual tweaks:** edit `library.css` (everything is prefixed `.sh-*` for the Shelf direction).
- **Behaviour tweaks:** edit `library-shelf.jsx`.
- **Add fields per book** (author, ISBN, etc.): update the data shape in `library-data.jsx`'s `useRemoteLibrary` and add the form fields in `library-shelf.jsx`.

## The other two directions

`directions.html` still works as a design-canvas reference if you ever want
to revisit the original Stickerbook (A) or Ledger (C) directions. It uses
localStorage only — not hooked to the backend. Open it locally; it's not
deployed by default unless you link to it.
