# Core Dictionary — Sage_OS Standard
**Version:** 0.3
**Applies to:** All Sage_OS dictionaries

---

## SECTION 1 — DESIGN STANDARD

Every Sage_OS dictionary follows this layout and behavior exactly.
These rules cannot be overridden by individual dictionaries.

---

### 1.1 Dictionary Address Format

```
type.region.domain.subdomain.org.dictionary
```

Each segment is a proper noun or category label.
- `N` = not-for-profit entity
- `B` = business entity
- `G` = government entity
- `P` = personal dictionary

Within a dictionary, content is addressed as:
```
dictionary.category.note.detail.fragment
```

Maximum 10 items at each level → maximum 10,000 fragments per dictionary.

---

### 1.2 Page Layout (Vertical, Top to Bottom)

```
┌─────────────────────────────────────────────┐
│  HEADER                                      │
│  Dictionary name · Login status · Theme      │
├─────────────────────────────────────────────┤
│  MODULE 1 — MAP                              │
│  Row 1: [INFO][PART][SEC ][EC  ]  ← cats     │
│  Row 2: [PART][MYLO][MYSC]  ← notes          │
│  Row 3: [NEW ][EDIT][ARCH]  ← details        │
├─────────────────────────────────────────────┤
│  MODULE 2 — DICTIONARY TREE                  │
│  Participants — Manage participant records   │
│  Lockers — Locker inventory and assignments  │
│  Bars — Bar management                       │
├─────────────────────────────────────────────┤
│  MODULE 3 — COMMAND BAR                      │
│  Lockers.EC.participants          ← address  │
│  EC.participants ›  ____________             │
│    [part]  [lock]  [bars]  [atte]  [sche]    │
└─────────────────────────────────────────────┘
```

---

### 1.3 Header

Displays:
- Full dictionary name (e.g., `N.Seattle.Housing.Homeless.SHARE.Lockers`)
- Current user role and name (or "Guest" if not logged in)
- Light / Dark theme toggle (persisted to localStorage key `sage_theme`)
- Log In / Log Out button

**After login:** the user is automatically navigated to their role's home category (the first category they have access to). The header Log In button and the in-dictionary login prompt are the same authentication system — logging in from either is equivalent.

---

### 1.4 Map (Module 1)

The map is a live navigation tracker. It is text-based — no graphics.

**Structure:**
- One row of colored boxes per active hierarchy level (max 4 rows)
- Each row shows exactly as many boxes as items **visible to the current user's role**
- Do not show empty slots — the map reflects only what exists and is accessible

**Box labels — abbreviation rule:**
- If the key is ≤ 3 characters: display as uppercase (e.g., `EC` → `EC`, `sec` → `SEC`)
- If the key is 4+ characters: display the first 4 alphanumeric characters of the label, uppercase (e.g., `participants` / "Participants" → `PART`, `attendance` / "Attendance" → `ATTE`)

**Color system — consistent by slot position across all dictionaries:**

| Slot | Color Name | Hex     |
|------|------------|---------|
| 1    | Red        | #E53E3E |
| 2    | Orange     | #DD6B20 |
| 3    | Yellow     | #D69E2E |
| 4    | Green      | #38A169 |
| 5    | Teal       | #319795 |
| 6    | Blue       | #3182CE |
| 7    | Indigo     | #5A67D8 |
| 8    | Purple     | #805AD5 |
| 9    | Pink       | #D53F8C |
| 10   | Gray       | #718096 |

**Box states:**
- **Active** (current position): Full color background, white text
- **Available** (exists but not in current path): 10% color tint background, matching color text

**Behavior:**
- Clicking a map box navigates directly to that position
- Rows 2–4 appear only when a parent level is selected
- Map rows reflect only the categories the current user's role can see (see 1.8)

---

### 1.5 Dictionary Tree (Module 2)

Text-only navigation panel. No icons, no addresses.

**Layout:**
- Single column (one item per row)
- Maximum 10 items per level (enforced by slot limit — see Section 4)
- Each item is a single line: **Label — Short description**
  - Example: `Participants — Manage participant records`
  - If no description: just the label
- A 4px colored left border (using slot color) marks each item's position identity
- Clicking an item navigates into it
- Items the user cannot access appear faded with a lock indicator; clicking prompts login

**What is NOT shown in the tree:**
- Icons
- Dictionary addresses (shown in Module 3 only)
- Definitions on a second line

**Breadcrumb above tree:** `Lockers › EC › participants`
Breadcrumb segments are clickable to navigate up.

**Grouping rule:** If a note would exceed 10 children, create a parent note and place the children inside it as details. A note can contain at most 10 details. Example: `reports` and `bag_tag` grouped into `center` (Report Center) with both as details.

---

### 1.6 Command Bar — Sage_OS Command Protocol v1 (Module 3)

Located at the bottom of every dictionary page. Always visible.

