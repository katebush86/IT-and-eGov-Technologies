**User Story**

As a librarian, I want to manage the book catalogue, book copies, and lending information, search books by ISBN, title, or author, and monitor their availability and lending status so that library records remain accurate and up to date.


**Context**

The library keeps a register of book titles identified by a unique ISBN. Each book title contains information such as title, author, genre, page count, and publication year.

A book title may have one or more copies. Each copy has its own status and lending information.

Borrower information is not stored as free text. The system stores only the borrower's personal identification code for the relevant copy and retrieves borrower details from a separate register whenever needed. One source of truth per fact.


**Acceptance Criteria**

1. A book title can be registered with an ISBN, title, author, genre, page count, publication year, and one or more copies.

2. An ISBN identifies exactly one book title, while individual copies are managed separately.

3. Invalid data is rejected with a clear message: duplicate ISBN, page count equal to or less than zero, or malformed personal identification code.

4. The catalogue is searchable by title, author, or ISBN and filterable by genre and copy status.

5. Library users can see whether a book is in the library catalogue and whether any of its copies are currently available for borrowing.

6. For a borrowed copy, the librarian sees the borrower's information fetched live from the borrower register. Borrower details are never stored locally.

7. A copy can have one of the following statuses:

- Available
- Borrowed
- In Queue
- Under Repair
- Pre-order
- Lost

8. If the publication year is in the future, all copies of the book title must have the status "Pre-order".

9. Copies with status "Under Repair", "Pre-order", or "Lost" cannot have an assigned borrower.

10. When a copy is borrowed, the lending information contains the borrower's personal identification code and lending due date.

11. The default lending period is 14 days.

12. If the current date exceeds the lending due date, the system displays the number of overdue days.

13. Copies of the same book title are handled independently. For example, if a title has two copies, one copy may be "Borrowed" while the other is "Available".

14. A book title cannot be removed while any of its copies are marked "Borrowed" or "In Queue".
