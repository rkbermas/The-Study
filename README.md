# The Study

A calm personal planner that connects long-term goals to weekly plans and daily focus.
It's a single HTML file hosted on GitHub Pages, with your data stored privately in Supabase.

This README is the maintenance guide: how it's built, how to set it up from scratch, and
where to look when you want to change something.

---

## Contents

1. [How the app works](#1-how-the-app-works)
2. [Files in this repo](#2-files-in-this-repo)
3. [How privacy works](#3-how-privacy-works)
4. [Setting up from scratch](#4-setting-up-from-scratch)
5. [Deploying a change](#5-deploying-a-change)
6. [Testing a change before deploying](#6-testing-a-change-before-deploying)
7. [Map of index.html](#7-map-of-indexhtml)
8. [How the data is stored](#8-how-the-data-is-stored)
9. [Common edits, step by step](#9-common-edits-step-by-step)
10. [Changing the data format safely](#10-changing-the-data-format-safely)
11. [Updating the Supabase library](#11-updating-the-supabase-library)
12. [Backups and restoring](#12-backups-and-restoring)
13. [Troubleshooting](#13-troubleshooting)
14. [Asking an AI to help edit](#14-asking-an-ai-to-help-edit)

---

## 1. How the app works

The app follows a "goal cascade": big goals are broken into small steps, a few steps are planned
into each week, and up to three are picked for each day.

| Tab | What it's for |
|---|---|
| **Today** | Up to three steps to do today. Pick from this week's plan or add a new one. |
| **This week** | The steps planned for the week, grouped by goal, with a progress bar. Flags goals with nothing planned, offers to carry over last week's unfinished steps, and has an end-of-week check-in. |
| **Goals** | Strategic goals grouped by life area. Open a goal to add steps, write why it matters, set a finish-by date, or mark it achieved. |
| **Progress** | Percentage of each week's plan completed (around 85% is a strong week), progress per goal, and achieved goals. |

**The key idea:** a step is one item that can appear in several places. Ticking it off in Today
also ticks it off in This week and counts toward its goal.

---

## 2. Files in this repo

| File | Purpose |
|---|---|
| `index.html` | The entire app: layout, styling, and code in one file. GitHub Pages serves this as the home page. |
| `README.md` | This guide. |
| `supabase.sql` | *(Optional to keep)* The database setup script. Its full contents are also in [section 4](#4-setting-up-from-scratch). |

---

## 3. How privacy works

- **The code is public, the data is not.** Anyone can see `index.html` on GitHub, but it contains no personal information.
- **Your data lives in Supabase**, protected by Row Level Security (RLS). The database only returns a row to the person who owns it, and only when they're signed in.
- **The key in `index.html` is the *publishable* (anon) key.** It's designed to be public. **Never** put the *secret* key (`sb_secret_...`) or the *service_role* key in the file: those bypass all privacy rules.
- **Sign-ups are turned off** in Supabase, so nobody else can create an account.
- The page tells search engines not to index it, and a Content Security Policy blocks it from sending data anywhere except Supabase.
- Signing out clears the copy stored in that browser. The data stays in your account.

---

## 4. Setting up from scratch

Use this if you ever need to rebuild everything (new Supabase project, new repo, etc.).

### 4a. Supabase

1. Create a project at [supabase.com](https://supabase.com).
2. Open **SQL Editor** (left sidebar, `>_` icon) → **New query**, paste the SQL below, and click **Run**.

   ```sql
   create table if not exists public.planner (
     user_id    uuid primary key default auth.uid() references auth.users(id) on delete cascade,
     data       jsonb not null default '{}'::jsonb,
     updated_at timestamptz not null default now()
   );

   alter table public.planner enable row level security;
   revoke all on public.planner from anon;

   create policy "Read own planner" on public.planner
     for select to authenticated using ((select auth.uid()) = user_id);

   create policy "Create own planner" on public.planner
     for insert to authenticated with check ((select auth.uid()) = user_id);

   create policy "Update own planner" on public.planner
     for update to authenticated
     using ((select auth.uid()) = user_id)
     with check ((select auth.uid()) = user_id);
   ```

3. **Authentication → Sign In / Providers**: keep Email on, turn **off** "Allow new users to sign up".
4. **Authentication → Users → Add user → Create new user**: enter your email and password, tick **Auto Confirm User**.
5. Get your keys: click **Connect** at the top of the dashboard, or go to **Project Settings → Data API** (Project URL) and **Project Settings → API Keys** (publishable key).

### 4b. Connect the app

In `index.html`, search for `SETUP` and fill in:

```js
const SUPABASE_URL = 'https://your-project-ref.supabase.co';
const SUPABASE_KEY = 'sb_publishable_...';
```

Leave both as `''` to run the app with everything stored only in the browser (no account, no sync).

### 4c. GitHub Pages

1. Upload `index.html` to the root of the repository.
2. **Settings → Pages → Build and deployment**: Source **Deploy from a branch**, branch `main`, folder `/ (root)`.
3. Wait a minute or two, then open the URL GitHub shows.

---

## 5. Deploying a change

1. **Download a backup first** (in the app: Settings and backup → Download a backup).
2. In the GitHub repo, open `index.html` → pencil icon (Edit), or upload the new file with **Add file → Upload files**.
3. **Commit changes.**
4. Wait 1–2 minutes for Pages to rebuild (progress shows under the repo's **Actions** tab).
5. Open the app and do a hard refresh: `Ctrl+Shift+R` (Windows) or `Cmd+Shift+R` (Mac). On a phone, close and reopen the tab.

If something breaks, go to the file's **History** on GitHub, open the previous version, and restore it.

---

## 6. Testing a change before deploying

The safest way to try a change without touching your real data:

1. Make a copy of `index.html` on your computer.
2. In the copy, set `SUPABASE_URL` and `SUPABASE_KEY` to `''` (local-only mode).
3. Double-click the copy to open it in your browser and try things out.

Test data stays in that browser only. To wipe it: open the developer console (`F12` → Console) and run
`localStorage.clear()`, then reload.

To test the real sign-in and sync locally, put back your URL and key and run a tiny local server in
the folder (opening the file directly sometimes behaves differently):

```bash
python3 -m http.server 8000
# then open http://localhost:8000
```

> ⚠️ Testing with your real URL and key means you're editing your real data. Download a backup first.

---

## 7. Map of index.html

Use your editor's search (`Ctrl+F` / `Cmd+F`) to jump to these.

### The `<head>` section

| Search for | What it controls |
|---|---|
| `Content-Security-Policy` | Which outside websites the page may load from or talk to (see [9h](#9h-adding-an-outside-script-font-or-service)). |
| `rel="icon"` | The browser tab icon. |
| `apple-touch-icon` | The phone home-screen icon. |
| `fonts.googleapis.com` | The fonts (Newsreader for text, Instrument Sans for labels). |
| `supabase-js@` | The Supabase library version and its integrity hash. |

### The `<style>` section

| Search for | What it controls |
|---|---|
| `:root{` | Colors and fonts for light mode (CSS variables like `--accent`). |
| `prefers-color-scheme:dark` | Colors for dark mode. Change these alongside the light ones. |
| `.tabs` | The navigation tabs. |
| `.list li` | Rows of steps. |
| `.goal`, `.ghead`, `.gbody` | Goals list, goal headers, and the opened goal panel. |
| `.pick` | The dashed "Do today" / "Plan it" suggestion buttons. |
| `.tally`, `.bars` | Big numbers and progress bars on the Progress tab. |
| `.toast` | The small pop-up at the bottom ("Removed… Undo"). |
| `@media (max-width:520px)` | Phone-specific layout tweaks. |

### The `<script>` section, top to bottom

| Search for | What it does |
|---|---|
| `SETUP` | Supabase URL and key. |
| `/* ---------- dates` | Date helpers. Weeks use ISO week keys like `2026-W40` (weeks start Monday). |
| `const AREAS` | The list of life areas. |
| `TODAY_MAX` | Maximum steps allowed in Today (currently 3). |
| `/* ---------- data` | Data structure, migrations from older versions (`fromV0`, `fromV1`), `normalize`, `save`. |
| `/* ---------- account sync` | Supabase sync: `pushNow` (upload), `syncNow` (compare and download), and the `STATUS` messages. |
| `function vToday` | Builds the Today tab. |
| `function vWeek` | Builds the This week tab, including the end-of-week check-in questions. |
| `function goalBlock` | Builds one goal (collapsed header + opened panel). |
| `function vGoals` | Builds the Goals tab. |
| `function vProgress` | Builds the Progress tab. |
| `function vSignIn` | The sign-in screen. |
| `function render` | Redraws the current tab and keeps keyboard focus in place. |
| `/* ---------- actions` | What every button does. Each button has a `data-act="..."` name that matches a function in the `acts` list (e.g. `sToday`, `gDone`). |
| `app.addEventListener('submit'` | What each form does: `tAdd` (Today), `wAdd` (week), `sAdd` (step under a goal), `gAdd` (new goal), `signIn`. |
| `/* ---------- settings and backup` | Name field, backup download/restore, sign out. |
| `function onAuth` / `function boot` | Start-up and sign-in handling. |

**How a button works, as an example:** the "Do today" button is written as
`data-act="sToday" data-id="(step id)"`. When clicked, the code finds `sToday()` in the `acts`
list, runs it, saves, and redraws the screen.

---

## 8. How the data is stored

Everything is one JSON object, saved in two places:

- **In the browser** (`localStorage`, key `the-study:simple:v1`) so it works instantly and offline.
- **In Supabase**, in the `planner` table: one row per user, with the whole object in the `data` column.

### Structure (version 2)

```js
{
  v: 2,                    // data format version
  name: "Sam",             // name used in the greeting
  goals: [
    { id, text, area,      // area is an id from AREAS, e.g. "career"
      why, due,            // due is "YYYY-MM-DD" or ""
      done, doneDate }
  ],
  steps: [
    { id, text,
      goalId,              // id of the goal it serves, or null
      week,                // "2026-W40" when planned into a week, else ""
      day,                 // "2026-10-01" when picked for a day, else ""
      done, doneDate }
  ],
  weeks: {
    "2026-W40": { good, hard, next }   // end-of-week check-in answers
  },
  meta: { updatedAt, owner }           // last edit time (ms) and user id, used by sync
}
```

**Where a step shows up:**
- Under its goal: whenever `goalId` matches.
- In This week: when `week` equals that week.
- In Today: when `day` equals today's date.
- In "Still open from earlier" on Today: when `day` is a past date and it isn't done.

### How sync decides what's newest

Every edit sets `meta.updatedAt`. When the app opens, regains focus, or comes back online, it
compares the browser copy with the Supabase copy and keeps whichever is newer **as a whole**.
If you edit on two devices at the same moment, the later save wins and the other device's
edits from that moment are lost. Reload other devices after big edits.

You can see the stored data in Supabase under **Table Editor → planner** (click the `data` cell).

---

## 9. Common edits, step by step

### 9a. Change wording

Most text is written directly in the `vToday`, `vWeek`, `vGoals`, `vProgress` functions.
Search for the exact words you see on screen and edit them. Keep the surrounding quotes and
backticks (`` ` ``) intact.

### 9b. Change the life areas

Search `const AREAS`:

```js
const AREAS=[
  ['presence','Inner life'],['career','Work'],['education','Education'],['health','Health'],
  ['money','Money'],['intellect','Reading & creating'],['ordinary','Relationships']
];
```

Each entry is `['id','Label']`.
- **Rename** an area: change only the label (second part). Safe.
- **Add** an area: add a new pair with a new, unique id, e.g. `['faith','Faith']`.
- **Remove or change an id**: goals still using the old id will appear under "Other". Move them to a new area first (open the goal → Details → Life area).

### 9c. Change the daily limit

Search `TODAY_MAX` and change `3`. Also update the wording that mentions "three" in `vToday`
and in the `sToday` action message ("Today already has three steps").

### 9d. Change the end-of-week check-in questions

In `vWeek`, find the lines starting with `${q(`:

```js
${q('What went well?','good','Big or small')}
```

The parts are: the question, the storage name, and the placeholder hint. You can freely change
the question and hint. If you **add** a question, give it a new storage name (e.g. `'grateful'`).
Don't reuse or rename existing storage names, or past answers will seem to disappear.

### 9e. Change colors

In the `<style>` section, edit the variables in `:root{` (light mode) **and** in both dark-mode
blocks (`prefers-color-scheme:dark` and `[data-theme="dark"]`).

| Variable | Used for |
|---|---|
| `--paper` | Page background |
| `--leaf` | Input field background |
| `--ink`, `--ink-2`, `--ink-3` | Main text, secondary text, faint text |
| `--rule` | Divider lines and empty bars |
| `--accent` | Buttons, links, checkmarks, progress bars |
| `--accent-ink` | Text on accent-colored buttons |
| `--accent-soft` | Pills ("Today") and the line beside an opened goal |
| `--warn` | Error messages |

Also update the two `theme-color` lines in `<head>` to match `--paper`.

### 9f. Change fonts

1. Pick fonts on [fonts.google.com](https://fonts.google.com) and copy the `<link href="https://fonts.googleapis.com/css2?...">` it gives you.
2. Replace the existing font `<link>` in `<head>`.
3. In `:root{`, change the first name in `--serif` (body text) and/or `--sans` (labels), keeping the fallbacks after it.

### 9g. Change the icon

The icon is embedded in `<head>` (search `rel="icon"`). The simplest way to change it:

1. Put your icon file in the repo, e.g. `icon.svg` or `icon.png`.
2. Replace the whole `<link rel="icon" ...>` line with:
   `<link rel="icon" href="icon.svg">`
3. For the phone home screen, add a 180×180 PNG to the repo and replace the `apple-touch-icon` line with:
   `<link rel="apple-touch-icon" href="icon-180.png">`

### 9h. Adding an outside script, font, or service

The Content Security Policy blocks everything not on its list, which protects your data.
If you add something from a new website and it silently doesn't work, open the console (`F12`):
an error mentioning "Content Security Policy" means you need to add that site to the right part of
the `Content-Security-Policy` line:

| Adding a… | Add the site to |
|---|---|
| Script | `script-src` |
| Stylesheet | `style-src` |
| Font file | `font-src` |
| Image | `img-src` |
| Data/API service | `connect-src` |

Only add sites you trust: anything in `script-src` can read your planner.

### 9i. Add a new field to goals (example: "Measure of success")

1. **Show and edit it** in `goalBlock`: copy the "Why it matters" `<label>` block and change the label, placeholder, and `data-gbind="why"` to `data-gbind="measure"`.
2. **Let it save while typing**: in `app.addEventListener('input'`, the line with `t.dataset.gbind==='why'` only lets `text` and `why` save on each keystroke. Add `||t.dataset.gbind==='measure'` in both places on that line. (Fields like dates and dropdowns save on change automatically.)
3. **Optional:** show it in the collapsed goal header the same way `gwhy` is shown.

Old goals simply won't have the field yet, which is fine (it shows as empty).
No Supabase change is needed for new fields.

### 9j. Add a new action button

1. Write the button in a view: `btn('myAction', s.id, 'Label')` (or plain HTML with `data-act="myAction" data-id="..."`).
2. Add a function with the same name to the `acts` list in the click handler:
   ```js
   myAction(){ if(s){ /* change s here */ } },
   ```
   Inside it, `s` is the step and `g` is the goal matching `data-id`. The app saves and redraws
   automatically afterwards. Return `'view'` instead if the action only changes what's displayed,
   not the data.

---

## 10. Changing the data format safely

Adding a new field (like 9i) is safe. **Renaming or restructuring** existing fields needs care,
because old data in browsers and in Supabase still uses the old shape.

1. **Download a backup.**
2. Change `fresh()` to the new shape and bump the version (`v:3`).
3. In `normalize`, add a conversion for `v===2` data into the new shape (see how `fromV1` converts version 1).
4. In `syncNow`, the check `data.data.v!==2` forces an upload of upgraded data. Update it to the new version number.
5. Test locally with a restored backup (section 6) before deploying.

The Supabase table never needs changing for this: it stores whatever object the app gives it.

---

## 11. Updating the Supabase library

The library is pinned to an exact version with an integrity hash, so a tampered copy can't run.
Updating is optional; the current version will keep working.

1. Find the latest version on [npmjs.com/package/@supabase/supabase-js](https://www.npmjs.com/package/@supabase/supabase-js) (stay on `2.x`; version 3 may need code changes).
2. Get the new hash. Either:
   - open `https://www.jsdelivr.com/package/npm/@supabase/supabase-js`, select the version, find `dist/umd/supabase.js`, and copy its SRI hash; or
   - run in a terminal (replace `VERSION`):
     ```bash
     curl -sL https://cdn.jsdelivr.net/npm/@supabase/supabase-js@VERSION/dist/umd/supabase.js \
       | openssl dgst -sha384 -binary | openssl base64 -A
     ```
     and put `sha384-` in front of the result.
3. In `index.html`, update the version in the `src` URL **and** the `integrity="sha384-..."` value.

If the version and hash don't match, the library won't load and the sign-in page will say it
"couldn't load the sign-in service".

---

## 12. Backups and restoring

- **Download:** Settings and backup (bottom of any tab) → **Download a backup**. Saves a `.json` file.
- **Restore:** Settings and backup → **Restore from a backup** → choose the file. This replaces everything and syncs to your account.
- Backups from older versions of the app are converted automatically when restored.

**Recovering data from an old copy of the app** (one that was opened from a different address):
open that old copy, press `F12` → Console, run
`copy(localStorage.getItem('the-study:simple:v1'))`, paste into a text file, save it as
`backup.json`, then restore it in the current app.

Download a backup before any edit to `index.html`, and occasionally anyway.

---

## 13. Troubleshooting

### Messages at the bottom of the app

| Message | Meaning | What to do |
|---|---|---|
| Saved to your account. | All good. | Nothing. |
| Saving… | An upload is in progress. | Wait a second. |
| Saved on this device only. | `SUPABASE_URL` or `SUPABASE_KEY` is blank. | Fill in the SETUP section. |
| Offline… | No internet. | Changes upload when you reconnect. |
| Couldn't reach your account… | Supabase didn't respond, or the project is paused. | Check the Supabase dashboard. Free projects pause after a period of inactivity; restore it from the dashboard. |
| …the planner table isn't set up yet. | The table is missing. | Run the SQL in section 4a. |

### Other problems

**Blank page or "Opening…" forever.** Press `F12` → Console and read the red error.
Most likely a typo from a recent edit (a missing quote, backtick, or bracket). Restore the
previous version from GitHub history, then redo the edit carefully.

**"That email and password don't match an account."** Check for typos. To reset the password,
go to Supabase → Authentication → Users, open your user, and send a password recovery email.
Don't delete and re-create the user: your data is tied to the user's id, so deleting the user
**deletes your data**.

**"Couldn't load the sign-in service."** The Supabase library didn't load. Check your connection;
if you recently changed the version, check the integrity hash (section 11).

**Changes don't appear after deploying.** Wait two minutes, check the **Actions** tab on GitHub
finished, then hard refresh (`Ctrl+Shift+R` / `Cmd+Shift+R`).

**Data on my phone and laptop doesn't match.** Switch away from the app and back, or reload.
The newer copy wins (see section 8).

**Something from an outside site doesn't load.** See section 9h.

---

## 14. Asking an AI to help edit

To get useful help with changes later, give the assistant:

1. This README (it explains the structure).
2. The full current `index.html`.
3. A clear description of what you want, e.g. *"Add a 'Measure of success' field to goals and show it in the collapsed goal header."*

Ask it to keep the data format compatible (or follow section 10 if it can't), to keep the
Content Security Policy and integrity hash in place, and to never include a secret or
service_role key. Download a backup before deploying whatever it gives you.
