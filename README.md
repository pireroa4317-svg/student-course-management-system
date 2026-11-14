STUDENT COURSE MANAGEMENT SYSTEM
=================================

Requirements
------------
- Python 3.9 or newer
- No third-party packages are required.

How to run
----------
1. Extract the project folder.
2. Open Command Prompt or Terminal in that folder.
3. Run:
       python student_course_management.py
   On some Windows installations, use:
       py student_course_management.py

Menu options
------------
1. Add Course              - records course code, title and units.
2. View All Courses        - displays all recorded courses.
3. Search Course by Code   - finds a course using its code.
4. Compute Total Units     - adds all course units recursively.
5. Save to File            - stores records in courses.json.
6. Load from File          - validates and restores saved records.
7. Exit Program            - closes the application.

Security and reliability notes
------------------------------
- JSON is used instead of pickle, avoiding unsafe object deserialization.
- Course fields, unit ranges, duplicate codes and file structure are validated.
- Loading is transactional: existing data is not replaced unless every record is valid.
- Saving uses a temporary file followed by replacement to reduce the chance of a
  partially written data file.
- The application limits the number of courses and the maximum file size.

The application is a local learning project, not a production student-information
system. Production deployment would additionally require authenticated users,
role-based access, encrypted backups, audit logging and managed database storage.