**The command bar is the primary navigation mechanism. Screens must not include navigation buttons** (no Cancel, no Back, no + New, no Go to X). The only buttons inside a screen are action buttons that submit or execute something (Save, Search, Issue, Archive, etc.).

#### Address Display

Above the prompt, the **full current address** is shown in muted monospace:
```
Lockers.EC.participants
```
This is the address the user is currently at. It updates on every navigation.
It can be copied and pasted back into the command bar to navigate anywhere.

#### Prompt Format

The prompt shows the last 1–2 path segments as context:

```
EC.participants ›  ___________________________
```

- At root: `Lockers ›`
- At category: `EC ›`
- At note: `EC.participants ›`
- At detail: `participants.new ›`

#### Addressing Rules

Commands are relative to current position unless prefixed otherwise.
The dictionary name prefix is always stripped — `Lockers.EC.participants` and `EC.participants` both work.

| Input | Current position | Navigates to |
|-------|-----------------|--------------|
| `participants` | EC | EC.participants |
| `new` | EC.participants | EC.participants.new |
| `part` | EC | EC.participants (prefix match) |
| `..` | EC.participants | EC |
| `...` | EC.participants | root |
| `.lockers` | EC.participants | EC.lockers (sibling) |
| `.lockers.inventory` | EC.participants | EC.lockers.inventory |
| `Clerk.bars` | anywhere | Clerk.bars (absolute) |
| `EC.participants.new` | anywhere | EC.participants.new (absolute) |
| `Lockers.EC.participants.new` | anywhere | EC.participants.new (prefix stripped) |
| `/` | anywhere | root |

Matching priority:
1. Special tokens (`..`, `...`, `/`, `.sibling`)
2. Dictionary name prefix stripped, then parsed as absolute
3. Exact match in current children
4. Prefix match in current children
5. Label match in current children (fuzzy)
6. Absolute address (first segment is a known category)

#### Note Default Routing

Navigating to a note (e.g., `EC.participants`) must always show content — never a blank or "under construction" state. Each note must have a registered default screen. If a note's primary function is a list or form, the note-level address shows that directly. Notes with multiple details (e.g., `attendance.worker`, `attendance.mandatory`) route to the most common child by default.

#### Suggestion Strip

Shows up to 5 suggestions from current level's children, updating as user types.
Each pill shows: `key — Label`.
Below the strip: a preview line shows the full address the highlighted suggestion would navigate to.

#### Keyboard Shortcuts

