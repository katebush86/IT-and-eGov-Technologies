**User Story**

As a library user, I want to search books by ISBN, title, or author and view their availability and lending status so that I can easily find books and know whether they can be borrowed.

As a librarian, I want to maintain the book catalogue and lending information so that library records remain accurate and up to date.


**Context**

The library keeps a register of books identified by a unique ISBN. Each book is registered once and contains information such as title, author, genre, page count, publication year, and status.

Borrower information is not stored as free text. The system stores only the borrower's personal identification code and retrieves borrower details from a separate register whenever needed. One source of truth per fact.


**Acceptance Criteria**
1. A book can be registered with an ISBN, title, author, genre, page count, publication year, status, optional borrower, and lending due date.
2. An ISBN identifies exactly one active book.

3. Obvious invalid data is rejected with a clear message: duplicate ISBN, page count equal to or less than zero, publication year in the future without a "Pre-order" status, or malformed personal identification code.

4. The catalogue is searchable by title, author, or ISBN and filterable by genre and status.

5. Library users can view a book's current status and availability.

6. For a borrowed book, the librarian sees the borrower's information fetched live from the borrower register and never stored locally.

7. A book can have one of the following statuses:

 - Available
 - Borrowed
 - In Queue
 - Under Repair
 - Pre-order
 - Lost
8. If the publication year is in the future, the book status must automatically be set to "Pre-order".

9. Books with status "Under Repair", "Pre-order", or "Lost" cannot have an assigned borrower.

10. The default lending period is 14 days.

11. If the current date exceeds the lending due date, the system displays the number of overdue days.
