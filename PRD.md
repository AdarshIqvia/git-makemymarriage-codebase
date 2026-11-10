# Make My Marriage — Product Requirements Document

## 1. Document status

- **Status:** Draft for approval
- **Product:** Make My Marriage
- **Version:** MVP
- **Target:** Ready for real wedding use within three months
- **Scale:** More than 1,000 guests and multiple wedding events

No application implementation should begin until this PRD is approved.

## 2. Product summary

Make My Marriage is a web application for organizing one personal wedding. Admins and Managers create wedding events, manage household invitations, collect event-level RSVPs, track planning tasks, publish digital invitations and livestream links, and collect wedding photos.

Each household receives a private invitation link and personalized QR code. Guests do not create accounts. The household chooses its attendance count separately for each invited event and may optionally provide attendee names.

## 3. Goals

1. Provide one dashboard for events, guests, RSVPs, tasks, invitations, livestream links, and photos.
2. Support different invitation lists for different wedding events.
3. Make RSVP submission simple for households without requiring guest accounts.
4. Support more than 1,000 guests through import, search, filters, bulk actions, and exports.
5. Generate invitation material that family members can distribute manually.
6. Provide event photo albums that only households invited to each event can view, download, and upload to.

## 4. Non-goals for the MVP

- Multiple weddings or professional wedding-planner accounts
- Automated WhatsApp or email delivery
- WhatsApp Business API integration
- Guest accounts or passwords
- Dietary preferences or meal selection
- Guest dress codes
- Seating charts
- Budget tracking
- Vendor management
- Accommodation or transport planning
- Gift registry
- Automated reminders
- Guest-uploaded videos
- In-app livestream hosting
- Native mobile applications
- Payment collection

## 5. Users and permissions

### 5.1 Admin

An Admin has full access to the wedding.

- Manage wedding details and settings.
- Add, edit, disable, or remove Managers.
- Manage events, households, invitations, RSVPs, and tasks.
- Import and export guest information.
- Generate invitation links, cards, messages, and QR codes.
- Configure livestream links.
- Upload photos and moderate any uploaded photo.
- Access all dashboard summaries and reports.
- Permanently delete data where supported.

### 5.2 Manager

A Manager helps operate the wedding.

- Manage events, households, event assignments, RSVPs, and tasks.
- Import and export guest information.
- Generate invitation material.
- Configure livestream links.
- Upload photos and moderate any uploaded photo.
- Cannot manage Admin access or delete the wedding.

### 5.3 Guest household

Guests do not create accounts. A private, unguessable household link is their access credential.

- View the personalized invitation.
- View only the events assigned to the household.
- RSVP separately for each assigned event.
- Enter an unrestricted attendance count.
- Optionally enter attendee names.
- Update responses later through the same link.
- View an assigned event's livestream link.
- View, download, and upload photos only in the albums of assigned events, whatever the household's RSVP answer.
- Delete only photos uploaded through their own household link.

### 5.4 Public visitor

- View public wedding information.
- Cannot view, download, upload, or delete photos.
- Cannot view or change household invitations or RSVPs.
- Cannot access private livestream links.

## 6. Functional requirements

### 6.1 Planner authentication

- Admins and Managers sign in securely.
- Guest access does not use accounts.
- Manager access can be disabled by an Admin.
- Planner sessions expire securely.
- Authentication errors must not reveal sensitive account information.

### 6.2 Wedding setup

Admins can configure:

- Couple's names.
- Wedding date or date range.
- City and general location.
- Wedding story and welcome message.
- Contact information for guest assistance.
- Cover image or invitation artwork.
- Public wedding-page address.

No guest dress-code or meal-choice fields are included.

### 6.3 Event management

Admins and Managers can:

- Create, edit, reorder, and archive events.
- Use common event starters such as Haldi, Mehndi, Sangeet, Ceremony, and Reception.
- Create custom event types.
- Set event name, description, date, start/end time, venue, full address, and map link.
- Add or remove an external livestream URL.
- View the number of invited households and expected attendees per event.

Archived events remain available in historical RSVP and task records.

### 6.4 Household management

Each household record supports:

- Household or group label.
- Primary contact name.
- Phone number and email address.
- Optional household-member names.
- Internal Admin/Manager notes.
- Active or disabled status.
- A unique private invitation token.
- Invitation delivery status, including not sent and sent.

Admins and Managers can:

- Add and edit households manually.
- Import households from CSV or Excel.
- Preview and validate imports before saving.
- Detect likely duplicate phone numbers, email addresses, and household names.
- Search, filter, sort, and bulk-edit households.
- Export household and RSVP data.
- Disable or regenerate a compromised private link.

