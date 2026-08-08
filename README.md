# Iris42 Shift Bid — Project README

A Google Apps Script web app (bound to Google Sheets) that runs a shift‑bidding
process for a police workgroup. Officers submit a ranked bid during a bidding
window; sergeants bid one at a time in strict seniority order via a
notification chain. An admin console manages the whole cycle. Notifications can
be sent by **email** or by **SMS (Twilio)**, switchable with a toggle.

- **Owner account:** `iris42hb@gmail.com` (owns the Apps Script project; the web app runs *as* this account)
- **Deployment:** Execute as **Me** (owner) · Access **Anyone** (no Google sign‑in required)
- **Brand for SMS/Twilio:** "Iris42 Shift Bid" (a personal tool, intentionally unaffiliated with any agency)
- **Email branding:** currently sends with the display name "HBPD Shift Bid" and reply‑to `SWhite@hbpd.org`

---

## 1. How it works

**Roles**
- **Officers** — rank available shifts (drag to order). Multiple submissions allowed (latest wins).
- **Sergeants** — pick a single shift. One submission each, enforced, and only in seniority order.
- **Admin** — starts/stops cycles, sets the window, manages options and notifications.

**Sergeant chain flow**
1. Admin taps **Start Sergeant Bid**.
2. The most senior sergeant is notified (text or email) with a link to the sergeant form.
3. That sergeant opens the form, picks a shift, submits.
4. On submit: the schedule updates, the taken shift is removed from the options, the sergeant gets a receipt, the admin gets a notification, and the **next** sergeant is notified automatically.
5. Repeats down the seniority list. When the last sergeant bids, the chain **auto‑closes**.
- Out‑of‑order submissions are rejected ("It is not your turn to bid yet"). One sergeant per shift; a selected shift disappears from later sergeants' forms.

**Two independent schedules**
- **Officer** form is open only during the **bidding window** (`BID_OPEN`/`BID_CLOSE`).
- **Sergeant** form is open whenever the **chain is active** (from Start until the last bid), regardless of the officer window.

---

## 2. Web app routes (`?page=`)

All served from the deployment's `/exec` URL.

| URL | Page | Gated by |
|---|---|---|
| `/exec` (or `?page=home`) | Landing page (Submit Bid Sheet / View Assignments buttons) | Always available |
| `?page=officer` | Officer bid form | Bidding window (`isFormOpen_`) |
| `?page=sergeant` | Sergeant bid form | Sergeant chain active (`SGT_CHAIN_ACTIVE`) |
| `?page=assignments` | Public live assignments board | Always available |
| `?page=admin` | Admin console | Passcode + optional TOTP 2FA |
| `?page=privacy` | SMS privacy policy | Always available |
| `?page=terms` | SMS terms & conditions | Always available |

---

## 3. Google Sheets structure

Two spreadsheets are referenced by ID in `Code.gs`:

- **`DATA_ID`** = `1WmSa00J9AznvPS549V3fZvAbASH4viNgvg7Zr_-KXN8` — the main data workbook.
- **`ROSTER_ID`** = `18IitXUbnBDrsFJRJu4nblU-I682npDNNUlOOSDnRPL4`, tab `Next Seniority` — largely vestigial.

### Tabs in `DATA_ID`
- **Data** — append‑only log of every submission (`Timestamp | Name | PIN | Email | Entries`). Used to detect who has submitted.
- **Officer Bids** — officer roster + bids.
- **Sergeant Bids** — sergeant roster + bids.
- **Bid Options** — one row per shift (`Label | Available | Slots | Sergeant`). Controls which shifts exist, how many officer slots, and whether a shift has a sergeant position.
- **Deployment Schedule** — the visual schedule grid the app paints assignments onto.

### Officer Bids columns (identify a real person by a comma in the Name)
| Col | Meaning |
|---|---|
| B | Name (`LAST, FIRST`) |
| C | PIN |
| E | Email |
| F–V | Ranked choices (17 slots) |
| W | Awarded shift ("ASSIGNED") |
| X | Confirmation‑sent timestamp |

