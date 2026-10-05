# Monuments Inventory, Booking & QR Verification System
## Software Development Requirements Document

This document defines a system for managing monument inventory, publishing available visiting slots, accepting bookings, generating QR tickets, verifying tickets at entry counters, and providing operational reports.

**Terminology:** “Monument” means a bookable attraction, heritage site, museum, or similar location. “Seats” means visitor capacity, even where physical seats are not provided.

---

## 1. Project Objectives

The system must enable administrators to:

1. Add and manage monuments.
2. Configure visiting time slots.
3. Set limited or unlimited visitor capacity.
4. Block dates or individual slots.
5. Automatically publish bookable listings from inventory.
6. Accept visitor bookings and generate QR tickets.
7. Verify QR tickets at entry counters.
8. Monitor bookings, attendance, capacity, and revenue through dashboards and reports.

### Proposed MVP assumptions

These assumptions should be confirmed before implementation:

- Visitors book one monument and one time slot per booking.
- Each booking can include multiple visitors.
- Capacity is measured in visitors, not bookings.
- QR verification requires an internet connection.
- The MVP supports one QR ticket per booking, admitting the entire group together.
- Payments are optional: the system can support free entry and paid entry.
- Each monument has a configured local time zone.

---

## 2. User Roles & Permissions

| Role | Main permissions |
|---|---|
| System Administrator | Manage all monuments, users, settings, bookings, and reports |
| Monument Manager | Manage inventory, availability, and reports for assigned monuments |
| Booking Counter Operator | Create walk-in bookings, search bookings, and issue tickets |
| Entry Verification Operator | Scan QR tickets and view entry eligibility for assigned monuments |
| Visitor | Browse monuments, book tickets, retrieve tickets, and view their bookings |
| Auditor / Report Viewer | View permitted reports and audit records without editing |

**Access requirements:**
- Staff must authenticate before accessing administrative or verification features.
- Monument-specific permissions must be enforced on the server.
- Ticket verification staff should see only the visitor information necessary for entry.
- Changes to inventory, blocked dates, bookings, and check-ins must be audited.

---

## 3. Functional Requirements

## 3.1 Monument Inventory Management

Administrators can create, edit, publish, unpublish, and archive monuments.

### Monument fields

| Field | Requirement |
|---|---|
| Monument ID | System-generated unique identifier |
| Name | Required |
| Description | Visitor-facing information |
| Location | Address, city, state, country |
| Time zone | Required for booking and verification rules |
| Images | Main image and optional gallery |
| Opening schedule | Operating days and hours |
| Ticket categories | Adult, child, student, or other configured categories |
| Ticket prices | Required for paid entry |
| Capacity mode | Limited or unlimited |
| Booking window | How far in advance visitors may book |
| Booking cutoff | Minimum time before a slot starts that booking remains permitted |
| Maximum visitors per booking | Configurable |
| Entry instructions | Visitor-facing guidance |
| Status | Draft, published, inactive, archived |

### Inventory rules

- Only published monuments appear in public listings.
- Existing booking records must remain accessible after a monument is unpublished or archived.
- A monument with bookings must not be permanently deleted.
- Changes to prices must not alter the prices recorded on existing bookings.
- Capacity must not be reduced below confirmed visitors plus active reservation holds.
- Changes to operating hours or slot definitions must identify affected bookings before being applied.

---

## 3.2 Time Slot Management

Each monument can have multiple visiting slots.

### Example

| Time slot | Capacity mode | Visitor limit |
|---|---|---:|
| 09:00–10:00 | Limited | 100 |
| 10:00–11:00 | Limited | 150 |
| 11:00–12:00 | Unlimited | Not applicable |

### Required features

- Create recurring schedules by weekday.
- Define slot start and end times.
- Set capacity for each slot.
- Create date-specific overrides.
- Disable individual slots.
- Prevent duplicate slots and overlapping slots by default.
- Support configurable booking cutoff and entry grace periods.

### Capacity calculation

For a limited-capacity slot:

