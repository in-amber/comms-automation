# Employer Communications Manager

## Product / Behavior Spec

## 1. Purpose

This project is a Claude-powered job-search operations assistant that helps manage communications with potential employers after applications have already been submitted.

Its core job is to maintain a unified picture of:

- submitted job applications and their statuses
- job-search-relevant contacts
- ongoing email threads
- later, LinkedIn conversations and calendar-linked follow-up actions

The system should reduce missed follow-ups, improve tracking hygiene, and help maintain timely, professional communications with recruiters and hiring managers.

For the first version, the system should be conservative. It may classify emails, update records automatically when confidence is high, and prepare Gmail drafts for user review. It must not send emails automatically in v1.

---

## 2. Primary goals

The product should:

1. Pull application records from a Google Sheet and sync them into a local database.
2. Monitor Gmail for job-related emails connected to those applications or contacts.
3. Automatically classify relevant incoming emails when confidence is high.
4. Automatically update application status when the email is routine and unambiguous.
5. Track job-search contacts even if they are not yet tied to a specific application.
6. Track email threads at the thread level and assign machine-actionable states.
7. Prepare Gmail drafts for routine responses or follow-ups, but require manual send in v1.
8. Escalate ambiguous, sensitive, or unusual cases rather than acting.
9. Provide a simple human-readable interface, initially a CLI, to inspect records and see what needs attention.
10. Preserve an audit trail of important decisions and actions.

---

## 3. Non-goals for v1

The first version should **not** attempt to do the following:

- automatically send emails
- autonomously schedule calls based on calendar availability
- fully manage human recruiter conversations end to end
- track interviews through all stages
- automate LinkedIn outreach or LinkedIn browser actions
- reconstruct full historical inbox state from old email
- infer uncertain links between contacts and applications without user input

These are later roadmap items.

---

## 4. Product principles

### 4.1 Conservative autonomy
When uncertain, the system should do nothing and escalate.

### 4.2 Human review before outbound communication
In v1, every outbound email draft must be reviewed and sent manually by the user from Gmail drafts.

### 4.3 Automatic status updates are allowed when high-confidence
Routine, unambiguous events such as confirmations and rejections may update records automatically.

### 4.4 The database is the full picture
The Google Sheet is an important upstream source, but it is not the full system of record. The local database must store the broader job-search relationship graph, including contacts and threads that do not yet map to an application.

### 4.5 Minimal duplication where possible
For email data, the system should prefer storing identifiers and structured metadata, and fetch full thread contents from Gmail when needed rather than duplicating all message text into the database.

### 4.6 Auditability
Important actions and decisions should be recorded in an event log, including why a status changed.

---

## 5. Scope by version

## v1
Focus on applications and Gmail only.

### v1 capabilities
- import and sync applications from Google Sheets
- maintain a local database of applications, contacts, and relevant email threads
- monitor Gmail on a schedule
- identify relevant incoming emails
- classify incoming emails into routine job-search categories
- update application statuses automatically when high-confidence
- create Gmail drafts for routine follow-ups or acknowledgments
- surface applications needing attention, stalled threads, and prepared drafts
- provide CLI visibility into current records and recent changes

### v1 excludes
- LinkedIn automation
- calendar-based interview workflows
- autonomous sending
- interview lifecycle management beyond detecting that a conversation has started

## v2
Add richer contact and email workflow support, and basic Google Calendar awareness.

### likely v2 capabilities
- better handling of recruiter / hiring-manager conversations
- contact-centric follow-up reminders
- Google Calendar integration for meetings
- calendar-triggered reminders and draft suggestions
- more nuanced thread state transitions
- optional creation / update of calendar events inside the agent’s calendar profile

## v3+
Add LinkedIn and more advanced interview workflow automation.

### likely v3+ capabilities
- LinkedIn contact discovery
- LinkedIn message drafting
- possibly browser automation, if safe and reliable
- separate LinkedIn conversation state machine
- interview requested / scheduled / completed lifecycle tracking
- end-of-day post-interview follow-up drafting
- more selective autonomous sending for safe categories

