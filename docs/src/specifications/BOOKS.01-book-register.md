# BOOKS.01 · Book catalogue and lending

> **BOOKS.01** · state **Proposed** · traces from `BOOKS-US-001` · source `backend/src/main/java/ee/taltech/ite4120/books/`, `backend/src/main/resources/books/db/changelog/`, `frontend/src/pages/`

## Description

A library catalogue: register a book title with one or more physical copies, search and filter the catalogue, see each copy's status and availability, record lending information per copy, and remove a book. A title is identified by its ISBN; each physical copy has its own status, borrower and due date. Borrower identity is held as a personal code only; the borrower's name and details are resolved live from the borrower register through the borrower-registry adapter (mock in this course, X-Road-shaped in production) and are **never stored locally**.

Excluded: authentication and role-based visibility (mock auth only; library users and librarians see the same screens in this course), adding copies to an existing title, editing a registered title, lending history (only the current lending state of each copy is kept), a managed genre list (see Genre codes), fines, audit beyond the platform sys columns.

## Data model *(→ session 2, changesets 01–02)*

| Table         | Field                    | Type    | Rules                                                                                                       |
|---------------|--------------------------|---------|-------------------------------------------------------------------------------------------------------------|
| `books.book`  | `id` PK                  | bigint  | a title; from `core.seq_id`                                                                                 |
|               | `isbn`                   | text    | required; 10 or 13 digits after stripping hyphens and spaces; stored without separators; **unique among active rows** |
|               | `title`                  | text    | required, ≤255                                                                                              |
|               | `author`                 | text    | required, ≤255                                                                                              |
|               | `genre`                  | text    | required; one of the genre codes below (check constraint)                                                   |
|               | `page_count`             | integer | required; **must be greater than 0**                                                                        |
|               | `publication_year`       | integer | required; 4 digits; **may be in the future** — see the Pre-order rule                                       |
|               | `sys_*`                  | —       | platform columns, trigger-managed; `sys_status` marks retired rows (soft delete)                            |
| `books.copy`  | `id` PK                  | bigint  | from `core.seq_id`                                                                                          |
|               | `book_id`                | bigint  | required; FK → `books.book`                                                                                 |
|               | `status`                 | text    | required; one of the six status codes below (check constraint)                                              |
|               | `borrower_personal_code` | text    | exactly 11 digits — a string, never a number; see the status rules below                                    |
|               | `due_date`               | date    | required for BORROWED, empty otherwise; defaults to the lending date + **14 days**                          |
|               | `sys_*`                  | —       | platform columns, trigger-managed; `sys_status` marks retired rows (soft delete)                            |

The borrower's name is not a column: it comes from the registry adapter (see API). There is no foreign key to the borrower register, because it is a separate system; the personal code is validated for format only.

### Genre codes

A fixed list, enforced by a check constraint and mirrored in the screens; it is not a table. Adding a genre needs a migration and a screen change.

`FICTION`, `NON_FICTION`, `FANTASY`, `SCIENCE_FICTION`, `MYSTERY`, `BIOGRAPHY`, `HISTORY`, `CHILDREN`, `POETRY`, `OTHER`

### Status codes

| Code           | Meaning                                              | Borrower | Due date |
|----------------|------------------------------------------------------|----------|----------|
| `AVAILABLE`    | on the shelf, can be borrowed                        | none     | none     |
| `BORROWED`     | lent out                                             | required | required |
| `IN_QUEUE`     | held for the person named as borrower, who asked for it while it was out; not yet collected | required | none     |
| `UNDER_REPAIR` | being repaired, cannot be borrowed                   | none     | none     |
| `PRE_ORDER`    | new, not yet arrived                                 | none     | none     |
| `LOST`         | lost                                                 | none     | none     |

A copy with no borrower must also have no due date. A copy has one status and at most one borrower at a time; when a `BORROWED` copy is returned and someone is waiting, it is set to `IN_QUEUE` for that person, and to `BORROWED` when they collect it.

### Rules that span fields