| Key | Context | Action |
|-----|---------|--------|
| `/` | anywhere (not in text input) | Focus command bar |
| `Ctrl+K` | anywhere | Focus command bar |
| `Tab` | command bar | Fill current top suggestion (doesn't navigate) |
| `Enter` | command bar | Navigate to parsed command or highlighted suggestion |
| `Esc` | command bar, text present | Clear input text |
| `Esc` | command bar, empty | Go up one level |
| `Backspace` | command bar, empty | Go up one level |
| `↓` | command bar, input empty | Scroll command history (newest first) |
| `↓` | command bar, input has text | Cycle down through suggestions |
| `↑` | command bar, input empty | Scroll command history |
| `↑` | command bar, input has text | Cycle up through suggestions |
| `Ctrl+←` | anywhere | Go up one level |
| `Ctrl+→` | anywhere | Enter first child of current node |
| `Ctrl+Home` | anywhere | Go to dictionary root |

#### Command History

The last 10 executed navigation paths are stored in session memory.
Up/Down arrow when input is empty scrolls through history (like a terminal).
History is cleared when the dictionary session ends.

---

### 1.7 Theme

- **Light theme:** White background, dark text, colored map boxes
- **Dark theme:** `#0f172a` background, light text, same map box colors
- Toggle button in header: `☀ Light` / `☾ Dark`
- Persisted to `localStorage` under key `sage_theme`
- Default: Light

---

### 1.8 Login Rules and Role Visibility

Each category within a dictionary may require a different login level.

**Login types:**
- **None** — public, no login (e.g., `.info`)
- **Shared** — one username/password shared by a user class (e.g., `.part`)
- **Individual** — personal username + password with a named role (e.g., `.EC`, `.Clerk`, `.sec`)

**Role normalization:** All role values are stored and compared in lowercase. The database may store roles in any case; the application normalizes to lowercase on login. This prevents case-mismatch visibility failures.

**Visibility rule — users see only the categories relevant to their role:**

| Role | Visible categories |
|------|--------------------|
| Guest (no login) | `info`, `part` (to prompt login) |
| Participant (`part`) | `info`, `part` |
| Security (`sec`) | `info`, `sec` |
| EC (`ec`) | `info`, `EC` |
| Clerk (`clerk`) | `info`, `EC`, `Clerk` |

Categories not listed for a role do not appear in the map or tree for that user.
This is a **display rule**, not just an access rule — hidden categories are not visible at all.

Access control (login required prompt) is a separate concern from visibility.
Attempting to navigate to a locked category via the command bar shows a login prompt.

**Session:** Login state stored in session memory (cleared on tab/window close).

---

### 1.9 Editable Info Pages

Any dictionary may designate its public `info` category as **staff-editable**:

- All users can view `info` notes (read-only by default)
- Staff roles defined by the dictionary (e.g., EC, Clerk) see an **inline edit textarea** instead of read-only text
- Content is plain text; line breaks are preserved
- Saved to persistent storage (keyed as `info.<noteKey>`)
- Changes take effect immediately for all users on next page load

This pattern allows the same `info` screens to serve as both public content pages and a lightweight CMS for staff.

---

### 1.10 Word Rule (Critical)

In any Sage_OS dictionary, **each word has exactly one definition.**

- A word used in two categories must mean the same thing in both.
- If a dictionary defines a word that conflicts with the Core Dictionary definition, the Core Dictionary wins and a conflict error is logged.
- Conflict detection is a planned Sage_OS system feature (not yet enforced in v0.3).

---

### 1.11 Data Loading

Dictionary data must be loaded on application initialization, not only on login.

- All data sources are fetched in parallel on startup
- Individual query failures must not prevent other data from loading — each source is wrapped in an independent error handler that returns an empty result on failure
- If the user is not authenticated and a data source requires it, that source returns empty; it will be re-fetched after login
- `S.loaded` is set to `true` only after all queries complete (even if some returned empty due to failure)

---

## SECTION 2 — CORE FUNCTIONS (Verbs)

These are the canonical action words shared across all dictionaries.
Individual dictionaries use these words; they do not redefine them.

| Word         | Meaning                                                    |
|--------------|------------------------------------------------------------|
| `new`        | Create a new record or item                                |
| `edit`       | Modify an existing record                                  |
| `archive`    | Mark as inactive; hidden from default views, data kept     |
| `delete`     | Permanently remove a record (use sparingly; prefer archive)|
| `search`     | Find records by keyword, name, or attribute                |
| `view`       | Display a record or list in read-only mode                 |
| `assign`     | Link one record to another (e.g., locker to participant)   |
| `release`    | Remove a link between two records                          |
| `renew`      | Extend the active period of a record                       |
| `issue`      | Formally create and deliver an action item (e.g., a bar)   |
| `dismiss`    | Close or discard a pending recommendation or alert         |
| `log`        | Record an event with timestamp                             |
| `check_in`   | Record arrival of a person                                 |
| `check_out`  | Record departure of a person                               |
| `manage`     | Access administrative controls for a resource              |
| `reset`      | Restore a setting or credential to a default state         |
| `message`    | Send or read a communication                               |
| `report`     | Generate or view a summary of data                         |
| `import`     | Bring data in from an external source                      |
| `export`     | Send data out to an external format                        |
| `mark`       | Flag a record with a status indicator                      |

---

## SECTION 3 — CORE NOUNS

These are the canonical object words shared across all dictionaries.

| Word           | Meaning                                                        |
|----------------|----------------------------------------------------------------|
| `participant`  | A person enrolled in the organization's program                |
| `user`         | A staff or admin account with a named role                     |
| `locker`       | A physical storage unit assigned to a participant              |
| `bar`          | A formal exclusion from a program or facility                  |
| `attendance`   | A record of presence at a required event                       |
| `schedule`     | A list of upcoming required dates                              |
| `note`         | A free-text annotation attached to a record                    |
| `counter`      | A numeric tally that increments per event per day              |
| `status`       | The current standing of a participant or item                  |
| `session`      | A timed period of activity (e.g., locker access time)          |
| `message`      | A unit of communication on a message board or chat             |
| `report`       | A generated summary of records over a time range               |
| `role`         | A named permission level (e.g., EC, Clerk, sec, part)          |
| `recommendation` | A pending suggestion requiring staff review and action       |
| `renewal`      | The act of extending a participant's active membership period  |
| `shift`        | A scheduled work obligation for a participant                  |
| `inventory`    | A complete enumerated list of physical assets                  |
| `tag`          | A label attached to a physical item for tracking               |
| `board`        | A shared space where multiple users can post messages          |
| `address`      | The full hierarchical path to any dictionary item              |
| `center`       | A grouping note that consolidates related details              |

---

## SECTION 4 — ERROR VOCABULARY

| Error                  | Meaning                                                   |
|------------------------|-----------------------------------------------------------|
| `word_conflict`        | A dictionary word conflicts with Core Dictionary          |
| `duplicate_definition` | Same word defined twice within one dictionary             |
| `address_not_found`    | The requested dictionary address does not exist           |
| `permission_denied`    | User's role does not have access to this address          |
| `login_required`       | Address requires authentication; user is not logged in    |
| `max_depth_exceeded`   | Dictionary path exceeds 5 levels                          |
| `slot_limit`           | A level already has 10 items; cannot add more             |
| `visibility_denied`    | Category is not visible to the current user's role        |
| `data_load_partial`    | One or more data sources failed to load; data may be incomplete |

---

*Core Dictionary v0.3 — Sage_OS. All other dictionaries extend this file.*