**Available capacity = Slot capacity − Confirmed visitors − Visitors in active reservation holds**

Cancelled bookings and expired holds do not consume capacity. Completed visits remain counted against the original slot allocation; check-in does not reopen that capacity.

For unlimited-capacity slots:
- No slot capacity limit applies.
- Booking quantity limits, booking cutoffs, and blocked-date rules still apply.

---

## 3.3 Blocked Dates & Slots

Managers can block:

- A single date.
- A date range.
- A particular slot on a date.
- Recurring closure days, such as every Monday.

### Block record fields

| Field | Description |
|---|---|
| Monument | Affected monument |
| Date or date range | Closure period |
| Slot | Optional; otherwise applies to the whole day |
| Reason | Maintenance, holiday, private event, emergency, etc. |
| Public message | Optional visitor-facing notice |
| Created by / created at | Audit information |

### Business rules

- Blocked dates and slots must not accept new bookings.
- A blocked date overrides the normal operating schedule.
- If bookings already exist, show the affected booking count before saving the block.
- Do not silently cancel existing bookings.
- Require an explicit decision to retain existing bookings or cancel and notify affected visitors.
- Where payment exists, track refunds separately from cancellations.
- Emergency closures must allow authorized staff to record the operational decision and reason.

---

## 3.4 Public Listings & Availability

Public listings must be generated from published monument inventory.

### Listing information

- Monument name and image.
- Location and description.
- Opening hours.
- Ticket price or “Free entry.”
- Available dates and slots.
- Remaining visitor capacity for limited slots.
- “Available” for unlimited slots.
- Entry instructions and closure notices.

### Availability states

| State | Meaning |
|---|---|
| Available | Booking is permitted |
| Sold out | Limited capacity has been exhausted |
| Closed | Date or slot is blocked |
| Booking closed | Booking cutoff has passed |
| Unavailable | Monument is unpublished or outside its booking window |

**Important:** Availability must be validated again on the server when reserving capacity and confirming a booking. The public listing is not the final authority.

---

## 3.5 Booking System

### Booking workflow

1. Visitor selects a monument.
2. Visitor selects a date and time slot.
3. Visitor selects ticket categories and quantities.
4. System validates availability and calculates the price.
5. Visitor provides contact details and accepts applicable terms.
6. For paid bookings, the system places a temporary capacity hold and initiates payment.
7. System confirms the booking after successful validation and, where required, verified payment.
8. System creates the QR ticket.
9. Visitor receives confirmation and can download or retrieve the ticket.

Free bookings may be confirmed immediately after an atomic capacity check.

### Booking information

| Field | Description |
|---|---|
| Booking reference | Unique public reference |
| Monument | Booked location |
| Visit date and slot | Booked admission period |
| Visitor name | Primary visitor or group contact |
| Email / mobile | Required according to configured delivery method |
| Ticket categories and quantities | Visitor breakdown |
| Total visitors | Capacity consumed |
| Price breakdown | Snapshot of prices, taxes, and fees |
| Booking status | Booking lifecycle state |
| Payment status | Separate payment lifecycle state |
| Booking channel | Website, counter, or authorized integration |
| Created at / created by | Audit information |

Collect individual visitor details only if operational or regulatory requirements justify them.

### Booking statuses

- Pending.
- Confirmed.
- Cancelled.
- Expired.

Attendance is tracked separately as:
- Not checked in.
- Checked in.

“No-show” is a reporting classification for a confirmed booking whose entry window has ended without a successful check-in.

### Capacity and payment safeguards

- Use database transactions and concurrency controls to prevent overbooking.
- Reservation holds must expire after a configurable interval.
- Payment callbacks must be authenticated and processed idempotently.
- Repeated requests must not create duplicate bookings, payments, or tickets.
- Payment received after a hold expires must not automatically confirm an oversold slot; apply a defined recovery or refund process.
- Cancellation releases allocated capacity only when the cancellation is successfully committed.
- Payment success must never be trusted solely from a browser redirect.