### Sergeant Bids columns
| Col | Meaning |
|---|---|
| B | Name (`LAST, FIRST`) — seniority = **row order**, top = most senior |
| C | PIN |
| E | Email (used for email notifications & receipts) |
| F | Assigned shift (their single selection) |
| G | "Notified" timestamp (auto‑written when notified) |
| H | Optional per‑sergeant contact override (email in email mode / phone in SMS mode) |
| I | Confirmation‑sent timestamp |
| **J** | **Mobile phone number** (for SMS notifications) |

Phone format is flexible — `7145550123`, `714-555-0123`, or `+17145550123` all
normalize to E.164 (US `+1` assumed).

---

## 4. Files

In the Apps Script editor, HTML files are named **without** an extension.

| Editor file | This repo | Purpose |
|---|---|---|
| `Code.gs` | `Code.gs` | All server logic (routing, bids, notifications, admin, auth) |
| `index` | `index.html` | Officer bid form (drag‑rank via SortableJS) |
| `sergeant` | `sergeant.html` | Sergeant bid form (single select) |
| `admin` | `admin.html` | Admin console |
| `assignments` | `assignments.html` | Public live assignments board |
| `home` | `home.html` | Landing page |
| `privacy` | `privacy.html` | SMS privacy policy page |
| `terms` | `terms.html` | SMS terms & conditions page |
| `appsscript.json` | `appsscript.json` | Manifest (OAuth scopes + webapp settings) |

Not in Apps Script, used for Twilio A2P registration:
- `privacy.md`, `terms.md` — the **hosted** policies (GitHub Pages: `https://swhite005.github.io/iris42-shift-bid/privacy` and `/terms`).
- `Iris42-SMS-Consent-Form.pdf` / `.png` — the printable in‑person opt‑in consent form.

---

## 5. Notifications

### Mode toggle
A single Script Property, **`NOTIFY_MODE`**, controls delivery:
- `email` (or unset) → sergeants are emailed their bid link.
- `sms` → sergeants are texted via Twilio.

Switch it from the admin console (Sergeant Bids & Notifications → **Notify sergeants by: Email / Text**), which appears highlighted for the current mode and updates instantly. All SMS/phone code stays in place either way.