- **Pre-order rule:** if a title's `publication_year` is in the future (later than the current calendar year), every one of its copies must have status `PRE_ORDER`. Registering such a title creates its copies as `PRE_ORDER`; any attempt to set another status on them is refused.
- **Default status:** a title registered with a publication year that is not in the future gets copies with status `AVAILABLE`.
- **Overdue days** are not stored. For a `BORROWED` copy: if today is later than `due_date`, overdue days = today − `due_date` in whole days; otherwise 0. For any other status it is 0.

### Design decisions

- **Title and copy are separate tables** because one title can have several physical copies with independent status; otherwise title, author and genre would be repeated for every copy and could contradict each other.
- **Every table has a surrogate `id`** (platform convention). The ISBN is the business identifier of a title, unique among active titles. Removing a title is a soft delete (platform convention), so after removal its ISBN can be registered again.
- **The personal code is a string** because it is an identifier, not a quantity; a number type would lose leading zeros.
- **Only the personal code is stored** because it is the minimum needed to link a copy to a person; the name and details live in the borrower register (one source of truth per fact). The code is cleared when the copy stops being borrowed or queued, so no personal data is kept after the lending ends.
- **Genre and status are fixed lists** enforced by check constraints, which keeps the model at two tables. The cost is that adding a genre needs a migration and a screen change; a genre table with its own endpoint is the natural next round.
- **Overdue days are computed** because a stored value would be wrong again the next day.
- **No lending history:** the story asks who has a copy now, not who had it before. A loan history table is the natural next round.

## API *(→ session 3)*

| Method + path              | Does                                                                                                                                                                 | Failure answers                      |
|----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------|
| `GET /api/books`           | the catalogue, **one row per copy**: `textContains` (matches ISBN, title **or** author), `genre`, `status`; each row carries the title data, the copy's status, due date, `overdueDays` and `borrowerName` | 400 unknown genre or status |
| `POST /api/books`          | register a title and create `copyCount` copies (default 1, at least 1); answers **201 + Location**                                                                   | 400 validation · 409 duplicate ISBN  |
| `PUT /api/copies/{id}`     | set the copy's lending data: `status`, `borrowerPersonalCode`, `dueDate` (the three fields are replaced as a whole)                                                  | 400 · 404                            |
| `DELETE /api/books/{id}` | remove (retire) the title and all its copies, a soft delete — **204**                                                                                                                        | 404 · 409 a copy is BORROWED         |

`POST /api/books` takes `isbn`, `title`, `author`, `genre`, `pageCount`, `publicationYear` and optionally `copyCount`. All its copies start as `AVAILABLE` (or `PRE_ORDER`, see the Pre-order rule); lending data is set per copy afterwards with `PUT /api/copies/{id}`. In a `PUT`, a BORROWED copy with no `dueDate` gets today + 14 days.

Every error is `application/problem+json`; a caller's mistake never answers 500. All endpoints require `Authorization: Bearer <user>` (mock auth); the API contract is live at `/swagger-ui.html`.

**Failure policy** for the borrower lookup: **fail open** — when the borrower register is unreachable, `borrowerName` is `null` and everything else is returned normally. Rationale: the borrower's name is auxiliary here, never a precondition for the catalogue.

## UI *(→ session 4)*