---

## 3.6 QR Ticket Generation

Each confirmed booking receives a QR ticket.

### Ticket display

- Booking reference.
- Monument name.
- Visit date and slot.
- Ticket categories and visitor count.
- QR code.
- Entry instructions.
- Relevant cancellation or entry conditions.

### QR security requirements

- Use a cryptographically random, opaque token or a securely signed token.
- Do not encode visitor names, mobile numbers, or other personal information directly in the QR code.
- Do not use the booking reference alone as the QR credential.
- QR possession must not grant access to visitor account details.
- Cancelled tickets must become invalid immediately.
- If a ticket is reissued, revoke the previous token.
- Verification must check current server-side booking and entry status.

**MVP decision:** A booking QR admits the whole group once. If visitors must enter separately, use individual visitor tickets or an explicitly designed partial check-in feature in a later phase.

---

## 3.7 Counter QR Verification

Verification operators access a mobile-friendly scanning screen using a phone, tablet, or compatible scanner.

### Verification workflow

1. Operator signs in.
2. Operator selects an authorized monument or entry gate.
3. Operator scans a QR code.
4. Server validates:
   - QR token validity.
   - Booking confirmation status.
   - Correct monument.
   - Allowed visit date and entry window.
   - Cancellation or revocation status.
   - Previous check-in status.
5. Screen displays ticket details and eligibility.
6. Operator confirms admission.
7. Server atomically records check-in.
8. Screen shows final admission success.

Scanning for preview must not itself consume the ticket. The final admission action must revalidate eligibility.

### Verification outcomes

| Result | Operator message |
|---|---|
| Eligible | Ready to admit; display visitor count |
| Admission recorded | Entry successfully recorded |
| Already checked in | Display previous entry time |
| Invalid QR | Ticket not recognized |
| Cancelled / revoked | Entry not permitted |
| Wrong monument | Ticket belongs to another monument |
| Wrong date / outside entry window | Entry not permitted under current rules |
| Not confirmed | Booking is not eligible for entry |
| Network/server failure | Verification unavailable; do not show a success result |

### Duplicate-entry prevention

- Check-in must be atomic.
- If two counters admit the same ticket simultaneously, only one operation may succeed.
- Rejected scans must not modify booking or attendance state.
- Repeated requests caused by network retries must not create duplicate check-ins.
- Supervisor overrides, if permitted, require a reason and an audit record.

### Check-in audit information

- Booking and ticket identifiers.
- Monument and gate/counter.
- Operator.
- Server timestamp.
- Admission outcome.
- Override reason, where applicable.

**Offline verification is outside the MVP:** it requires a separate design for synchronization and duplicate-use risk.

---

## 4. Dashboard & Reports

## 4.1 Dashboard Summary

Dashboard filters:
- Date range.
- Monument.
- Time slot.
- Booking channel.
- Ticket category.

Summary cards:
- Confirmed bookings.
- Booked visitors.
- Checked-in visitors.
- Visitors awaiting entry.
- Cancelled bookings.
- No-shows.
- Remaining capacity.
- Gross payments and refunds, if payments are enabled.

Charts:
- Bookings by day.
- Visitors by monument.
- Slot occupancy.
- Attendance versus bookings.
- Revenue trends.
- Website versus counter bookings.

Show capacity percentages only for limited-capacity slots.

---

## 4.2 Required Reports

| Report | Main contents |
|---|---|
| Booking report | Reference, monument, date, slot, quantities, status, channel |
| Visitor attendance report | Booked visitors, admitted visitors, check-in time |
| Slot occupancy report | Capacity, booked visitors, remaining capacity, utilization |
| No-show report | Confirmed bookings not admitted after the entry window closes |
| Revenue report | Payments received, refunds, net collections, payment method |
| Cancellation report | Cancellation date, reason, actor, refund state |
| Counter activity report | Admissions and rejected scans by operator and counter |
| Blocked-date report | Closure dates, affected slots, reasons, affected bookings |
| Audit report | Sensitive administrative and operational changes |