---

## 6. Core entities

The system should manage the following record types.

## 6.1 JobApplication
Represents one submitted application.

Should include:
- job title
- company name
- source sheet identifiers / row reference
- date applied
- current application status
- linked contacts
- linked email threads
- optional notes / tags
- timestamps for created, updated, last status change

### v1 application statuses
- `applied_unconfirmed`
- `applied_confirmed`
- `action_required`
- `response_received`

### status meanings
- **applied_unconfirmed**: application exists in the sheet / DB, but no confirmation or relevant employer response has been detected yet
- **applied_confirmed**: an automated confirmation or other clear confirmation of receipt was received
- **action_required**: an automated message indicates the user must complete a next step, such as registration, assessment, form completion, or similar
- **response_received**: a human or otherwise meaningful conversation has begun; the application is no longer just sitting in a passive submitted state

## 6.2 Contact
Represents a job-search-relevant person.

A contact may exist even if not linked to any current application.

Should include:
- full name
- company or companies
- email address(es)
- LinkedIn URL or identifier later
- notes
- linked applications
- linked email threads
- communication relevance flag
- manually assigned relationship notes if needed

A single contact may link to multiple companies and multiple applications.

## 6.3 EmailThread
Represents a job-search-relevant Gmail thread.

This should be tracked at the **thread level** in v1.

Should include:
- Gmail thread ID
- subject
- participants
- linked contact(s)
- linked application(s), if any
- thread state
- last inbound message timestamp
- last outbound message timestamp
- follow-up due date if applicable
- escalation flag
- summary / extracted metadata as needed

### v1 thread states
- `awaiting_response`
- `followup_due`
- `escalated`
- `send_required`
- `no_action`
- `closed`

### state meanings
- **awaiting_response**: the last meaningful action is waiting on the employer
- **followup_due**: the thread should receive a follow-up
- **escalated**: the thread requires user attention because it is ambiguous, sensitive, or outside automation bounds
- **send_required**: a draft should be created or has been created and awaits user send
- **no_action**: the thread is relevant but does not currently require action
- **closed**: the thread is no longer active for follow-up purposes

## 6.4 EventLog
Immutable audit-style history of important system decisions and actions.

Examples:
- application imported from sheet
- email classified as confirmation
- application status changed from `applied_unconfirmed` to `applied_confirmed`
- thread marked `followup_due`
- Gmail draft created
- thread escalated due to ambiguity
- user manually linked a thread to an application

Each event should include:
- timestamp
- event type
- affected entity or entities
- reason / explanation
- evidence reference when applicable

## 6.5 Company
Optional for v1.

This is not required to ship v1. It may be added later if it becomes useful to group:
- multiple applications at the same company
- multiple contacts at the same company
- outreach limits or anti-spam rules later

For now, company may remain a field on applications and contacts rather than a first-class record.

## 6.6 CalendarEvent
Not required for v1. Planned for v2.

## 6.7 LinkedInConversation
Not required for v1. Planned for v3+.

---

## 7. Sources of truth and synchronization

## 7.1 Google Sheet
The Google Sheet is a partial upstream source that contains completed applications.

Behavior:
- the system should read from the Sheet using the Google Sheets API
- the user and other scripts may continue updating the Sheet
- the database must pull and reconcile those updates
- the Sheet is not sufficient to represent the full job-search state, because contacts and conversations may exist without a linked application

## 7.2 Local database
The local database is the broader operational source of truth for the product.

It should store:
- applications from the Sheet
- contacts not present in the Sheet
- relevant email threads
- current states for threads and applications
- event log records
- workflow-relevant metadata

## 7.3 Reconciliation
The system should support:
- scheduled Gmail monitoring three times per day
- weekly reconciliation between the Sheet and database
- manual reconciliation triggers

If the Sheet and DB disagree in a way that is not clearly resolvable, the system should escalate for human review.