### 6.5 Event-level invitations

- A household can be assigned to any combination of events.
- Event assignments are independent for each household.
- The private invitation page displays only assigned events.
- Removing an event assignment removes it from the household's private page.
- Admins and Managers can assign or remove events individually or in bulk.
- The system records the latest invitation update time.

### 6.6 RSVP

For every assigned event, a household can:

- Select Yes, No, or Maybe.
- Enter the number of people attending.
- Enter any non-negative attendance number; no invitation maximum is enforced.
- Optionally enter attendee names.
- Submit without providing attendee names.
- Return later and update the response.

RSVP does not collect dietary requirements or meal choices.

Admins and Managers can:

- View totals by event and response status.
- View households that have not responded.
- Search and filter RSVP records.
- Manually update a response after contacting a household.
- View when a response was last updated.
- View each household's RSVP change history, including who made each change.
- View recent cancellations, meaning responses changed from Yes to No.
- Export event-level headcount reports.
- Review unusually high submitted counts without blocking submission.

Headcount is calculated from submitted attendance counts, not from named household members.

### 6.7 Digital invitation and manual distribution

The system generates but does not automatically send invitations.

- Personalized digital invitation page.
- Personalized private household URL.
- Personalized QR code opening the same private household page.
- Downloadable personalized invitation card.
- WhatsApp-ready message containing the private link.
- Email-ready invitation content containing the private link.
- Copy and download actions.
- Admin-only sending sheet listing each household's name, phone number, private link, and a one-tap WhatsApp link, so family members can split the sending.
- Sent/not-sent tracking managed manually by Admins and Managers.

Because a personalized QR code grants RSVP access, Admins and Managers must be warned not to publish it publicly. A compromised link can be regenerated.

### 6.8 Public wedding page

The public page may display:

- Couple's names.
- Wedding story and welcome message.
- General wedding date and city.
- Public contact information.

It must not display:

- Household or guest names.
- RSVP information.
- Private event assignments.
- Internal notes.
- Private livestream links.
- Photos.

### 6.9 Task management

Admins and Managers can:

- Create, edit, complete, and delete tasks.
- Add a title, description, due date, and priority.
- Assign a task to an Admin or Manager.
- Optionally associate a task with an event.
- Set status to To Do, In Progress, or Done.
- Search and filter by status, assignee, event, priority, and due date.
- View overdue tasks.

Automated notifications and recurring tasks are outside the MVP.

### 6.10 External livestream links

- Each event can contain one external livestream URL.
- Supported destinations may include YouTube, Zoom, or Google Meet.
- The application does not broadcast, record, or host video.
- A Watch Live action appears only when a link is configured.
- Only households assigned to the event can see its livestream link.
- Admins and Managers can change or remove the link.

### 6.11 Event photo albums

- Photos are organized into one album per event.
- A household can view, download, and upload photos only in albums of events it is invited to, regardless of its RSVP answer.
- Admins and Managers can view, upload to, and moderate every album.
- Public visitors and households not invited to an event cannot see that event's album.
- A newly uploaded photo appears in its album after successful processing.
- A guest can delete only photos uploaded through the same household link.
- One guest cannot edit or delete another guest's photos.
- Admins and Managers can hide or delete any photo.
- Hidden photos are not visible or downloadable by guests.
- The album supports thumbnails and full-photo viewing.
- Downloads provide a high-quality optimized copy; original camera files are not kept.
- Photos are resized on the guest's device before upload to save mobile data and handle iPhone photo formats.
- Uploads accept supported image formats only; videos are rejected.
- File type and size are validated.
- Embedded GPS/location metadata is removed before publication.
- The upload screen states that the photo will be visible to everyone invited to that event.
- The system records upload time, event album, and source household for moderation.

Album pages and photo links are excluded from search-engine indexing.

### 6.12 Dashboard

The dashboard displays:

- Upcoming events.
- Total active households.
- RSVP totals by response and event.
- Expected attendance per event.
- Households awaiting responses.
- Recent RSVP cancellations.
- Unusually high attendance submissions.
- Invitations marked sent and not sent.
- Tasks by status and overdue tasks.
- Recent photo uploads.

Quick actions include:

- Add an event.
- Add or import households.
- Create a task.
- Generate or copy invitation material.
- Open RSVP reports.
- Open gallery moderation.

## 7. Primary user flows

### 7.1 Initial setup

1. Admin signs in.
2. Admin enters wedding details.
3. Admin creates wedding events.
4. Admin adds Managers if required.