| Screen        | Element          | Component                 | Rules                                                                                                                                              |
|---------------|------------------|---------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| Catalogue     | table            | `ResourceList`            | one row per copy; columns: ISBN (locked), title, author, genre tag, year, pages (hidden by default), status tag, due date, overdue days            |
|               | search field     | `ResourceList` search     | searches ISBN, title and author                                                                                                                    |
|               | filters          | `Select` ×2               | genre (the fixed genre list), status (the six status codes)                                                                                        |
|               | overdue days     | table cell                | shown only when greater than 0, as a warning tag "N days overdue"                                                                                  |
|               | detail panel     | `ResourceList.detailView` | shows the borrower's name; silently omits it when it is `null`                                                                                     |
|               | row action       | `AppButtonPrimary`        | "Update lending" opens the lending form for that copy                                                                                              |
|               | row action       | `AppButton`               | "Remove book" asks for confirmation, then `DELETE`; a 409 problem text is shown in a notification                                                  |
|               | register button  | `AppButtonPrimary`        | opens the register form                                                                                                                            |
| Register form | ISBN             | `Input`                   | required, 10 or 13 digits                                                                                                                          |
|               | title            | `Input`                   | required, ≤255                                                                                                                                     |
|               | author           | `Input`                   | required, ≤255                                                                                                                                     |
|               | genre            | `Select`                  | required, options from the fixed genre list                                                                                                        |
|               | pages            | `Input` (number)          | required, greater than 0                                                                                                                           |
|               | year             | `Input` (number)          | required; a future year shows the hint "copies will be registered as Pre-order"                                                                    |
|               | copies           | `Input` (number)          | at least 1, default 1                                                                                                                              |
|               | submit           | `AppButtonPrimary`        | 400/409 problem text shown verbatim in a notification                                                                                              |
| Lending form  | copy             | read-only text            | ISBN, title and copy number of the selected copy                                                                                                   |
|               | status           | `Select`                  | required, the six status codes                                                                                                                     |
|               | borrower code    | `Input`                   | pattern `\d{11}`; enabled only for BORROWED and IN_QUEUE                                                                                           |
|               | due date         | `DatePicker`              | enabled only for BORROWED; prefilled with today + 14 days                                                                                          |
|               | submit           | `AppButtonPrimary`        | 400 problem text shown verbatim in a notification                                                                                                  |

## Business tests *(→ executable in `BookBusinessRulesIT`)*

1. **A future publication year means Pre-order.** Given a title whose publication year is next year, when it is registered, then all its copies are `PRE_ORDER`. And when the status of one of those copies is set to `AVAILABLE`, then the change is refused with "status must be PRE_ORDER while publication year is in the future".
2. **The page count must be positive.** Given a request to register a title with a page count of 0 (or −5), when it is submitted, then it is refused with "pageCount must be greater than 0".
3. **Overdue days are shown once the due date has passed.** Given a `BORROWED` copy with a due date 3 days ago, when the catalogue is read, then its `overdueDays` is 3. And given a due date of today or later, then `overdueDays` is 0 and the UI shows no overdue tag.
4. **Copies that are not out have no borrower.** Given a copy, when it is set to `UNDER_REPAIR`, `PRE_ORDER` or `LOST` together with a borrower personal code, then the change is refused with "borrower not allowed for this status". And when a `BORROWED` copy is set to `UNDER_REPAIR` without a borrower, then its borrower and due date become empty.
5. **A borrowed copy has a borrower and a due date.** Given a request to set status `BORROWED` without a borrower personal code, when it is submitted, then it is refused (400). And given a `BORROWED` request with a borrower but no due date, then the due date is set to today + 14 days.
6. **An ISBN identifies exactly one title.** Given a registered title with ISBN X, when a second title claims X (with or without hyphens), then the registration conflicts (409). And when the first title has been removed, then X may be registered again.
7. **A malformed personal code is rejected.** Given a lending request whose personal code has 10 digits or contains letters, when it is submitted, then it is refused (400) with a message naming the personal code.
8. **The genre must be on the list.** Given a registration with a genre that is not one of the genre codes, when it is submitted, then it is refused (400).
9. **Copies are independent.** Given a title with 2 copies, when one is set to `BORROWED`, then the catalogue shows one `BORROWED` row and one `AVAILABLE` row for that ISBN.
10. **A title with a borrowed copy cannot be removed.** Given a title with a `BORROWED` copy, when it is removed, then the removal conflicts (409). And given a title whose copies are not borrowed, when it is removed, then 204 is returned and the title and its copies no longer appear in the catalogue.
11. **The catalogue can be searched and filtered.** Given several titles, when searching by part of an ISBN, a title or an author, then only the matching rows are returned. And when filtering by genre and status, then only rows with both are returned.
12. **The borrower register being down does not break the catalogue.** Given the borrower register is unreachable, when the catalogue is read, then it answers 200 and `borrowerName` is `null` for borrowed copies.
