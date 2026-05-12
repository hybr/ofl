# Operational Flow Language (OFL)
## User Guide & Specification — Version 2.0

A practical language for describing how work actually gets done — and what happens when it doesn't.

---

## Table of Contents

1. [What is OFL?](#1-what-is-ofl)
2. [Quickstart](#2-quickstart)
3. [Anatomy of an OFL Document](#3-anatomy-of-an-ofl-document)
4. [Authoring Conventions](#4-authoring-conventions)
5. [Process Header](#5-process-header)
6. [Roles and Identity](#6-roles-and-identity)
7. [Inputs and Outputs (with Schemas)](#7-inputs-and-outputs)
8. [The Flow Section](#8-the-flow-section)
9. [Conditionals, Parallel, and Loops](#9-conditionals-parallel-and-loops)
10. [Subprocesses and Includes](#10-subprocesses-and-includes)
11. [Time, SLAs, and Schedules](#11-time-slas-and-schedules)
12. [Approvals and Delegation](#12-approvals-and-delegation)
13. [Failures, Retries, Holds, and Escalations](#13-failures-retries-holds-and-escalations)
14. [Notifications](#14-notifications)
15. [Evidence and Audit](#15-evidence-and-audit)
16. [Metrics and Costs](#16-metrics-and-costs)
17. [Process States and Lifecycle](#17-process-states-and-lifecycle)
18. [Versioning and Migration](#18-versioning-and-migration)
19. [Full Worked Example](#19-full-worked-example)
20. [Pattern Cookbook](#20-pattern-cookbook)
21. [Keyword Quick Reference](#21-keyword-quick-reference)
22. [Validation Checklist](#22-validation-checklist)
23. [Glossary](#23-glossary)

---

## 1. What is OFL?

**OFL is a human-readable language for describing operational processes.** It is written by Yogesh Sharma (sharma.yogesh.1234@gmail.com).

OFL exists because real processes break in predictable places:

- People forget what they were supposed to do.
- Responsibility gets blurry the moment something goes wrong.
- Escalations expose which systems are real and which were theatre.
- Without metrics, no one is actually in control.
- Good systems survive the absence of any single individual.

OFL treats those five facts as the design problem. It optimises for **accountability, traceability, delegation, failure handling, measurement, and operational continuity** — not for theoretical elegance.

### When to use OFL

Use OFL when you need to:

- Document a process that involves more than one role.
- Define what "done" looks like and who confirms it.
- Specify what happens when steps are missed or delayed.
- Make handoffs survivable when someone is on leave.
- Produce an audit trail that holds up under review.

Do **not** use OFL for:

- Pure software algorithms (use code).
- Personal to-do lists (use a list).
- One-off ad-hoc decisions (use a memo).

### When OFL 2.0 differs from 1.0

This guide describes OFL v2.0, which adds: typed input/output schemas, step IDs, `ELIF`/`ELSE`, loops and retries, subprocesses and `INCLUDE`, explicit time semantics (business vs calendar hours), notification channels, evidence-to-step linking, an instance state model, hold/resume, approval chains with quorum, and migration rules for in-flight instances. Where v2.0 adds new syntax, it is marked **(v2.0)**.

---

## 2. Quickstart

The smallest useful OFL document looks like this:

```ofl
PROCESS PackageReceiving
VERSION 1.0
OWNER SecurityDesk
PURPOSE "Receive resident packages and notify the owner."

ROLES
    DeliveryAgent
    SecurityGuard
    Resident

FLOW
    [s1] DeliveryAgent
        DELIVER Package TO SecurityGuard

    [s2] SecurityGuard
        VERIFY Package.FlatNumber
        STORE Package
        NOTIFY Resident VIA SMS

    [s3] Resident
        COLLECT Package WITHIN 24h CALENDAR

FAILURES
    IF Resident NO_COLLECTION 48h
        NOTIFY Resident VIA Call
        ESCALATE OfficeExecutive
```

That's a complete, valid OFL document. You can read it top to bottom and know: who is involved, what they do, in what order, and what happens if a resident doesn't pick up the package.

The rest of this guide is about scaling this idea up to anything from a complaint workflow to a full compliance regime — without losing the readability you see above.

---

## 3. Anatomy of an OFL Document

An OFL document has up to **eleven sections**, in this order. Only `PROCESS`, `ROLES`, and `FLOW` are mandatory.

| Section          | Purpose                                                | Required |
|------------------|--------------------------------------------------------|----------|
| `PROCESS` header | Identity, version, owner, purpose                      | Yes      |
| `ROLES`          | Who acts in the process                                | Yes      |
| `INPUTS`         | Data/objects the process starts with                   | No       |
| `OUTPUTS`        | Data/objects the process produces                      | No       |
| `STATES` *(v2.0)* | Allowed lifecycle states for the process instance     | No       |
| `FLOW`           | The sequence of actions                                | Yes      |
| `FAILURES`       | What to do when steps break                            | No       |
| `ESCALATIONS`    | The escalation chain                                   | No       |
| `RULES`          | Delegation and permission rules                        | No       |
| `EVIDENCE`       | Required artefacts and retention                       | No       |
| `METRICS`        | What to measure                                        | No       |

Sections appear once each, in the order shown. A linter should reject documents that reorder them — order is part of what makes OFL scannable.

---

## 4. Authoring Conventions

Consistency is what lets a supervisor read an OFL document written by someone else without translation. The following conventions are part of the standard.

### Naming

| Element                | Style                  | Example                       |
|------------------------|------------------------|-------------------------------|
| Process names          | `PascalCase`           | `ResidentComplaintHandling`   |
| Roles                  | `PascalCase`           | `MaintenanceSupervisor`       |
| Inputs / Outputs       | `PascalCase`           | `ComplaintTicket`             |
| Step IDs *(v2.0)*      | `[snake_case]`         | `[verify_ticket]`             |
| Keywords               | `UPPERCASE`            | `SLA`, `ESCALATE`             |
| Fields                 | `snake_case`           | `flat_number`, `created_at`   |
| Constants              | `UPPER_SNAKE_CASE`     | `BUDGET_LIMIT`                |

### Comments *(v2.0)*

```ofl
# Single-line comment

"""
Multi-line comment.
Use for block explanations.
"""
```

Comments are stripped at parse time but should be retained in any rendered diagrams as tooltips.

### Indentation

Two spaces. The hierarchy is: section → actor block → action. The flow reads like an outline; indentation is not just cosmetic, it carries meaning.

### File naming

Use `.ofl` as the extension. File names are `snake_case` and end in the noun being processed:

- `resident_complaint.ofl`
- `lift_breakdown.ofl`
- `monthly_fire_safety_audit.ofl`

---

## 5. Process Header

The header identifies the process and answers the question "what is this thing for?".

```ofl
PROCESS ResidentComplaintHandling
VERSION 2.3
OWNER MaintenanceOffice
PURPOSE "Resolve resident complaints with accountability and a measurable SLA."
EFFECTIVE_FROM 2026-01-01           # (v2.0) when this version became active
SUPERSEDES 2.2                       # (v2.0) the version this replaces
```

- `VERSION` follows `MAJOR.MINOR`. Bump the major version when in-flight instances cannot migrate.
- `OWNER` must be a role or named body that exists in `ROLES` or in the organisation directory. If the owner is gone, the process is orphaned, which is itself an audit finding.
- `PURPOSE` is one sentence. If you cannot say what the process is for in one sentence, the process is doing more than one thing — split it.

---

## 6. Roles and Identity

Roles are the actors in the process. Each role becomes a **swimlane** in any rendered diagram.

```ofl
ROLES
    Resident
    OfficeExecutive
    MaintenanceSupervisor
    Vendor                   EXTERNAL
    Committee                COLLECTIVE QUORUM=MAJORITY     # (v2.0)
```

Modifiers:

- `EXTERNAL` — the role is outside the organisation (vendor, contractor, regulator). External roles cannot be assigned internal tasks like `APPROVE`.
- `COLLECTIVE` — the role represents a body, not an individual. Use with `QUORUM` to define when the body has decided.
- `QUORUM = MAJORITY | ALL | ANY | N`  *(v2.0)* — how many members must agree for a collective action.

### Identity binding *(v2.0)*

A role is a slot; an **identity** is the person currently filling it. Identity is bound at instance time, not at design time. The standard does not specify the directory system, but it does specify the contract:

- Every action in `FLOW` produces an immutable record of `(step_id, role, identity, timestamp)`.
- If an identity changes mid-process (e.g., the OfficeExecutive goes on leave), the new identity inherits the role's pending actions. Substitution is a `RULES`-level concern (see §12).

---

## 7. Inputs and Outputs

### Simple form

```ofl
INPUTS
    ComplaintTicket
    PhotoEvidence
    FlatNumber

OUTPUTS
    ResolvedComplaint
    ResidentConfirmation
    ClosureRecord
```

### Typed schemas *(v2.0)*

When inputs and outputs need structure, declare it. Schemas catch a class of failure that purely narrative OFL cannot: missing fields.

```ofl
INPUTS
    ComplaintTicket {
        id           : UUID            AUTO
        flat_number  : Text            REQUIRED
        category     : Enum(Plumbing, Electrical, Civil, Safety, Other) REQUIRED
        description  : Text(min=10, max=2000) REQUIRED
        priority     : Enum(Low, Normal, High, Critical) DEFAULT Normal
        created_at   : DateTime        AUTO
    }

    PhotoEvidence {
        file_url     : URL             REQUIRED
        taken_at     : DateTime        OPTIONAL
    }
```

Supported primitive types: `Text`, `Integer`, `Decimal`, `Boolean`, `Date`, `DateTime`, `URL`, `UUID`, `Money`, `Enum(...)`.

Modifiers: `REQUIRED`, `OPTIONAL`, `AUTO` (system-set), `DEFAULT <value>`, `IMMUTABLE` (cannot change once set).

Schemas are checked at the moment the input is produced. A `SUBMIT` of a `ComplaintTicket` with no `description` is rejected at the boundary; the process never starts in an invalid state.

---

## 8. The Flow Section

`FLOW` is the heart of OFL. Each block names an actor, followed by an indented list of actions. Actions are verbs in `UPPERCASE` followed by a noun and optional modifiers.

### Step IDs *(v2.0)*

Every action can carry a step ID in square brackets. IDs let you reference steps from `FAILURES`, `EVIDENCE`, metrics, and diagrams without ambiguity.

```ofl
FLOW

    Resident
        [s_submit]   SUBMIT ComplaintTicket
        [s_attach]   ATTACH PhotoEvidence TO ComplaintTicket

    OfficeExecutive
        [s_verify]   VERIFY ComplaintTicket               SLA 2h BUSINESS
        [s_create]   CREATE WorkOrder FROM ComplaintTicket
        [s_assign]   ASSIGN MaintenanceSupervisor TO WorkOrder

    MaintenanceSupervisor
        [s_inspect]  INSPECT Issue                        SLA 6h BUSINESS
        [s_vendor]   ASSIGN Vendor TO WorkOrder

    Vendor
        [s_repair]   COMPLETE Repair
        [s_invoice]  SUBMIT VendorInvoice

    Resident
        [s_confirm]  CONFIRM Resolution                   SLA 48h CALENDAR

    OfficeExecutive
        [s_close]    CLOSE Ticket
                     STORE ClosureRecord
```

### Standard action verbs

| Verb        | Meaning                                                  |
|-------------|----------------------------------------------------------|
| `SUBMIT`    | Send into the process from outside                       |
| `RECEIVE`   | Accept something from another actor                      |
| `VERIFY`    | Confirm validity                                         |
| `CREATE`    | Produce a new artefact                                   |
| `ASSIGN`    | Hand responsibility to a role                            |
| `INSPECT`   | Look at, in person or remotely                           |
| `APPROVE`   | Authorise to proceed                                     |
| `REJECT`    | Refuse to proceed                                        |
| `COMPLETE`  | Finish a unit of work                                    |
| `CONFIRM`   | Acknowledge completion                                   |
| `NOTIFY`    | Send a message (see §14)                                 |
| `STORE`     | Persist an artefact                                      |
| `CLOSE`     | End the instance                                         |
| `HOLD`      | Pause the instance *(v2.0)*                              |
| `RESUME`    | Unpause the instance *(v2.0)*                            |
| `CANCEL`    | End the instance without success *(v2.0)*                |

You may introduce additional verbs for your domain, but keep them uppercase and intransitive in spirit ("VERIFY", not "CHECKING").

### Data references *(v2.0)*

Actions can reference data from earlier steps using dot notation:

```ofl
OfficeExecutive
    [s_create] CREATE WorkOrder {
        complaint_id : s_submit.ComplaintTicket.id
        priority     : s_submit.ComplaintTicket.priority
        opened_by    : ACTOR
        opened_at    : NOW
    }
```

`ACTOR` and `NOW` are reserved tokens for the current identity and the current timestamp.

---

## 9. Conditionals, Parallel, and Loops

### Conditionals *(v2.0: ELIF / ELSE added)*

```ofl
IF ComplaintTicket.priority == Critical
    PRIORITY Critical
    SLA 1h CALENDAR
ELIF ComplaintTicket.category == Plumbing
    SLA 4h BUSINESS
ELSE
    SLA 24h BUSINESS
END
```

Conditionals can be used inside an actor block or at top level inside `FLOW`. Each branch must terminate with `END`.

### Parallel actions *(v2.0: explicit JOIN)*

```ofl
PARALLEL
    BRANCH inspection
        MaintenanceSupervisor INSPECT Issue
    BRANCH quotation
        OfficeExecutive REQUEST VendorQuotation
    BRANCH approval_setup
        OfficeExecutive PREPARE ApprovalPackage
JOIN ALL                       # wait for every branch
# or: JOIN ANY 2 OF 3          # proceed when two branches complete
```

A `PARALLEL` block must always close with a `JOIN`. Without it, the flow has no defined moment of convergence and is invalid.

### Loops and retries *(v2.0)*

```ofl
REPEAT
    Vendor ATTEMPT Repair
UNTIL Repair.status == Successful
    MAX_ATTEMPTS 3
    BACKOFF 30m
    ON_EXHAUSTED ESCALATE Committee
END
```

For a single action with retry semantics:

```ofl
RETRY Vendor.SubmitInvoice
    UP_TO 3 ATTEMPTS
    INTERVAL 1h
    ON_FINAL_FAILURE
        NOTIFY OfficeExecutive
        HOLD WorkOrder REASON "Invoice missing"
END
```

For iteration over a collection:

```ofl
FOR_EACH unit IN BlockA.Units
    SecurityGuard INSPECT unit.FireExtinguisher
    SecurityGuard LOG InspectionResult
END
```

---

## 10. Subprocesses and Includes

### Calling a subprocess *(v2.0)*

```ofl
OfficeExecutive
    CALL_PROCESS VendorOnboarding(vendor: NewVendor)
        AWAIT Result
        BIND vendor_id = Result.id
```

`AWAIT Result` blocks until the subprocess completes. `BIND` captures outputs into named variables visible in subsequent steps.

To run a subprocess in the background:

```ofl
TRIGGER WeeklyMaintenanceReport()
    DETACHED                       # do not wait; the parent flow continues
```

### Including shared definitions *(v2.0)*

Put commonly reused roles, escalation chains, or notification templates in shared files and include them:

```ofl
INCLUDE "library/escalation_chains.ofl"
INCLUDE "library/notification_templates.ofl"
INCLUDE "library/common_roles.ofl"
```

Includes are flat — they cannot themselves contain `FLOW` blocks. They are libraries, not nested processes.

---

## 11. Time, SLAs, and Schedules

Time is where most process definitions get vague. OFL is explicit.

### SLAs *(v2.0: BUSINESS / CALENDAR distinction)*

```ofl
SLA 2h BUSINESS         # 2 hours of business time (Mon–Sat, 9–18, IST)
SLA 30m CALENDAR        # 30 minutes of wall-clock time, including nights
SLA 3d BUSINESS_DAYS    # 3 business days
```

The interpretation of "business" is set per-organisation in a `CALENDAR` file referenced by the deployment, not hard-coded into the process. Emergencies should generally use `CALENDAR`; routine work should use `BUSINESS`.

### Deadlines

Where an absolute time matters, use `DEADLINE`:

```ofl
Committee
    APPROVE BudgetProposal
    DEADLINE 2026-03-31T17:00 IST
```

### Waits *(v2.0)*

```ofl
WAIT 24h CALENDAR              # wait a fixed duration
WAIT UNTIL Resident.Confirms   # wait for an event
WAIT UNTIL 2026-04-01T09:00 IST  # wait until an absolute time
```

### Scheduled processes

```ofl
SCHEDULE MONTHLY ON 1st AT 09:00 IST
    TRIGGER FireSafetyInspection

SCHEDULE CRON "0 9 * * MON"     # every Monday 09:00 server time
    TRIGGER WeeklyMaintenanceReview
```

---

## 12. Approvals and Delegation

### Delegation rules

```ofl
RULES
    OfficeExecutive
        MAY_DELEGATE Verification TO SecurityGuard
        MAY_NOT_DELEGATE Closure

    MaintenanceSupervisor
        MAY_NOT_DELEGATE VendorApproval

    Committee
        APPROVAL_REQUIRED ABOVE 50000 INR
```

### Substitution *(v2.0)*

```ofl
RULES
    OfficeExecutive
        SUBSTITUTE_BY AssistantOfficeExecutive ON LEAVE
        SUBSTITUTE_BY Committee ON LEAVE > 5d
```

Substitution is automatic when the identity registered for the role is marked unavailable. The process does not pause; the work routes to the substitute.

### Approval chains *(v2.0)*

```ofl
APPROVAL BudgetApproval
    REQUEST_FROM Committee
    QUORUM MAJORITY                  # ANY | ALL | MAJORITY | N
    SLA 48h BUSINESS
    ON_TIMEOUT
        ESCALATE Chairman
        EXTEND_SLA 24h BUSINESS
```

Multi-stage approvals are written as a sequence:

```ofl
APPROVAL StagedBudgetApproval
    STAGES
        STAGE 1 REQUEST_FROM OfficeExecutive QUORUM ANY
        STAGE 2 REQUEST_FROM Committee QUORUM MAJORITY
        STAGE 3 REQUEST_FROM Chairman QUORUM ANY
    REQUIRE_ALL_STAGES
```

### Permissions matrix *(v2.0)*

```ofl
RULES
    PERMISSIONS
        OfficeExecutive
            CAN VIEW   ComplaintTicket
            CAN UPDATE WorkOrder
            CAN CLOSE  Ticket WHEN ResidentConfirmation EXISTS
        SecurityGuard
            CAN VIEW   ComplaintTicket
            CANNOT UPDATE WorkOrder
```

Permissions are evaluated for every action. A `CLOSE` attempted before the required confirmation is rejected by the runtime, not just by convention.

---

## 13. Failures, Retries, Holds, and Escalations

In OFL, failure is not a footnote — it is a first-class section. The single most common audit finding in real organisations is "we did not write down what to do when things went wrong."

### The Failures section

```ofl
FAILURES

    IF [s_verify] NO_RESPONSE 2h BUSINESS
        ESCALATE Committee
        NOTIFY Resident VIA SMS

    IF [s_repair] DELAY > 24h CALENDAR
        REASSIGN Vendor
        NOTIFY Resident VIA SMS

    IF VendorInvoice MISSING AFTER [s_repair] + 48h BUSINESS
        HOLD WorkOrder REASON "Invoice pending"
        NOTIFY Vendor VIA Email, SMS

    IF Budget APPROVAL_REJECTED
        HOLD WorkOrder REASON "Budget denied"
        SEND CommitteeReview

    IF EmergencyIssue
        BYPASS NormalApproval
        PRIORITY Critical
        SLA 1h CALENDAR
```

### Hold and Resume *(v2.0)*

```ofl
HOLD WorkOrder
    REASON "Parts unavailable from vendor"
    NOTIFY Resident VIA SMS
    AUTO_RESUME WHEN parts.arrived
    MAX_HOLD 7d CALENDAR
    ON_MAX_HOLD ESCALATE Committee
```

A held instance is not a failed instance — it is a paused instance with a defined resumption condition. If `MAX_HOLD` is exceeded, the failure path runs.

### Escalation chains

```ofl
ESCALATIONS
    Level1 MaintenanceSupervisor    SLA 4h  BUSINESS
    Level2 Committee                SLA 24h CALENDAR
    Level3 EmergencyBoard           SLA 4h  CALENDAR
```

Each level inherits the previous level's responsibility plus the SLA at which it itself escalates. Escalation does not transfer the work alone; it transfers **accountability**. The original assignee remains on the instance record.

---

## 14. Notifications

The original spec said `NOTIFY` without saying how. v2.0 requires a channel and supports fallbacks.

```ofl
NOTIFY Resident
    VIA SMS, Email
    TEMPLATE "complaint_received"
    PARAMS {
        ticket_id : ComplaintTicket.id
        eta       : "4 hours"
    }
    FALLBACK Email IF SMS_FAILED
    RETRY 3 INTERVAL 5m
```

Supported channels: `SMS`, `Email`, `Call`, `Push`, `WhatsApp`, `Notice` (physical notice board), `Webhook`.

Templates are external to OFL — they are message bodies with parameter placeholders, stored as separate files and referenced by name. Keeping message text out of process definitions is deliberate: it lets compliance review wording without rewriting flows.

---

## 15. Evidence and Audit

Evidence is what proves the process actually happened.

```ofl
EVIDENCE
    ComplaintTicket        REQUIRED AT [s_submit]
    PhotoEvidence          OPTIONAL AT [s_submit]
    WorkOrder              REQUIRED AT [s_create]
    VendorInvoice          REQUIRED AT [s_invoice]
    ResidentConfirmation   REQUIRED AT [s_confirm]
    ClosureRecord          REQUIRED AT [s_close]

AUDIT
    RETAIN ClosureRecord    5y
    RETAIN VendorInvoice    7y
    RETAIN PhotoEvidence    2y
    LOG ALL_ESCALATIONS
    LOG ALL_HOLDS
    LOG ALL_SUBSTITUTIONS
    IMMUTABLE_AFTER CLOSE
```

`AT [step_id]` is the v2.0 improvement that ties each evidence requirement to a specific step. Without this, evidence sections drift away from the flow they were meant to anchor.

---

## 16. Metrics and Costs

### Metrics

```ofl
METRICS
    TRACK ResolutionTime              FROM [s_submit] TO [s_close]
    TRACK PendingTickets              SNAPSHOT DAILY
    TRACK RepeatComplaints            WINDOW 30d
    TRACK EscalationFrequency
    TRACK VendorDelayRate             PER Vendor
    TRACK ResidentSatisfaction        FROM ResidentConfirmation.rating

    ALERT WHEN PendingTickets > 50
    ALERT WHEN ResolutionTime.median > 24h BUSINESS
```

Where the original spec only listed names, v2.0 lets you specify **how** the metric is computed and **when** it fires alerts.

### Costs *(v2.0)*

```ofl
COSTS
    TRACK MaterialCost    AT [s_invoice]   FROM VendorInvoice.materials
    TRACK LaborCost       AT [s_invoice]   FROM VendorInvoice.labor
    BUDGET LIMIT 25000 INR PER Instance
    ALERT WHEN ACTUAL > 80% OF BUDGET
    REJECT WHEN ACTUAL > 100% OF BUDGET WITHOUT Committee.Approval
```

---

## 17. Process States and Lifecycle

A process **definition** is a template. A process **instance** is one execution of that template. v2.0 makes the instance lifecycle explicit.

```ofl
STATES
    NEW              -> IN_PROGRESS
    IN_PROGRESS      -> { AWAITING_APPROVAL, ON_HOLD, ESCALATED, CLOSED, CANCELLED }
    AWAITING_APPROVAL -> { IN_PROGRESS, REJECTED, ESCALATED }
    ON_HOLD          -> { IN_PROGRESS, CANCELLED }
    ESCALATED        -> { IN_PROGRESS, CLOSED }
    REJECTED         -> CLOSED
    CLOSED           # terminal
    CANCELLED        # terminal
```

If `STATES` is omitted, OFL uses a default chain: `NEW → IN_PROGRESS → { CLOSED, CANCELLED }`.

The state of an instance is visible to all roles, queryable in metrics, and immutable after a terminal state is reached.

---

## 18. Versioning and Migration

What happens when you change the process while instances are running?

```ofl
PROCESS ResidentComplaintHandling
VERSION 2.3
SUPERSEDES 2.2

MIGRATION FROM 2.2
    KEEP_ON_OLD_VERSION instances WHERE state IN { ON_HOLD, AWAITING_APPROVAL }
    MIGRATE instances WHERE state == IN_PROGRESS
        MAP step [old_verify]  -> [s_verify]
        MAP step [old_inspect] -> [s_inspect]
    REQUIRE_REVIEW BY OfficeExecutive
```

The rule is simple: no in-flight instance changes silently. Either it stays on the old version until it terminates, or it migrates with an explicit mapping that a human has approved.

A major version bump (`2.x → 3.0`) signals that no migration path exists — old instances must complete on their original version. This is the safe default for serious changes.

---

## 19. Full Worked Example

Here is a realistic complaint workflow that uses most of v2.0's features:

```ofl
PROCESS ResidentComplaintHandling
VERSION 2.0
OWNER MaintenanceOffice
PURPOSE "Resolve resident complaints with accountability and a measurable SLA."
EFFECTIVE_FROM 2026-01-01

INCLUDE "library/escalation_chains.ofl"
INCLUDE "library/notification_templates.ofl"

ROLES
    Resident
    OfficeExecutive
    MaintenanceSupervisor
    Vendor               EXTERNAL
    Committee            COLLECTIVE QUORUM=MAJORITY

INPUTS
    ComplaintTicket {
        id          : UUID AUTO
        flat_number : Text REQUIRED
        category    : Enum(Plumbing, Electrical, Civil, Safety, Other) REQUIRED
        description : Text(min=10, max=2000) REQUIRED
        priority    : Enum(Low, Normal, High, Critical) DEFAULT Normal
        created_at  : DateTime AUTO
    }
    PhotoEvidence {
        file_url : URL REQUIRED
    }

OUTPUTS
    ResolvedComplaint
    ResidentConfirmation
    ClosureRecord

STATES
    NEW -> IN_PROGRESS
    IN_PROGRESS -> { AWAITING_APPROVAL, ON_HOLD, ESCALATED, CLOSED, CANCELLED }
    AWAITING_APPROVAL -> { IN_PROGRESS, REJECTED }
    ON_HOLD -> { IN_PROGRESS, CANCELLED }
    ESCALATED -> { IN_PROGRESS, CLOSED }

FLOW

    Resident
        [s_submit]   SUBMIT ComplaintTicket
        [s_attach]   ATTACH PhotoEvidence TO ComplaintTicket OPTIONAL

    OfficeExecutive
        [s_verify]   VERIFY ComplaintTicket            SLA 2h BUSINESS

    IF ComplaintTicket.priority == Critical
        PRIORITY Critical
        SLA 1h CALENDAR
    ELIF ComplaintTicket.category == Plumbing
        SLA 4h BUSINESS
    ELSE
        SLA 24h BUSINESS
    END

    OfficeExecutive
        [s_create]   CREATE WorkOrder {
            complaint_id : s_submit.ComplaintTicket.id
            priority     : s_submit.ComplaintTicket.priority
        }
        [s_assign]   ASSIGN MaintenanceSupervisor TO WorkOrder

    MaintenanceSupervisor
        [s_inspect]  INSPECT Issue                     SLA 6h BUSINESS
        [s_vendor]   ASSIGN Vendor TO WorkOrder

    Vendor
        [s_repair]   COMPLETE Repair                   SLA 24h CALENDAR
        [s_invoice]  SUBMIT VendorInvoice              SLA 48h BUSINESS

    Resident
        [s_confirm]  CONFIRM Resolution                SLA 48h CALENDAR

    OfficeExecutive
        [s_close]    CLOSE Ticket
                     STORE ClosureRecord

FAILURES
    IF [s_verify] NO_RESPONSE 2h BUSINESS
        ESCALATE Committee
        NOTIFY Resident VIA SMS TEMPLATE "verify_delayed"

    IF [s_repair] DELAY > 24h CALENDAR
        REASSIGN Vendor
        NOTIFY Resident VIA SMS TEMPLATE "vendor_changed"

    IF [s_invoice] MISSING AFTER [s_repair] + 48h BUSINESS
        HOLD WorkOrder REASON "Invoice pending"
        NOTIFY Vendor VIA Email, SMS RETRY 3 INTERVAL 1h

    IF WorkOrder.cost > 50000 INR
        REQUEST_APPROVAL Committee
        QUORUM MAJORITY
        SLA 48h BUSINESS
        ON_TIMEOUT ESCALATE Chairman

ESCALATIONS
    Level1 MaintenanceSupervisor    SLA 4h  BUSINESS
    Level2 Committee                SLA 24h CALENDAR
    Level3 EmergencyBoard           SLA 4h  CALENDAR

RULES
    OfficeExecutive MAY_DELEGATE Verification TO SecurityGuard
    OfficeExecutive MAY_NOT_DELEGATE Closure
    OfficeExecutive SUBSTITUTE_BY AssistantOfficeExecutive ON LEAVE
    MaintenanceSupervisor MAY_NOT_DELEGATE VendorApproval
    Committee APPROVAL_REQUIRED ABOVE 50000 INR

    PERMISSIONS
        OfficeExecutive CAN CLOSE Ticket WHEN ResidentConfirmation EXISTS
        SecurityGuard CANNOT UPDATE WorkOrder

EVIDENCE
    ComplaintTicket       REQUIRED AT [s_submit]
    PhotoEvidence         OPTIONAL AT [s_submit]
    WorkOrder             REQUIRED AT [s_create]
    VendorInvoice         REQUIRED AT [s_invoice]
    ResidentConfirmation  REQUIRED AT [s_confirm]
    ClosureRecord         REQUIRED AT [s_close]

AUDIT
    RETAIN ClosureRecord 5y
    RETAIN VendorInvoice 7y
    LOG ALL_ESCALATIONS
    LOG ALL_HOLDS
    IMMUTABLE_AFTER CLOSE

METRICS
    TRACK ResolutionTime         FROM [s_submit] TO [s_close]
    TRACK PendingTickets         SNAPSHOT DAILY
    TRACK RepeatComplaints       WINDOW 30d
    TRACK EscalationFrequency
    TRACK VendorDelayRate        PER Vendor
    TRACK ResidentSatisfaction   FROM ResidentConfirmation.rating

    ALERT WHEN PendingTickets > 50
    ALERT WHEN ResolutionTime.median > 24h BUSINESS

COSTS
    TRACK MaterialCost  AT [s_invoice] FROM VendorInvoice.materials
    TRACK LaborCost     AT [s_invoice] FROM VendorInvoice.labor
    BUDGET LIMIT 25000 INR PER Instance
    ALERT WHEN ACTUAL > 80% OF BUDGET
```

---

## 20. Pattern Cookbook

Common situations and the OFL idioms that handle them.

### Pattern: "Approver is on leave"

```ofl
RULES
    OfficeExecutive
        SUBSTITUTE_BY AssistantOfficeExecutive ON LEAVE
        SUBSTITUTE_BY Committee ON LEAVE > 5d
```

### Pattern: "Don't let one vendor block everything"

```ofl
FAILURES
    IF Vendor.response_time.weekly_avg > 24h
        REVIEW Vendor
        SUGGEST_REASSIGN ALL OPEN_WORK_ORDERS
```

### Pattern: "Skip the queue for emergencies"

```ofl
IF EmergencyIssue
    BYPASS NormalApproval
    PRIORITY Critical
    SLA 1h CALENDAR
    REQUIRE Post-Hoc-Review BY Committee WITHIN 48h BUSINESS
```

Note the post-hoc review — bypass is allowed but not unaccountable.

### Pattern: "Two-eyes for high-value spend"

```ofl
APPROVAL HighValueSpend
    STAGES
        STAGE 1 REQUEST_FROM OfficeExecutive QUORUM ANY
        STAGE 2 REQUEST_FROM Committee QUORUM MAJORITY
    REQUIRE_ALL_STAGES
    APPLIES_WHEN amount > 100000 INR
```

### Pattern: "Recurring inspection that produces a report"

```ofl
SCHEDULE MONTHLY ON 1st AT 09:00 IST
    TRIGGER FireSafetyInspection

PROCESS FireSafetyInspection
ROLES
    SecurityGuard
    MaintenanceSupervisor
    Committee

FLOW
    FOR_EACH unit IN Building.Units
        SecurityGuard INSPECT unit.FireExtinguisher
        SecurityGuard LOG InspectionResult
    END

    MaintenanceSupervisor
        GENERATE ComplianceReport
        SUBMIT ComplianceReport TO Committee

EVIDENCE
    ComplianceReport REQUIRED
    RETAIN ComplianceReport 7y
```

### Pattern: "Resident disputes the closure"

```ofl
FAILURES
    IF Resident REJECTS Resolution AT [s_confirm]
        REOPEN Ticket
        ESCALATE Committee
        NOTIFY MaintenanceSupervisor VIA SMS, Email
```

---

## 21. Keyword Quick Reference

### Section headers
`PROCESS` · `VERSION` · `OWNER` · `PURPOSE` · `EFFECTIVE_FROM` · `SUPERSEDES` · `ROLES` · `INPUTS` · `OUTPUTS` · `STATES` · `FLOW` · `FAILURES` · `ESCALATIONS` · `RULES` · `EVIDENCE` · `AUDIT` · `METRICS` · `COSTS` · `MIGRATION`

### Flow control
`IF` · `ELIF` · `ELSE` · `END` · `PARALLEL` · `BRANCH` · `JOIN` · `REPEAT` · `UNTIL` · `FOR_EACH` · `RETRY` · `WAIT` · `CALL_PROCESS` · `TRIGGER` · `INCLUDE` · `SCHEDULE` · `AWAIT` · `BIND`

### Actions
`SUBMIT` · `RECEIVE` · `VERIFY` · `CREATE` · `ASSIGN` · `INSPECT` · `APPROVE` · `REJECT` · `COMPLETE` · `CONFIRM` · `NOTIFY` · `STORE` · `CLOSE` · `HOLD` · `RESUME` · `CANCEL` · `ESCALATE` · `REASSIGN` · `BYPASS`

### Time
`SLA` · `DEADLINE` · `BUSINESS` · `CALENDAR` · `BUSINESS_DAYS` · `NOW`

### Modifiers
`REQUIRED` · `OPTIONAL` · `AUTO` · `DEFAULT` · `IMMUTABLE` · `EXTERNAL` · `COLLECTIVE` · `QUORUM` · `MAY_DELEGATE` · `MAY_NOT_DELEGATE` · `SUBSTITUTE_BY` · `CAN` · `CANNOT`

### Constants
`ACTOR` (current identity) · `NOW` (current timestamp)

---

## 22. Validation Checklist

Before considering an OFL document complete, verify:

- [ ] The `PROCESS` header has a one-sentence `PURPOSE`.
- [ ] Every role in `FLOW` is declared in `ROLES`.
- [ ] Every step has a step ID if it is referenced from `FAILURES`, `EVIDENCE`, or `METRICS`.
- [ ] Every `SLA` specifies `BUSINESS` or `CALENDAR`.
- [ ] Every `PARALLEL` block is closed with a `JOIN`.
- [ ] Every `IF` is closed with `END`.
- [ ] Every `NOTIFY` specifies `VIA` and at least one channel.
- [ ] Every action that can fail is covered in `FAILURES` or has a documented reason for being uncovered.
- [ ] Every monetary threshold is paired with a currency.
- [ ] Every approval has a `QUORUM` and an `ON_TIMEOUT` path.
- [ ] Every `HOLD` has an `AUTO_RESUME` condition or a `MAX_HOLD`.
- [ ] `EVIDENCE` items are linked to steps with `AT [step_id]`.
- [ ] `AUDIT` retention is specified for every required evidence type.
- [ ] The `OWNER` is a role or body that still exists in the directory.
- [ ] A non-technical reviewer can read the document and explain it back.

The last check is the one that matters most.

---

## 23. Glossary

**Action** — A verb-noun pair inside an actor block, e.g. `VERIFY ComplaintTicket`.

**Actor** — A role taking an action in the flow. Becomes a swimlane in diagrams.

**Branch** — One parallel path inside a `PARALLEL` block.

**Calendar time** — Wall-clock time, including nights and holidays.

**Business time** — Time that elapses only during defined working hours.

**Escalation** — Transferring accountability to a higher level when a step is unmet.

**Evidence** — An artefact whose existence proves a step happened.

**Hold** — A paused state with a defined resumption condition; distinct from cancellation.

**Identity** — The specific person filling a role at a specific time.

**Instance** — One execution of a process. The process is the template; instances are the runs.

**Migration** — The act of moving an in-flight instance from one process version to another.

**Quorum** — The number of members of a collective role required to make a decision.

**Role** — A named slot in the process. Roles are filled by identities at run time.

**SLA** — Service-level agreement. The maximum allowed time for a step before failure handling triggers.

**Step ID** — A short identifier in square brackets used to reference a specific action from elsewhere in the document.

**Subprocess** — A separate process invoked from inside a parent process.

**Substitution** — Automatic re-routing of work when the identity in a role is unavailable.

**Swimlane** — A horizontal band in a process diagram corresponding to a single role.

---

## Closing note

OFL exists because a process is not what is written on a poster in the office. A process is what survives a Monday morning when two people are on leave, the vendor is late, the resident is angry, and someone has to decide.

If the document you have just written does not tell you who decides, in what time, with what evidence, and what happens when they don't — then it is not yet a process. It is an aspiration.

OFL is the difference.
