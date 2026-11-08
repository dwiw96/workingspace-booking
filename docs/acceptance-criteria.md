US-01:  As a user, I want to log in so that I can access features based on my role.
```gherkin
Scenario: Log in with correct email and password
    Given a registered user with email "dwiw@mail.com" and password "123"
    When I log in with email "dwiw@mail.com" and password "123"
    Then I'm logged in and see the features for my role

Scenario Outline: Log in fails with invalid credentials
    Given a registered user with email "dwiw@mail.com"
    When I log in with email "<email>" and password "<password>"
    Then I see the message "Invalid email or password"
    And I remain on the log in page

    Examples:
        | email              | password      |
        | dwiw@mail.com      | wrong-pass    |
        | unknown@mail.com   | any-pass      |

Scenario Outline: Log in with different uppercase and lowercase email
    Given a registered user with email "dWIw@mail.com"
    When I log in with email "<email>" and the correct password
    Then I'm logged in and see the features for my role

    Examples:
        | email |
        | dwiw@mail.com |
        | DWIW@MAIL.COM |
        | Dwiw@Mail.cOm |

Scenario: Log in with correct email and password failed because account is inactive
    Given a registered user with email "dwiw@mail.com" but status is "inactive"
    When I log in with email "dwiw@mail.com" and correct password
    Then I see the message "Account is inactive, please contact the admin"
    And I remain on the log in page

Scenario: Log in with wrong password failed because account is inactive
    Given a registered user with email "dwiw@mail.com" whose account status is inactive
    When I log in with email "dwiw@mail.com" and wrong password
    Then I see the message "Invalid email or password"
    And I remain on the log in page

Scenario Outline: Log in fails when a required field is empty
    Given I am on the log in page
    When I submit the form with "<field>" left empty
    Then I see the message "<message>" below the "<field>" field
    And I remain on the log in page

    Examples:
        | field    | message                    |
        | email    | Email is required         |
        | password | Password is required      |

Scenario: Too many login attempts from the same client
    Given I have failed to log in 5 times within 1 minute
    When I try to log in again
    Then I see the message "Too many attempts. Please try again later"
    And the response is the same whether or not the email is registered

Scenario: Access the system with an expired session
    Given I logged in more than 1 hour ago
    When I open a page that requires login
    Then I see the message "Your session has expired. Please log in again"
    And I am redirected to the log in page
```


US-04:  As a borrower, I want to book a room so that I can use it at a specific time.

Business rules:
1. Start time must be in the future.
2. End time must be after start time.
3. Duration must not exceed the room's maximum booking hours.
4. A room cannot have overlapping bookings with status Pending or Approved.
   Time ranges are half-open: 09:00–10:00 and 10:00–11:00 do not overlap.
5. If the room does not require approval, the booking is created as "Approved".
6. If the room requires approval and the borrower holds one of its approver roles,
   the booking is created as "Approved" (auto-approved), and the status history
   records the reason: "Auto-approved: borrower holds an approver role for this room".
7. Otherwise, the booking is created as "Pending".