### 7.2 Guest import and event assignment

1. Admin or Manager uploads a CSV/Excel file.
2. The system validates rows and highlights possible duplicates.
3. The user confirms the import.
4. Households are assigned to selected events individually or in bulk.
5. The system generates private links and personalized QR codes.

### 7.3 Manual invitation delivery

1. Admin or Manager opens a household.
2. The user copies a WhatsApp/email message or downloads its invitation card.
3. A family member sends it outside the application.
4. The user marks the invitation as sent.

### 7.4 Household RSVP

1. Household opens its private link or QR code.
2. The personalized invitation displays assigned events.
3. The household chooses Yes, No, or Maybe for each event.
4. For attending events, it enters the attendance count.
5. It may optionally enter attendee names.
6. The household submits and sees confirmation.
7. It may return through the same link to update the response.

### 7.5 Photo upload and viewing

1. An invited household opens its private link.
2. It selects the album of an event it is invited to and chooses photos.
3. The application explains that the photos will be visible to everyone invited to that event.
4. Photos are resized on the device, then validated, stripped of location metadata, processed, and published.
5. Households invited to that event can view and download the published photos.
6. The uploader can delete its own photos; Admins and Managers can moderate all photos.

## 8. Data and reporting

The system must support exports for:

- Household contact list.
- Household-to-event assignments.
- RSVP status by household and event.
- Submitted attendance counts by event.
- Optional attendee names.
- Invitations marked sent/not sent.
- Task list and status.

Exports must not expose private invitation tokens unless an Admin explicitly requests an invitation-link export.

## 9. Non-functional requirements

### 9.1 Scale and performance

- Support more than 1,000 guests and at least 10 wedding events.
- Household lists, RSVP reports, and gallery pages use pagination or incremental loading.
- Photo uploads go directly to cloud object storage.
- Images use compressed thumbnails for gallery browsing.
- Common invitation and RSVP pages should load well on typical mobile networks.

### 9.2 Security and privacy

- Private invitation tokens must be long, random, and unguessable.
- Tokens must not appear in analytics payloads or application logs.
- RSVP pages must not be indexed by search engines.
- Planner passwords are securely hashed.
- Planner-only actions require server-side authorization.
- Upload endpoints enforce rate limits and file validation.
- Photo links are private, expire, and must not expose household data.
- Sensitive changes and moderation actions are auditable.
- Data is backed up and recoverable.

### 9.3 Usability and accessibility

- Guest flows are mobile-first.
- RSVP must work without registration.
- Forms have clear validation and confirmation messages.
- Core screens support keyboard use and readable color contrast.
- Date, time, and venue information is presented consistently.

### 9.4 Reliability

- An RSVP submission is saved atomically and must not be duplicated by repeated taps.
- Upload failures can be retried without duplicating completed photos.
- Regenerating a private invitation token invalidates the previous link.
- A small pilot group should validate the complete flow before links are distributed to all guests.

## 10. MVP acceptance criteria

The MVP is ready when:

1. An Admin can configure one wedding and at least 10 events.
2. An Admin can add a Manager with restricted permissions.
3. More than 1,000 guests can be imported as households without manually entering every record.
4. Households can be assigned different combinations of events.
5. Each household receives a distinct private URL and personalized QR code.
6. A household sees only its assigned events and can RSVP separately to each one.
7. A household can submit an unrestricted attendance count and omit attendee names.
8. Dashboard and exports calculate event headcounts correctly.
9. Invitations can be prepared for manual WhatsApp, email, and QR/card distribution.
10. A configured livestream link is visible only to households assigned to that event.
11. An invited household can view, download, and upload photos in the albums of its invited events only.
12. A household cannot open the album of an event it was not invited to, and public visitors cannot open any album.
13. Guests cannot delete another household's photos.
14. Admins and Managers can hide or delete any photo.
15. Guests are not offered dress-code, dietary, or meal-choice fields.
16. Core invitation, RSVP, and gallery flows work on mobile devices.

## 11. Launch approach

1. Configure the real wedding and events.
2. Import a small internal test list.
3. Test private links, QR codes, event visibility, RSVP counts, livestream privacy, and photo permissions.
4. Run a pilot with 10–20 trusted households.
5. Correct data or usability issues found in the pilot.
6. Import and verify the complete guest list.
7. Generate invitation material and begin manual distribution.

## 12. Approval

Approval of this PRD confirms the MVP product scope. Technical architecture, UI design, implementation sequencing, and delivery estimates should be finalized only after product approval.
