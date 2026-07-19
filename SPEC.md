# Rolodesk — Product Specification (v1)

A clean sales-prospecting workspace for working through exported ZoomInfo lead lists.
Part digital Rolodex, part structured calling desk. Deliberately **not** an enterprise CRM.

> This document is the concise design brief. The working prototype lives beside it
> (`index.html` + `assets/`). See `README.md` to run it.

---

## 1. Product summary

**Job to be done:** Import a ZoomInfo export → clean out duplicates → work the list
one contact at a time → log every touch automatically → always know who I called and when,
and who to call next.

**Design principles**
- Two doors on the home screen: *Upload New List* and *Call Existing List*. Nothing else competes.
- Fast during a live call. One primary contact card, a big green **Call** button, keyboard-friendly.
- Never lose history. Merges preserve notes, calls, and follow-ups.
- Time is recorded automatically — the user never types a call time.
- Persistent local database, not throwaway browser storage.

---

## 2. Recommended technology stack

Kept intentionally small for v1, with a clear upgrade path.

| Concern | v1 (prototype, in this repo) | Production path |
|---|---|---|
| App shell | Vanilla JS SPA, no build step (runs from `file://` or any static host) | Same code → wrap in Vite + TypeScript + React when complexity warrants |
| Persistence | **IndexedDB** (durable, survives restarts, ~hundreds of MB) | Postgres (hosted: Supabase / Neon / RDS) + a thin API |
| Auth | Local profile + optional passphrase-gate | Supabase Auth / Auth0 (email + SSO), row-level security |
| CSV/Excel import | Built-in RFC-4180 CSV parser; SheetJS (lazy-loaded) for `.xls/.xlsx` | Same, plus server-side import service for large files |
| Time zones | Browser `Intl` API + IANA zones + state/province inference table | Same; add a maintained tz-lookup service |
| Calendar | Built-in calendar + `.ics` download + Google Calendar deep-link | Google/Microsoft OAuth two-way sync |
| Calling | Copy number + `tel:` launch | Twilio / Aircall / RingCentral / Dialpad click-to-call |
| Search/filter | In-memory indexes over IndexedDB | Postgres GIN indexes / trigram search |

**Why this shape:** ZoomInfo exports are CSV/XLSX and lists are on the order of hundreds–thousands of
rows — well within what a browser + IndexedDB handles instantly. Starting server-free means the tool is
usable *today* with zero setup, while every choice (schema, dedupe, import) ports straight to a hosted DB.

---

## 3. Database structure

Object stores (IndexedDB in v1; 1:1 with SQL tables in production).

- **lists** — `id, name, description, source, fileName, importedAt, columnMapping, stats{}`
- **accounts** — `id, name, nameKey (normalized), domain, website, industry, employeeCount, revenue,
  street, city, state, country, postalCode, timezone, status, priority, owner, lastOutcome,
  lastContactedAt, nextFollowUpAt, sourceListIds[], createdAt, updatedAt`
- **contacts** — `id, accountId, firstName, lastName, fullName, nameKey, title, department, jobFunction,
  seniority, directPhone, mobilePhone, companyPhone, phoneKey, email, emailKey, linkedin, linkedinKey,
  timezone, status, priority, interest, lastOutcome, lastContactedAt, nextFollowUpAt, callCount,
  doNotContact, originalListId, sourceListIds[], createdAt, updatedAt`
- **activities** — immutable log — `id, contactId, accountId, type, at (ISO), actor,
  tzUser, tzContact, contactLocalTime, outcome, notes, meta{}`
  Types: `import, contact_created, contact_updated, call, note, email, linkedin, referral, meeting,
  followup_created, followup_changed, status_change, merge, session_complete`.
- **followups** — `id, contactId, accountId, dueAt, timezone, reminder, reason, plannedAction, status,
  calendarLinked, createdAt, updatedAt`
- **sessions** — `id, name, filterSnapshot{}, queue[contactId], cursor, status, createdAt, updatedAt`
  (this is what powers *resume where I left off*).
- **settings** — `preferredCallWindow{start,end}, userTimezone, theme, ...`

Indexes: contacts by `accountId / emailKey / phoneKey / linkedinKey`; accounts by `domain / nameKey`;
activities by `contactId / accountId / at`; followups by `dueAt / status`.

`activities` is append-only — the audit trail. Records are archived (soft-deleted), never hard-deleted.

---

## 4. Duplicate-detection logic

Runs at import against (a) other rows in the same file and (b) everything already stored.

**Normalization**
- *Company key:* lowercase → strip punctuation → drop suffixes (`inc, incorporated, llc, ltd, limited,
  corp, corporation, co, company, plc, gmbh, sa, group, holdings, the`) → collapse spaces.
  So `JD Irving`, `J.D. Irving`, `J.D. Irving, Limited`, `Irving Limited` → same key family.
- *Domain:* derived from website/email, `www.` stripped, lowercased.
- *Phone key:* digits only, last 10 kept. *Email key:* trimmed + lowercased. *LinkedIn key:* path only.