```gherkin
Scenario: Book an available room that does not require approval
    Given room "Meeting Room A" does not require approval
    And it has no bookings between 2026-11-10 09:00 WIB and 10:00 WIB
    When I book it from 2026-11-10 09:00 WIB to 10:00 WIB
    Then the booking is created with status "Approved"

Scenario: Book a room that requires approval
    Given "Meeting Room A" requires approval by role "APPROVER_GA" 
    And I have no approver role for "Meeting Room A"
    And "Meeting Room A" is available all time in 2026-11-10 09:00
    When I book it from 2026-11-10 09:00 WIB to 10:00 WIB
    Then the booking is created with status "Pending"

Scenario: Adjacent booking is allowed
    Given "Meeting Room A" has an approved booking from 2026-11-10 09:00 WIB to 10:00 WIB
    When I book it from 2026-11-10 10:00 WIB to 11:00 WIB
    Then the booking is created

Scenario Outline: Rejected or cancelled bookings do not block the slot
    Given "Meeting Room A" has a status "<Status>" booking from 2026-11-10 09:00 WIB to 10:00 WIB
    When I book it from 2026-11-10 09:00 WIB to 10:00 WIB
    Then the booking is created

    Examples:
        | Status    |
        | Rejected  |
        | Cancelled |

Scenario: Booking in the past is rejected
    Given the current time is 2026-11-10 10:00 WIB
    When I book a "Meeting Room A" from 2026-11-10 09:00 WIB to 11:00 WIB
    Then the request is rejected with a validation error

Scenario Outline: Booking longer than the maximum duration is rejected
    Given "Meeting Room A" has a maximum booking duration of 4 hours
    When I book it for "<Booking Hours>" hours
    Then the request is "<Response>"

    Examples:
        | Booking Hours | Response                          |
        | 4             | Created                           |
        | 5             | Rejected with a validation error  |

Scenario: Two simultaneous requests for the same slot
    Given "Meeting Room A" is available from 2026-11-07 09:00 WIB to 10:00 WIB
    When two users submit bookings for that slot at the same moment
    Then exactly one booking is created
    And the other request receives a conflict error
    
Scenario: Booking is auto-approved when the borrower holds an approver role for the room
    Given "Meeting Room BOD" requires approval by role "APPROVER_GA"
    And I have role "APPROVER_GA"
    And "Meeting Room BOD" is available from 2026-11-10 10:00 WIB to 12:00 WIB
    When I book "Meeting Room BOD" from 2026-11-10 10:00 WIB to 12:00 WIB
    Then the booking is created with status "Approved"
    And a status history entry records that it was auto-approved

Scenario: Admin still need approval when booking a room that doesn't hold an approver for that room
    Given "Meeting Room BOD" requires approval by role "APPROVER_BOD"
    And I only have role "ADMIN"
    When I book "Meeting Room BOD" from 2026-11-02 08:00 WIB to 11:00 WIB
    Then the booking is created with status "Pending"

Scenario: An approver role for another room does not auto-approve the booking
    Given "Meeting Room BOD" requires approval by role "APPROVER_BOD"
    And I'm a user that have role "APPROVER_IT"
    When I book "Meeting Room BOD" from 2026-11-10 10:00 WIB to 12:00 WIB
    Then the booking is created with status "Pending"

Scenario Outline: Booking is rejected when the slot is held by a <status> booking
    Given "Meeting Room A" has a <status> booking from 2026-11-10 09:00 WIB to 10:00 WIB
    When I book "Meeting Room A" from 2026-11-10 09:00 WIB to 10:00 WIB
    Then the request is rejected with a conflict error
    And no booking is created

    Examples:
        | status   |
        | pending  |
        | approved |

Scenario Outline: Booking room but the time is overlap 
    Given "Meeting Room BOD" booked from "<From Date Time>" to "<To Date Time>"
    When I book "Meeting Room BOD" from 2026-11-05 14:00 WIB to 2026-11-05 16:00 WIB
    Then the request is rejected

    Examples:
        | From Date Time        | To Date Time          |
        | 2026-11-05 15:50 WIB  | 2026-11-05 16:30 WIB  |
        | 2026-11-05 13:00 WIB  | 2026-11-05 15:00 WIB  |
        | 2026-11-05 13:00 WIB  | 2026-11-05 17:00 WIB  |
        | 2026-11-05 14:30 WIB  | 2026-11-05 15:00 WIB  |

Scenario Outline: Booking room but the time start and time and is overlap 
    Given I successfully login
    And in the booking page
    When I book "Meeting Room A" from "<From Date Time>" to "<To Date Time>"
    Then the request is rejected

    Examples:
        | From Date Time        | To Date Time          |
        | 2026-11-05 14:00 WIB  | 2026-11-05 13:00 WIB  |
        | 2026-11-05 13:00 WIB  | 2026-11-05 13:00 WIB  |
```


US-05:  As a borrower, I want to see my bookings so that I can track their status.