---

## 8. Gmail monitoring behavior

## 8.1 Scope
The system should monitor Gmail for **job-related emails only**.

It may need to inspect incoming mail broadly enough to decide relevance, but should avoid retaining irrelevant content in the database.

## 8.2 Monitoring cadence
Default schedule:
- three runs per day

The exact times can be configured later.

## 8.3 Relevance filtering
Incoming emails should be evaluated for whether they are job-search relevant.

Likely relevance signals include:
- known company names from applications
- known contact addresses
- recruiter / talent / hiring language
- apply / application / interview / assessment / schedule / next steps language
- threading with already-known relevant threads

If uncertain whether an email is relevant, the system may mark it for review rather than ignoring it or storing it aggressively.

## 8.4 Storage strategy
Prefer to store:
- Gmail thread ID
- message IDs if useful
- participants
- timestamps
- classification
- state
- linked entities
- structured metadata
- generated summaries when helpful

Prefer **not** to duplicate full raw message bodies unless needed for caching, audit, or reasoning.

## 8.5 Initial history
The system does not need to reconstruct historical inbox state in v1. It should primarily process new mail going forward.

---

## 9. Email classification behavior

For v1, the system should classify relevant incoming emails into behaviorally meaningful categories.

Suggested v1 email classes:
- application confirmation
- automated action required
- rejection
- meaningful response / conversation initiated
- irrelevant to job search
- uncertain

### expected mapping examples
- **application confirmation**  
  application status may move to `applied_confirmed`

- **automated action required**  
  application status moves to `action_required`; thread may move to `escalated` or `send_required` depending on what is needed

- **rejection**  
  this is a meaningful status change and should surface in alerts; thread may move to `closed`

- **meaningful response / conversation initiated**  
  application status moves to `response_received`; thread usually becomes `escalated` or `no_action` depending on message type

- **irrelevant**  
  no DB action beyond perhaps lightweight filtering metrics

- **uncertain**  
  escalate; do not auto-update anything important

### important rule
Status changes should be automatic **unless uncertain**.

---

## 10. Drafting behavior

## 10.1 Draft destination
All v1 outbound drafts should be created directly in **Gmail drafts**.

The user will review and send manually.

## 10.2 v1 draft use cases
Potential v1 draft categories:
- simple acknowledgment
- basic follow-up after silence on an existing thread
- response to routine administrative requests
- polite reply to low-risk employer emails

## 10.3 v1 sending policy
The system must not send emails automatically in v1.

## 10.4 draft tracking
The event log should record:
- draft proposed / created
- the thread or application it is associated with
- why it was created

Tracking user edits before send is optional and not required.

---

## 11. Escalation policy

The system should escalate instead of acting when:

- classification confidence is low
- the email is ambiguous
- the email involves a human conversation that appears non-routine
- scheduling availability is required
- offer-related, compensation-related, relocation-related, visa-related, or similarly important topics appear
- the system cannot confidently determine which record to update
- the system detects conflicting evidence between the database and external systems

### escalation behavior
Escalation should:
- set thread state to `escalated`
- create an event log entry explaining why
- surface the item prominently in the CLI / dashboard view

---

## 12. Follow-up behavior

## 12.1 General rule
Follow-up logic should be thread-based, not purely application-based.

## 12.2 v1 follow-up policy
The system should not blanket-follow up on every application automatically.

Instead:
- follow-up should trigger only for thread types or records that are explicitly marked as follow-up eligible
- one week is the default follow-up interval for stalled email conversations
- only one follow-up should be issued automatically as a draft before checking with the user

## 12.3 application-only records
For passive applications with no conversation thread, v1 should focus on status tracking, not aggressive automated follow-up.

---

## 13. CLI / operator visibility

The first interface may be a CLI.

At minimum, it should support:

### 13.1 Search and inspection
The user should be able to search application records and inspect:
- company
- title
- current status
- linked contacts
- linked email threads
- recent events