**Account match:** exact domain → same; else exact company key → same; else fuzzy company key
(token overlap + edit-distance ratio ≥ 0.82) → *possible*. Prevents duplicate accounts while still
allowing many legitimate contacts under one company.

**Contact classification**
- **Exact duplicate:** same email key, OR same LinkedIn key, OR same phone key, OR (same name key **and**
  same account).
- **Possible duplicate:** similar name at the same account, same name with a changed title, a
  company-name variation, or the same person appearing in a newer export.

**Resolution (never auto-deletes anything uncertain).** Per record: **Keep existing · Replace ·
Merge · Import as separate · Skip**. Exact and Possible are shown in separate review groups with a
one-click bulk default.

**Merge rules:** union of source lists; keep the most recent non-empty field values; **preserve all
notes, call history, and follow-ups**; write a `merge` activity with the timestamp.

---

## 5. Calendar & time-sync approach

**Automatic timestamps (never typed).** Every upload, import, contact create/update, call attempt,
saved outcome, note, follow-up, status change, merge, and completed session is stamped with an exact
ISO timestamp and the actor. The UI shows both the exact stamp *and* a friendly form
("Called July 14, 2026 at 9:42 a.m." · "Last contacted 3 days ago").

**Per call, captured automatically:** call date, start time, end time/duration (when placed in-app),
your time zone, the contact's time zone, and the contact's local time at dial.

**Follow-ups → calendar.** Scheduling a follow-up produces a calendar event titled
`Follow up with {Name} at {Company}`, description pre-filled with role, phone, email, LinkedIn, previous
outcome, latest notes, and the planned next action. v1 offers a built-in calendar view, an **`.ics`
download**, and a **Google Calendar deep-link**; production adds Google/Microsoft OAuth for true
two-way sync (change in app ↔ change on calendar) with a stored event id to avoid duplicate events.

**Time-zone awareness.** Contact local time shown live on the card with a calling-window indicator:
`Outside hours · Early · Good time to call · Business hours · Lunch · Late`. Preferred window defaults to
**8:00–9:00 a.m. contact-local** (editable). The queue can prioritize contacts currently in-window;
calling outside it warns but never blocks.

---

## 6. Primary screens

1. **Home** — two large cards (*New: Upload New List*, *Call Existing List*) + a compact dashboard strip.
2. **Upload** — drag-drop / browse → preview stats → column mapping → duplicate review → name & save →
   *Start calling* or *Home*.
3. **Rolodex** — filter bar + search; toggle **Accounts ⇄ Contacts**; open an account to see its people.
4. **Calling workspace** — one contact card, green **Call**, action row, outcome form, follow-up scheduler.
5. **Activity / Previous Notes** — pinned next step + chronological typed timeline with exact stamps.
6. **Dashboard** — funnel + activity stats, filterable by date range and list.

---

## 7. User journey

```
Home ─▶ Upload New List ─▶ drop file ─▶ preview ─▶ map columns ─▶ review duplicates
     └─▶ Call Existing List ─▶ Rolodex ─▶ filter/search ─▶ Start calling session
                                                              │
   ┌──────────────────────────────────────────────────────────┘
   ▼
Calling card ─▶ Call (auto-stamp) ─▶ Outcome form ─▶ Note / Schedule follow-up ─▶ Next contact
   │                                                                                    │
   └── Skip / Pause / View Account / View Previous Notes / Mark Complete ───────────────┘
Session state saved continuously ─▶ later: "Resume where I left off"
```

---

## 8. MVP feature list (this prototype targets all of these)

Home (two doors) · drag-drop upload · CSV + Excel import · column mapping · duplicate detection ·
duplicate review & merge · persistent DB · account + contact Rolodex · filters + search · contact
calling cards · green Call button · Previous Notes · call-outcome form · automatic timestamps · contact
time-zone awareness · follow-up scheduling · calendar sync (ics/deep-link/built-in) · full account +
contact activity history · calling queue · resume where I left off · confirmations before delete / DNC.

---

## 9. Phased build plan

- **Phase 1 — Core loop (this prototype):** import → dedupe → Rolodex → calling workspace → outcomes →
  history → follow-ups → queue → resume, all on a durable local DB.
- **Phase 2 — Accounts & hosting:** move to Postgres + hosted API + real auth; Google/Microsoft OAuth
  calendar two-way sync; automatic backups + restore.
- **Phase 3 — Dialer & email:** Twilio/Aircall click-to-call with real durations; email logging.
- **Phase 4 — Team:** multi-user, owners, shared Rolodex, permissions, reporting.

---

## 10. Assumptions made (v1)

- Single user, local-first. "Secure sign-in / access controls / encryption / backups" are represented
  by an optional passphrase gate, JSON/CSV export (backup), archive-instead-of-delete, and confirm
  dialogs; full auth/encryption is Phase 2 (documented, not faked).
- Calendar sync in v1 = built-in calendar + `.ics` + Google deep-link (no live OAuth in a server-free build).
- Time-zone inference uses ZoomInfo's tz column when present, else a US/Canada state/province table,
  else country default; unknown zones fall back to the user's zone with a visible "assumed" flag.
- "Owner" and team features are stored but single-user in v1.
