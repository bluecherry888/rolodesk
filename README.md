# 📇 Rolodesk

A clean prospecting **Rolodex + calling desk** for working through exported ZoomInfo lead lists.
Import a list → clean out duplicates → work it one contact at a time → every touch logged
automatically.

**▶ Live app: https://bluecherry888.github.io/rolodesk/**

No server, no account, no tracking — the entire app is a single self-contained HTML file.
Your contacts, notes, and call history live in **your browser's local database (IndexedDB)**
and never leave your machine. Use *Settings → Export backup (JSON)* to back up or move
between devices, and *Restore from backup* to load it elsewhere.

## Features

- Two-door home: **Upload New List** / **Call Existing List**
- Drag-and-drop CSV import with automatic ZoomInfo column mapping
- Duplicate detection (exact + fuzzy: email, LinkedIn, phone, name+company,
  company-name variations like `JD Irving` ≡ `J.D. Irving, Limited`) with per-row
  Merge / Replace / Keep / Skip review — nothing is auto-deleted
- Account + contact Rolodex with filters, quick chips, and universal search
- Calling workspace: one card at a time, big green **Call** button, contact-local clock
  with best-time-to-call indicator, outcome form, automatic timestamps
- Contact editing mid-call (wrong/dead numbers) with update notes and a field-level audit trail
- Follow-ups with `.ics` / Google Calendar export; overdue and due-today surfacing
- Resumable calling queue, complete per-contact and per-account activity timelines
- Dashboard: calls, connections, conversion rates, untouched contacts
- Backup export/restore + CSV export of the full master list

## Notes

- **CSV works everywhere.** Excel (`.xls/.xlsx`) import needs network access to load the
  Excel reader, which some hosted contexts block — export from ZoomInfo as CSV if in doubt.
- `index.html` here is the built single-file bundle. Full source (modular `assets/`,
  build script, sample data) lives in the project workspace and is pushed as the repo evolves.
- See [SPEC.md](SPEC.md) for the product specification, data model, duplicate-detection
  logic, and the phased roadmap (hosted DB + auth → dialer integration → team features).
