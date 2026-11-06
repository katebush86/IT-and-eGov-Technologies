**User Story**

As a librarian, I want to register and manage books identified by a unique ISBN, maintain catalogue and lending information, search and filter books, monitor availability and overdue loans, and access borrower information when needed so that library records remain accurate and library operations are performed efficiently.


**Context**

The library maintains a catalogue of books, each identified by a unique ISBN. Each book is registered only once and contains information such as title, author, genre, page count, year of publication, and status.

Borrower information is not stored locally as free text. The system stores only the borrower's personal identification code and retrieves borrower details from an external borrower register whenever required, ensuring a single source of truth for borrower data.


**Acceptance Criteria**

1. A book can be registered with an ISBN, title, author, genre, page count, year of publication, and status.

2. A book can have one of the following statuses:
   - Available
   - Borrowed
   - In Queue
   - Under Repair
   - Pre-order
   - Lost

3. When a book is borrowed, the lending information includes the book ISBN, the borrower's personal identification code, and the lending due date.

4. Each book record must have a unique ISBN.

5. Obvious invalid data is rejected with a clear error message, including duplicate ISBNs, page counts less than or equal to zero, and malformed personal identification codes.

6. The catalogue is searchable by title, author, or ISBN and filterable by genre and status.

7. Library users can view a book's current status and availability.

8. For a borrowed book, the librarian can view borrower information retrieved live from the borrower register, which is never stored locally.

9. If the year of publication is in the future, the book status is automatically set to "Pre-order".

10. Books with the statuses "Under Repair", "Pre-order", or "Lost" cannot have an assigned borrower.

11. The default lending period is 14 days.

12. If the current date exceeds the lending due date, the system displays the number of overdue days.