Business rules:
1. The list shows only bookings where I am the borrower.
2. The list is sorted by start time, newest first, and shows 10 bookings per page.
3. The list can be filtered by status.
4. Requesting the details of another user's booking returns "not found".
5. Times are shown in the time zone of the user's browser.
6. Rejected bookings show the approver's rejection note.

```gherkin
Scenario: My booking list shows only my bookings
    Given I have 3 bookings
    And another user has 2 bookings
    When I open my booking list
    Then I see exactly my 3 bookings
    And each booking shows its title, room, time, and status

Scenario: Booking list is sorted and paginated
    Given I have 12 bookings with different start times
    When I open my booking list
    Then I see the 10 bookings with the latest start times, newest first
    And when I open the next page, I see the remaining 2 bookings

Scenario: Booking list is empty
    Given I have no bookings
    When I open my booking list
    Then I see the message "You don't have any bookings yet"

Scenario: Filter my bookings by status
    Given I have 2 pending bookings and 3 approved bookings
    When I filter my booking list by status "Pending"
    Then I see exactly the 2 pending bookings
    And no bookings with other statuses are shown

Scenario: View the details of my booking
    Given I have an approved booking titled "Project A Meeting" for "Meeting Room A"
    When I open the details of that booking
    Then I see its title, room, purpose, time, and status "Approved"

Scenario: Cannot view another user's booking by its id
    Given another user has a booking
    When I request that booking's details directly by its id
    Then I receive a "not found" response

Scenario: Rejected booking shows the rejection note
    Given I have a booking for "Meeting Room A" that was rejected with the note "Room is reserved for an audit"
    When I open the details of that booking
    Then I see the status "Rejected" and the note "Room is reserved for an audit"

Scenario: Booking time is shown in my local time zone
    Given I have a booking stored from 2026-11-10 03:00 to 05:00 UTC
    And my browser time zone is Asia/Jakarta
    When I open the details of that booking
    Then I see the booking time as 2026-11-10 10:00–12:00 WIB
```


US-06:  As a borrower, I want to cancel my booking so that others can use the slot.

Business rules:
1. Only the borrower can cancel their booking.
   Cancelling another user's booking returns "not found".
2. Only bookings with status Pending or Approved can be cancelled
3. Cancellation is allowed only before start_at. At or after start_at, it is rejected
4. A status history entry is recorded for every cancellation.

```gherkin
Scenario: Cannot cancel another user's booking
    Given user B has a pending booking
    When I (user A) try to cancel it
    Then the request is rejected as not found
    And the booking status remains "Pending"

Scenario Outline: Cannot cancel a booking that is no longer active
    Given I have "<status>" booking
    When I cancel it
    Then the request is rejected
    And the booking status remains "<status>"

    Examples:
        | status    |
        | Rejected  |
        | Cancelled |

Scenario: Cancel a pending booking before it starts
    Given I have a pending booking from 2026-11-01 08:00 WIB to 11:45 WIB
    And the current time is 2026-11-01 07:59 WIB
    When I cancel the booking
    Then its status becomes "Cancelled"
    And a status history entry is recorded

Scenario Outline: Cancellation is allowed only before the booking starts
    Given I have an approved booking from 2026-11-01 08:00 WIB to 10:00 WIB
    And the current time is <now>
    When I cancel the booking
    Then <result>

    Examples:
        | now                  | result                                      |
        | 2026-11-01 07:59 WIB | its status becomes "Cancelled"              |
        | 2026-11-01 08:00 WIB | the request is rejected and status remains "Approved" |
        | 2026-11-01 09:00 WIB | the request is rejected and status remains "Approved" |
```


US-07:  As an approver, I want to approve or reject pending bookings so that restricted rooms are controlled.

Business rules:
1. Only users holding one of the room's approver roles can approve or reject its bookings.
   Others receive a "forbidden" error.
2. Only Pending bookings can be approved or rejected.
3. A pending booking cannot be approved after its start time, but it can still be rejected.
4. A rejection note is required.
5. Approvers may decide their own bookings, including when the approver role
   was granted after the booking was created.
6. If two approvers decide the same booking at the same time, only the first
   decision is saved. The other receives "This booking has already been decided"