### 13.2 Attention views
The system should surface:
- applications with meaningful status changes  
  excluding routine confirmation emails, unless the user explicitly wants them shown
- applications needing attention
- recent relevant emails
- contacts with stalled but not closed threads
- prepared Gmail drafts
- follow-ups due

### 13.3 manual interventions
The user should be able to:
- manually link contacts to applications
- manually link threads to contacts or applications
- override state when needed
- trigger reconciliation manually

A graphical dashboard is desirable later, but not required for v1.

---

## 14. Calendar behavior

Calendar is a later-stage feature, but the spec should reserve space for it.

### target platform
Google Calendar is the first target.

### v2 direction
The system should eventually:
- read meetings from Google Calendar
- create and update events within its calendar profile
- treat all meetings uniformly
- use scheduled meetings as follow-up triggers
- ping the user before sending a meeting follow-up in case the meeting was canceled but the calendar was not updated

Interview-specific scheduling workflows are not part of v1.

---

## 15. LinkedIn behavior

LinkedIn is a later-stage feature.

### v3+ goals
- identify job-search-relevant people to message
- prioritize UC Berkeley alumni
- prioritize mutual connections, subject to user review of whether the connection is meaningful
- avoid spamming multiple people at the same company
- link LinkedIn conversations to contacts
- treat LinkedIn and email as separate communication records under the same contact

### future LinkedIn conversation state machine
Not required in v1, but a separate state machine is expected later.

---

## 16. Safety and trust model

This system exists to automate simple administrative and organizational work, not to run serious hiring conversations autonomously.

### never autonomous in early versions
- scheduling calls without checking availability
- offer-related discussions
- nuanced human conversations with ambiguous intent
- anything where the cost of a wrong response is high

### trust-building approach
Autonomy should expand only after successful testing.

The sequence is:
1. classify and track
2. draft for approval
3. selectively automate a few safe cases later

---

## 17. Failure handling

### if classification is uncertain
Escalate.

### if records conflict
Escalate for human intervention.

### if a thread could link to multiple applications
Do not infer automatically unless evidence is strong and rules are explicit. Prefer user linking.

### if relevance is unclear
Prefer conservative handling.

The system should optimize for avoiding wrong actions over maximizing automation.

---

## 18. Roadmap summary

## v1
- Google Sheet sync
- local DB
- Gmail monitoring 3x/day
- relevant-email classification
- automatic high-confidence status updates
- contact records
- thread-level tracking
- Gmail draft generation
- CLI visibility
- weekly / manual reconciliation
- no auto-send

## v2
- richer contact-centric workflows
- better thread management
- Google Calendar integration
- meeting-triggered reminders
- possible event creation/update
- broader drafting support for human conversations

## v3+
- LinkedIn contact discovery
- LinkedIn message drafting
- possible browser automation
- separate LinkedIn state machine
- interview requested / scheduled / completed stages
- end-of-day interview follow-up drafting
- selective autonomous sending for very safe categories

---

## 19. Open design decisions intentionally deferred

These should not block the product spec, but can be resolved by the implementation team later:

- exact database schema
- whether `Company` becomes a first-class entity in v1 or later
- exact classifier prompts and thresholds
- exact Gmail filtering strategy
- how to represent cached thread summaries
- CLI command design
- secrets handling and connector strategy
- exact reconciliation mechanics
- exact Cowork / Claude Desktop runtime arrangement

---

## 20. Acceptance criteria for v1

v1 is successful if it can reliably do the following:

1. sync applications from the Google Sheet into the local database
2. detect new relevant Gmail threads tied to job search activity
3. classify clear confirmations, rejections, automated action-required emails, and conversation-starting replies
4. update application status automatically when classification is high-confidence
5. track relevant contacts and email threads even when not all are tied cleanly to one application
6. create Gmail drafts for low-risk routine responses or follow-ups
7. surface stalled threads, drafts, and important status changes in a simple operator interface
8. escalate uncertain or sensitive cases instead of acting