### When things fire
Under Execute‑as‑Me, everything happens **inline the moment a sergeant submits**:
- the next sergeant is notified,
- the submitting sergeant gets an email **receipt** (with their selection; also stamps the confirmed column so it isn't duplicated),
- the **designated admin** gets an email with the sergeant's name + selection.

### The background job (5‑minute trigger)
An installable time trigger (`sergeantChainTick_`) is a **backstop** plus the driver for **automatic opening/closing announcement emails**. Because normal actions run inline, the background job is *optional* for day‑to‑day operation — needed only if you use the automatic window announcements. Installed/removed via **Set up / Remove background job** in admin.

### Confirmations & admin alert
- **Sergeant receipt:** sent to the sergeant's email (col E). Always email.
- **Admin alert:** sent to `ADMIN_NOTIFY_EMAIL` if set, otherwise to `SWhite@hbpd.org` (`NOTIFY_REPLY_TO`).

### Announcements (officers)
Optional bulk emails to **all officers** (BCC, addresses hidden), sent automatically when the window opens/closes, or manually. Configured in admin under **Bidding Window & Announcements**. `{LINK}` becomes a "Submit Bid Sheet" button; `{ASSIGNMENTS}` becomes a "View Assignments" button. Sergeants are **excluded** from these.

### SMS (Twilio) specifics
- Sent via `UrlFetchApp` to the Twilio REST API using `TWILIO_SID` / `TWILIO_TOKEN` / `TWILIO_FROM`.
- Message text: `Iris42 Shift Bid: Sgt. LAST, you are up to submit your shift bid... Reply STOP to opt out, HELP for help.`
- A **Send test text** box (Text mode) lets an admin verify delivery to their own number without starting a chain.
- Requires a valid Twilio number and an **approved A2P 10DLC campaign** (see §9).

---

## 6. Admin console

Reached at `?page=admin`. One accordion open at a time.

- **Unlock:** passcode + (if enabled) authenticator code.
- **View Assignments Page** — opens the public board.
- **Bid Options & Positions** — per‑shift officer slot counts, whether a shift has a sergeant, open/close for bidding.
- **Bidding Window & Announcements** — set officer window open/close times; edit and send the officer opening/closing announcement emails (collapsible editor; auto‑send toggle).
- **Officer Assignments** — live view of officer awards.
- **Sergeant Bids & Notifications** — Email/Text toggle; per‑sergeant list showing each sergeant's on‑file contact and status; per‑sergeant override; web‑app URL field; **Start Sergeant Bid**, **Pause auto‑advance**, **Clear notification history**, **Set up / Remove background job**; test‑text box (Text mode).
- **End & Reset Bid Cycle** — archives the Data tab, clears bids/assignments/notified/confirmed columns, closes the window, pauses the chain. Keeps Bid Options, overrides, the web‑app URL, and the background job. Typed "RESET" confirmation.

**Clear notification history** just erases the "who was already notified" marks (col G) and pauses auto‑advance — used only to restart a chain from the top. It never touches bids or assignments, and End & Reset already does it.

---

## 7. Security / admin authentication

Two factors, with a server‑issued session token so the passcode alone can't drive actions:

1. **Passcode** — Script Property `ADMIN_KEY` (something you know).
2. **TOTP authenticator code** — if `TOTP_SECRET` is set (something you have). Standard RFC 6238 (Google Authenticator, Authy, 1Password, etc.).

On successful login the server mints a short‑lived **session token** (3 hours) held only in the page; every admin action requires that token. Login locks out for 10 minutes after 5 failed attempts.

**Turn 2FA on:** run **`setupTotp`** from the editor once (as the owner), read the setup key from the execution log, add it to your authenticator app. **Turn it off:** run **`disableTotp`**. Until a secret is set, the page works with the passcode alone (smooth rollout).

---

## 8. Script Properties reference

Set under **Apps Script → Project Settings → Script Properties**. Property changes take effect immediately (no redeploy needed).

**You set these manually:**
| Property | Purpose |
|---|---|
| `ADMIN_KEY` | Admin passcode (required to use admin) |
| `NOTIFY_MODE` | `email` or `sms` (also set by the admin toggle) |
| `TWILIO_SID` | Twilio Account SID (starts `AC…`) — SMS mode |
| `TWILIO_TOKEN` | Twilio Auth Token — SMS mode |
| `TWILIO_FROM` | Twilio sending number, E.164 (e.g. `+17145550123`) — SMS mode |
| `ADMIN_NOTIFY_EMAIL` | Where per‑bid admin alerts go (defaults to `SWhite@hbpd.org`) |
| `WEBAPP_URL` | Optional manual override of the `/exec` URL used in links |

**Managed by the app (don't normally edit by hand):**
`TOTP_SECRET`, `ADMIN_SESSIONS`, `ADMIN_LOGIN_FAILS`, `ADMIN_LOGIN_LOCK_UNTIL`,
`BID_OPEN`, `BID_CLOSE`, `SGT_CHAIN_ACTIVE`, `SGT_TRIGGER_OWNER`, `SGT_TRIGGER_AT`,
`ANN_AUTO`, `ANN_START_SUBJECT`, `ANN_START_BODY`, `ANN_END_SUBJECT`, `ANN_END_BODY`,
`ANN_START_SENT`, `ANN_END_SENT`.

---

## 9. Setup & deployment

### First‑time / after any code change
1. Paste updated file(s) into the editor (HTML files named with **no** extension).
2. **Deploy → Manage deployments → edit (pencil) → Version: New version → Deploy.**
   - The live `/exec` URL always serves the **last deployed version** — saving alone is not enough. This is the #1 source of "it didn't update."
3. If the admin page looks stale on mobile, fully close the tab and reopen, or append a throwaway param (`&v=2`).

### Deployment settings
- **Execute as:** Me (owner) → all server ops + emails run as `iris42hb`.
- **Who has access:** Anyone → forms/board reachable without a Google login (PIN still gates bidding; passcode+2FA gate admin).

### Authorization / OAuth scopes
`appsscript.json` pins the required scopes so authorization is predictable:
- `spreadsheets` (Sheets), `script.send_mail` (email), `script.scriptapp` (triggers),
  `script.external_request` (**Twilio via UrlFetchApp**), `userinfo.email`.

If Twilio calls fail with *"You do not have permission to call UrlFetchApp.fetch"*, the `script.external_request` grant is missing: ensure the manifest includes it, then re‑authorize (run any function from the editor and approve, or remove the app at **myaccount.google.com/permissions** and run again), and redeploy a new version.

---

## 10. Twilio / A2P 10DLC (SMS mode)

Required only if you use SMS notifications.

1. **Credentials** → Script Properties `TWILIO_SID`, `TWILIO_TOKEN`, `TWILIO_FROM`.
   - SID + Auth Token: Console dashboard → Account Info.
   - Number: Phone Numbers → Manage → Active numbers (buy an SMS‑capable number; a paid number is required for real use).
2. **Brand:** registered as an individual **Sole Proprietor** ("Iris42", contact on the brand). Sole‑prop limits: low daily volume, one 10‑digit number.
3. **A2P 10DLC campaign** must be **Approved** before carriers deliver messages. Registration fields used:
   - **Description / samples:** personal, low‑volume, internal shift‑bid notifications; samples say "Iris42 Shift Bid: …".
   - **Opt‑in:** in‑person only (no website/keyword). Proof = the signed consent form (`Iris42-SMS-Consent-Form`).
   - **Privacy Policy URL / Terms URL:** the GitHub Pages links (must be publicly readable — the Apps Script `?page=privacy` URL is *not* crawlable for this purpose, so the GitHub‑hosted `.md` versions are what's registered).
   - Opt‑in keywords `START, YES, UNSTOP`; STOP/HELP handled by Twilio.
4. **Consent form:** print `Iris42-SMS-Consent-Form.pdf`, have each participant complete/sign it (voluntary, unchecked‑by‑default checkbox), retain it.
5. Once approved, in admin flip **Notify sergeants by → Text**, and use **Send test text** to confirm.

Trial accounts can only text **verified** numbers and add a trial prefix — upgrade to send freely.

---

## 11. Policy pages

- In‑app: `?page=privacy`, `?page=terms` (branded "Iris42 Shift Bid", personal/unaffiliated, non‑sharing statement, frequency, rates, STOP/HELP).
- Registered with Twilio: the **GitHub‑hosted** `privacy.md` / `terms.md` (reviewer‑crawlable). Keep both copies' wording in sync.

---

## 12. Operational notes & gotchas

- **Redeploy a new version after every code change** — the `/exec` URL serves the deployed version, not saved edits.
- **HTML files have no extension** in the editor (`index`, `sergeant`, `admin`, `assignments`, `home`, `privacy`, `terms`).
- **Sergeant seniority = row order** on the Sergeant Bids tab (top = most senior). The chain and the out‑of‑order guard both use this.
- A sergeant with **no contact on file** is skipped by the chain — if a senior sergeant has no email/phone, populate it (col E/J or the override in col H) so the chain doesn't stall or skip.
- The in‑app browser may block popups/native dialogs; the forms use in‑page modals, and the admin uses tap‑to‑confirm patterns instead.
- Email vs SMS branding currently differs (emails: "HBPD Shift Bid"; SMS: "Iris42 Shift Bid"). Only SMS content is reviewed by carriers.

---

## 13. Editor functions (run manually from the Apps Script editor, as the owner)

| Function | What it does |
|---|---|
| `setupTotp` | Turn on 2FA — generates a secret; read the setup key from the execution log |
| `disableTotp` | Turn off 2FA |
| `authorizeAndTest` | Send a test email to confirm/authorize email sending |
| `manualRun` | Re‑run the assignment pipeline (convenience) |

The web app's own buttons handle everything else (starting the chain, notifications, resets, mode switching, test texts).