7. Every decision records a status history entry with the approver's name.

```gherkin
Scenario: "APPROVER_GA" approve one pending booking room
    Given I have role "APPROVER_GA" 
    And another user has a pending booking for "Meeting Room A"
    And "APPROVER_GA" is approver of "Meeting Room A"
    When I approve the booking
    Then that booking status change from "Pending" to "Approved"
    And a status history entry is recorded

Scenario: "APPROVER_GA" Reject one pending booking room
    Given I have role "APPROVER_GA" 
    And another user has a pending booking for "Meeting Room A"
    And "APPROVER_GA" is approver of "Meeting Room A"
    When I reject the booking with the note "Room is under maintenance"
    Then that booking status change from "Pending" to "Rejected"
    And the note is save
    And a status history entry is recorded

Scenario: User without approver role try to approve booking
    Given I have no approver role for "Meeting Room A"
    And another user has a pending booking for "Meeting Room A"
    When I approve the booking
    Then I get "Forbidden" error
    And the booking status remains "Pending"

Scenario: "APPROVER_IT" isn't granted room "Meeting Room GA" approver
    Given "Meeting Room GA" can be approved by role "APPROVER_GA"
    And I have role "APPROVER_IT"
    And another user has a pending booking for "Meeting Room GA"
    When I approve booking for "Meeting Room GA"
    Then I get "Forbidden" error
    And the booking status remains "Pending"

Scenario: Reject pending booking in the middle of booking time
    Given another user has a pending booking for "Room A" from 2026-10-30 10:00 WIB to 12:00 WIB
    And "Room A" can be approved by role "APPROVER_IT"
    And I have role "APPROVER_IT"
    When I reject the booking for "Room A" with note on 2026-10-30 10:30 WIB
    Then the booking is "Rejected"

Scenario: Reject pending booking after the end of booking time
    Given another user has a pending booking for "Room A" from 2026-10-30 10:00 WIB to 12:00 WIB 
    And "Room A" can be approved by role "APPROVER_IT"
    And I have role "APPROVER_IT"
    When I reject the booking for "Room A" with note on 2026-10-30 13:00 WIB
    Then the booking is "Rejected"

Scenario: Role "APPROVER" given after the book is created
    Given "Meeting Room GA 01" requires approval by role "APPROVER_GA"
    And I have given role "APPROVER_GA" after booked "Meeting Room GA 01"
    When I approve my booking
    Then the booking status change from "Pending" to "Approved"

Scenario: Two approvers decide the same booking at the same time
    Given "Meeting Room A" can be approved by role "APPROVER_GA"
    And users "Approver 1" and "Approver 2" both have role "APPROVER_GA"
    And another user has a pending booking for "Meeting Room A"
    When "Approver 1" approves it and "Approver 2" rejects it at the same moment
    Then exactly one decision is saved
    And the other approver receives the error "This booking has already been decided"
    And exactly one status history entry is recorded

Scenario: Cannot approve a pending booking after it has started
    Given "Meeting Room A" can be approved by role "APPROVER_GA"
    And I have role "APPROVER_GA"
    And another user has a pending booking for "Meeting Room A" from 2026-10-30 10:00 WIB to 12:00 WIB
    And the current time is 2026-10-30 10:30 WIB
    When I approve the booking
    Then the request is rejected with a validation error
    And the booking status remains "Pending"

Scenario Outline: Cannot change booking status for a "Rejected" book
    Given I have role "APPROVER_GA"
    And another user has a rejected booking for "Meeting Room A"
    And "APPROVER_GA" has approval permission for "Meeting Room A"
    When I "<Action>" it
    Then the request is rejected
    And the booking status remains "Rejected"

    Examples:
        | Action    |
        | Approve   |
        | Reject    |

Scenario: Reject without a note is not allowed
    Given "Meeting Room A" can be approved by role "APPROVER_GA"
    And I have role "APPROVER_GA"
    And another user has a pending booking for "Meeting Room A"
    When I reject the booking without a note
    Then the request is rejected with the message "A rejection note is required"
    And the booking status remains "Pending"
```

