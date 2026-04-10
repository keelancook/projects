# Project Dashboard

A personal project tracking web app — vibe-coded to fit exactly how I think about and manage work. No subscription, no bloat, no features I'll never use. Just a clean dashboard that lives at a subdomain, syncs across all my devices, and stays out of the way.

---

## Background

This started as a single HTML file I could open on my phone. It grew into a fully synced, cross-device web app as I kept adding features I actually needed — priority tiers, Obsidian-compatible markdown in project descriptions, a kanban layout on desktop, drag-and-drop list reordering, interactive checkboxes, and real-time sync through Supabase. The whole thing is still one `index.html` file deployed to GitHub Pages.

It was built through iterative conversations with Claude (Anthropic), tweaking and extending it session by session. The design decisions, feature set, and workflow are personal and opinionated — this isn't meant to be a general-purpose tool, it's tailored to how I actually work.

---

## Live URL

Hosted at: `projects.keelancook.com`  
Access is gated behind Cloudflare Zero Trust (email-based one-time PIN login).

---

## Features

### Project Management
- Add, edit, and delete projects with a name, description, color, priority, start date, and deadline
- Five priority tiers: **Critical**, **High**, **Medium**, **Low**, and **Someday** — each with distinct color coding
- Progress tracking via a slider or tap-anywhere progress bar, displayed on every card
- Archive completed projects; restore or permanently delete from archive view
- Overdue projects are highlighted with a warm red card tint

### Lists (Tabs)
- Projects are organized into **Lists** — a default General list plus unlimited custom lists
- On **mobile**: lists appear as a horizontal tab bar at the top; drag tabs to reorder them
- On **desktop**: all lists render simultaneously as a side-by-side kanban-style column layout with horizontal scroll
- Each list has its own color dot, project count, and independent sort preference that persists across sessions
- Drag column headers to reorder lists on desktop

### Sorting
Each list supports six independent sort modes, remembered per list:
- **Default** — manual insertion order
- **Priority** — Critical → Someday
- **Due ↑ / Due ↓** — by deadline, ascending or descending
- **Start ↑ / Start ↓** — by start date, ascending or descending

### Markdown Descriptions
Project descriptions support a rich Obsidian-compatible markdown subset:

| Syntax | Result |
|--------|--------|
| `**bold**` | Bold text |
| `*italic*` | Italic text |
| `==highlight==` | Yellow highlight |
| `~~strikethrough~~` | Strikethrough |
| `` `code` `` | Inline code |
| `# Heading` / `## H2` / `### H3` | Headings |
| `- item` / `* item` | Bullet list |
| `1. item` | Numbered list |
| `> blockquote` | Blockquote |
| `---` | Horizontal divider |
| `[text](url)` | Hyperlink |
| `> [!note]` / `> [!tip]` / `> [!warning]` | Obsidian-style callout blocks |
| `- [ ] task` / `- [x] done` / `- [/] in progress` | Interactive checkboxes |

Callout blocks follow Obsidian syntax exactly — the opener line sets the type, and subsequent `>` lines before a blank line are included in the block.

### Interactive Checkboxes
Checkboxes in descriptions are **clickable directly from the card view** without opening the edit form. Each click cycles through three states:

- `- [ ]` → **Unchecked** (empty box)
- `- [x]` → **Complete** (green ✓, strikethrough label)
- `- [/]` → **In Progress** (amber ◑, amber label)

State changes save to Supabase immediately.

### Markdown Toolbar
A toolbar sits directly below the description textarea in the edit form. All buttons are uniform SVG icons (no text labels) grouped by function:

- **Checkbox** — inserts `- [ ] ` at cursor
- **Bold, Italic, Highlight, Strikethrough** — wraps selected text or inserts placeholder
- **Bullet list, Numbered list**
- **Note, Tip, Warning callouts** — inserts full Obsidian callout syntax
- **Heading, Divider, Inline code**

### Sync & Storage
- **Supabase** (PostgreSQL) is the single source of truth — all devices read from and write to the same row
- **IndexedDB** and **localStorage** serve as offline caches; they are read-only fallbacks and never overwrite Supabase automatically
- A **timestamp guard** compares local change time against the remote `updated_at` before every Supabase write — if another device saved more recently, a conflict banner appears with the choice to **Pull Latest** or **Keep Mine**
- A **↻ Sync** button in the header force-pulls the latest state from Supabase
- An **offline banner** appears if Supabase is unreachable on load, indicating cached data is being shown
- A **● Synced / ⚠ Offline** status badge in the header shows current sync state at all times

### Backup & Restore
- Export the full app state as a `.json` file at any time
- Import a backup file to restore or migrate data
- Useful for manual cross-device transfers or safekeeping in cloud storage

### Responsive Layout
- **Mobile (< 900px)**: tab bar navigation, bottom FAB for adding projects, slide-up sheets for forms
- **Desktop (≥ 900px)**: kanban column layout with all lists visible simultaneously, horizontal scroll if columns exceed viewport width, drag-to-reorder columns

### Visual Design
- Warm paper-toned color palette with dark ink typography
- *DM Serif Display* for headings and card titles, *DM Mono* for UI text
- Priority badges color-coded per tier
- Start date displays as future tense ("Starts") or past tense ("Started") based on the current date
- Overdue cards receive a red background tint scaled to urgency

---

## How It Works

### Architecture

```
GitHub Pages (index.html)
      ↓
Cloudflare Access (Zero Trust login gate)
      ↓
Browser (app logic, rendering, local cache)
      ↓
Supabase (PostgreSQL — single source of truth)
```