### Report requirements

- Export authorized reports to CSV and Excel; PDF can be added if required.
- Respect monument-level access restrictions.
- Use the monument’s time zone for operational dates.
- Include generation time and applied filters.
- Restrict and audit exports containing personal information.
- Define metrics consistently: bookings and visitor counts must not be interchangeable.
- Revenue reports should distinguish payment date from visit date.

---

## 5. Suggested System Architecture

| Component | Responsibility |
|---|---|
| Visitor web application | Listings, availability, booking, ticket retrieval |
| Admin application | Inventory, schedules, closures, users, reports |
| Counter interface | Walk-in booking and ticket issuance |
| Verification interface | QR scanning and admission |
| Backend API | Business rules, authentication, capacity, ticket validation |
| Relational database | Booking, inventory, payment, and check-in records |
| Background worker | Notifications, hold expiry, exports, retries |
| Object storage | Monument images and generated documents |
| Payment provider | Optional online payment processing |
| Notification provider | Email and optional SMS |

A practical implementation could use **React or Next.js**, **Node.js, Django, or Laravel**, and **PostgreSQL**. The final stack should follow the development team’s experience and hosting requirements.

The relational database should remain the authoritative source for capacity and check-in state.

---

## 6. Core Data Entities

| Entity | Purpose |
|---|---|
| User / Role / Monument Assignment | Authentication and permissions |
| Monument | Location and public inventory |
| Ticket Category / Price | Visitor categories and pricing rules |
| Schedule / Slot Template | Recurring operating schedule |
| Slot Instance | A dated slot with effective capacity and availability |
| Blocked Period | Full-day or slot-specific closure |
| Reservation Hold | Temporary capacity reservation |
| Booking | Visitor reservation and lifecycle |
| Booking Item | Quantity and price snapshot per category |
| Payment / Refund | Financial transactions |
| Ticket | QR credential and revocation state |
| Check-in | Successful admission |
| Verification Attempt | Accepted or rejected verification activity |
| Notification | Delivery status and retry information |
| Audit Log | Administrative and sensitive operational changes |

### Database integrity requirements

- Unique booking references and QR token identifiers.
- A uniqueness rule preventing multiple successful check-ins for a booking in the MVP.
- Valid foreign-key relationships.
- Nonnegative quantities and amounts.
- Consistent currency handling using decimal values or integer minor units, not floating-point money.
- Indexes for monument/date/slot queries and ticket lookup.
- Store timestamps consistently and retain monument time zones for display and operational rules.

---

## 7. API Functional Groups

| Group | Operations |
|---|---|
| Authentication | Staff login, logout, session management, password recovery |
| Monuments | Create, update, publish, archive, list |
| Schedules | Manage recurring slots and date overrides |
| Closures | Create and remove blocked periods |
| Availability | Retrieve valid dates, slots, and remaining capacity |
| Bookings | Reserve, confirm, retrieve, cancel |
| Counter bookings | Create and issue walk-in tickets |
| Payments | Start payment, receive verified callback, track refund |
| Tickets | Retrieve, download, revoke, reissue |
| Verification | Preview eligibility and commit admission |
| Reports | Filter, summarize, export |
| Audit | Retrieve authorized activity records |

All state-changing APIs must validate authorization and business rules on the server.

---

## 8. Nonfunctional Requirements

### Security
- HTTPS throughout.
- Secure staff authentication; MFA recommended for privileged users.
- Server-side permission checks.
- Rate limiting for public booking and ticket endpoints.
- Input validation and secure file-upload handling.
- Secrets stored outside source code.
- Minimal collection and controlled retention of personal data.
- Sensitive tokens and payment details excluded from logs.

### Reliability
- Automated backups and tested restoration procedures.
- Monitoring for booking failures, payment failures, and verification outages.
- Retryable background jobs with duplicate-processing protection.
- Ticket retrieval must remain available when notification delivery fails.
- A documented entry-counter outage procedure.

