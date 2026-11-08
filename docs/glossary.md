# Glossary

User
  Any person who can log in. Every active user can book rooms.
  There is no "Employee" role.

Borrower
  The user a booking is made for.

Role
  A named permission assigned to a user. A user can have many roles.
  Naming: UPPER_SNAKE_CASE.

System role
  A role checked directly by code and not editable in the portal.
  MVP: ADMIN (manages rooms and the catalog). ADMIN alone gives no approval authority.

Approver role
  A role that is assigned to a room as one of its approvers.
  Naming: APPROVER_<DIVISION_CODE>, e.g. APPROVER_GA, APPROVER_BOD.
  The word "APPROVAL" is never used in role names.

Requires approval
  A room setting. If false, new bookings are created as "Approved".
  If true, the room must have at least one approver role.

Approval
  The decision (approve or reject) made by a user holding one of the room's approver roles.

Auto-approved
  A booking created as "Approved" because the borrower holds one of the room's approver roles.