The entire app is a **single self-contained HTML file** with no build step, no framework, no npm. HTML, CSS, and JavaScript all live in `index.html`. External dependencies are limited to two Google Fonts loaded via CDN.

### Storage Schema

One table in Supabase:

```sql
create table projects_data (
  id         text primary key,
  data       jsonb not null,
  updated_at timestamp with time zone default now()
);
```

One row with `id = 'state'` stores the entire application state as a JSON blob. This is intentionally simple — it's a single-user app and the data volume is tiny (a few KB even with many projects).

### State Shape

```json
{
  "activeTab": "general",
  "tabs": [
    { "id": "general", "name": "General", "color": "blue" }
  ],
  "projects": [
    {
      "id": "abc123",
      "name": "Project Name",
      "desc": "Markdown description",
      "progress": 40,
      "priority": 2,
      "color": "teal",
      "start": "2026-01-15",
      "deadline": "2026-06-01",
      "tabId": "general",
      "archived": false,
      "created": 1700000000000
    }
  ],
  "tabSortModes": {
    "general": "priority"
  }
}
```

### Sync Strategy

1. On load: fetch from Supabase → update local caches (IndexedDB + localStorage) → render
2. On change: update local caches immediately (instant UI) → debounce 1200ms → fetch remote `updated_at` → compare to local change timestamp → write to Supabase if local is newer, else show conflict banner
3. On offline load: read from IndexedDB or localStorage → show offline banner → never auto-write cache back to Supabase

### Access Control

Cloudflare Zero Trust (free tier) sits in front of the GitHub Pages subdomain. It intercepts every request and requires email authentication via one-time PIN before the page loads. Approved email addresses are managed in the Cloudflare Access policy. GitHub Pages itself is public — the data is in Supabase, not the HTML file, so the publishable key being visible in source is acceptable by design (Supabase's Row Level Security handles database access).

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Hosting | GitHub Pages |
| Access control | Cloudflare Zero Trust (free tier) |
| Database | Supabase (PostgreSQL, free tier) |
| Frontend | Vanilla HTML/CSS/JavaScript — no framework |
| Fonts | DM Serif Display + DM Mono (Google Fonts) |
| Local cache | IndexedDB (primary), localStorage (fallback) |
| Auth | Cloudflare One-Time PIN (email-based) |

---

## Deployment

### Prerequisites
- GitHub account with a custom domain configured on GitHub Pages
- Supabase account (free tier)
- Cloudflare account managing the domain's DNS (free tier)

### Setup Steps

**1. Supabase**

Create a new project, then run in the SQL editor:

```sql
create table projects_data (
  id         text primary key,
  data       jsonb not null,
  updated_at timestamp with time zone default now()
);

insert into projects_data (id, data)
values (
  'state',
  '{"tabs":[{"id":"general","name":"General","color":"blue"}],"projects":[],"activeTab":"general"}'::jsonb
);

alter table projects_data enable row level security;

create policy "Allow all operations" on projects_data
  for all using (true) with check (true);

-- Trigger to auto-update updated_at on every write
create or replace function update_updated_at()
returns trigger as $$
begin
  new.updated_at = now();
  return new;
end;
$$ language plpgsql;

create trigger set_updated_at
before update on projects_data
for each row execute function update_updated_at();
```

Copy the **Project URL** and **Publishable API key** from Settings → API.

**2. index.html**

Update the two constants at the top of the `<script>` block:

```js
const SUPABASE_URL = 'https://your-project-id.supabase.co';
const SUPABASE_KEY = 'sb_publishable_your_key_here';
```

**3. GitHub Pages**

- Create a repo (can be public — data is in Supabase, not the HTML)
- Add `index.html` and a `CNAME` file containing your subdomain (e.g. `projects.yourdomain.com`)
- Enable Pages in repo Settings → Pages → deploy from main branch
- Add a CNAME DNS record in Cloudflare: `projects` → `yourusername.github.io` with proxy **off** (gray cloud)

**4. Cloudflare Zero Trust**

- Go to Cloudflare dashboard → Zero Trust
- Access → Applications → Add an Application → Self-hosted
- Set domain to `projects.yourdomain.com`
- Add a policy: Action = Allow, Selector = Emails, Value = your email address(es)
- One-time PIN is enabled by default — no additional identity provider setup needed
- Turn the DNS record proxy **on** (orange cloud) after Access is configured

---

## Local / Offline Use

The app works as a local file too. Open `index.html` directly in Brave Android (or any modern browser) and it will persist data via IndexedDB across sessions. Without Supabase credentials it won't sync, but it will work fully offline as a single-device app.

---

## Known Limitations

- **Single-user only** — the entire state is stored in one database row. Concurrent writes from multiple users would cause data loss. Multi-user support would require a proper relational schema with per-user rows and row-level security tied to authenticated sessions.
- **No real-time push** — changes on one device don't instantly appear on another. Use the ↻ button when switching devices before making changes.
- **Supabase free tier pause** — Supabase pauses inactive projects after 7 days on the free plan. Use an uptime monitor (e.g. UptimeRobot) to ping the project periodically to keep it awake.
- **Publishable key in source** — the Supabase publishable key is visible in the HTML source. This is intentional and safe by design (it has the same permission level as an anonymous public key), but Row Level Security must remain enabled.

---

## License

Personal use. Not intended for redistribution or production use by others. No warranty expressed or implied.