### Performance
Initial targets, to be validated through load testing:
- QR eligibility checks: typically within two seconds under expected load.
- Public availability requests: typically within two seconds.
- Large exports processed asynchronously.
- Capacity and duplicate-entry guarantees remain correct under peak concurrent use.

### Usability
- Responsive visitor and staff interfaces.
- Clear visual and textual verification outcomes.
- Manual ticket lookup for damaged QR codes, under staff authorization.
- Accessible forms and keyboard navigation.
- Clear local date, time, and currency display.

---

## 9. Key Acceptance Criteria

| Scenario | Expected result |
|---|---|
| Administrator publishes a monument with slots | Monument and valid availability appear publicly |
| A slot has capacity 100 and 95 allocated visitors | A request for six visitors is rejected |
| Two users compete for the last available capacity | Total confirmed visitors and active holds never exceed capacity |
| A slot is unlimited | Bookings are accepted subject to other booking rules |
| A date is blocked | New bookings for that date are rejected |
| A block affects existing bookings | System requires an explicit handling decision |
| A booking is confirmed | A retrievable QR ticket is created |
| Payment callback is repeated | No duplicate confirmation or ticket is created |
| A valid ticket is admitted | One check-in record is created |
| The same ticket is admitted at two counters | Only one admission succeeds |
| A cancelled ticket is scanned | Admission is rejected |
| Ticket is scanned at the wrong monument | Admission is rejected |
| Ticket is outside its entry window | Admission is rejected unless an authorized override applies |
| A report is filtered by monument and date | Results and totals match the selected scope |
| Unauthorized staff request another monument’s data | Access is denied |

---

## 10. Development Phases

### Phase 1 — Inventory & Administration
- Authentication and role-based permissions.
- Monument inventory.
- Ticket categories and prices.
- Recurring schedules and dated slots.
- Capacity configuration.
- Blocked dates and slots.

### Phase 2 — Listings & Booking
- Public listings.
- Availability checks.
- Visitor and counter bookings.
- Reservation holds.
- Optional payment integration.
- Booking confirmation and QR tickets.

### Phase 3 — Entry Verification
- QR scanner interface.
- Eligibility checks.
- Atomic admission.
- Duplicate-entry prevention.
- Verification and audit records.

### Phase 4 — Reporting & Release
- Dashboards and reports.
- Exports.
- Security and concurrency testing.
- Backup and monitoring setup.
- User acceptance testing.
- Staff training and deployment.

---

## 11. Expected Deliverables

1. Approved requirements and business rules.
2. Visitor, admin, and counter UI designs.
3. Database schema and migrations.
4. Backend API and documentation.
5. Visitor booking application.
6. Inventory administration application.
7. QR verification interface.
8. Dashboard and reporting module.
9. Automated tests and acceptance-test results.
10. Deployment, backup, and operational documentation.
11. Staff user guide.

---

## 12. Decisions Required Before Development

1. **Payments:** Free bookings, paid bookings, or both?
2. **Ticket structure:** One group QR or individual visitor QR codes?
3. **Entry rules:** Exact-slot entry, configurable grace period, or all-day validity?
4. **Visitor categories:** Which categories and prices are required?
5. **Cancellation policy:** Who can cancel, and when are refunds allowed?
6. **Notifications:** Email, SMS, or both?
7. **Counter bookings:** Do they share the same capacity pool as online bookings? Recommended: yes.
8. **Closure handling:** What happens to bookings affected by a new closure?
9. **Verification connectivity:** Is online-only acceptable at every gate?
10. **Scale:** How many monuments, gates, daily visitors, and peak simultaneous bookings?
11. **Languages and currency:** Which are required at launch?
12. **Data retention:** How long must visitor, financial, and audit records be retained?

**Recommended MVP scope:** inventory, limited/unlimited slots, blocked dates, public listings, online and counter booking, group QR tickets, online entry verification, and core booking/attendance reports. Add payment and revenue reporting if paid admission is required at launch.
