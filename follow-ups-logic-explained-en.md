# AI Follow-Ups - Complete Logic and Conditions

Document explaining ALL conditions for when the system generates and sends automatic follow-ups.

---

## Table of Contents
1. [Quick Reference Summary](#summary)
2. [Main Flow](#main-flow)
3. [PHASE 1: Qualification for Assessment](#phase-1-qualification)
4. [PHASE 2: AI Assessment](#phase-2-ai-assessment)
5. [PHASE 3: Automatic Sending](#phase-3-automatic-sending)
6. [Working Hours and Postponement](#working-hours)
7. [Anti-Spam Protection (25 minutes)](#anti-spam)
8. [Deletion of Obsolete Follow-ups](#deletion)
9. [All Time Thresholds](#time-thresholds)
10. [FAQ for Product Designer](#faq)

---

## Summary

### When CAN the system generate a follow-up?

| # | Condition | Value | Source File |
|---|-----------|-------|-------------|
| 1 | RFP status | SENT or ACTIVE (or has `still_in_progress_at`) | RfpIdsForFollowUpAssessmentQuery:57-58 |
| 2 | DE status | NEW or IN_PROGRESS | DirectEnquiriesForFollowUpAssessmentQuery:52 |
| 3 | Enquiry status | SENT or ACTIVE | RfpIdsForFollowUpAssessmentQuery:62 |
| 4 | Enquiry | NOT archived, NOT deleted | RfpIdsForFollowUpAssessmentQuery:60-61 |
| 5 | Thread | NOT deleted | RfpIdsForFollowUpAssessmentQuery:65 |
| 6 | Enquiry age | Created max 3 months ago | RfpIdsForFollowUpAssessmentQuery:63 |
| 7 | Event date | In the future | RfpIdsForFollowUpAssessmentQuery:64 |
| 8 | Pending follow-up | NO unsent follow-up exists | RfpIdsForFollowUpAssessmentQuery:51-54,66 |
| 9 | Last manager message | EXISTS | RfpIdsForFollowUpAssessmentQuery:71-75 |
| 10 | Last manager message | is NOT a follow-up | RfpIdsForFollowUpAssessmentQuery:77-80 |
| 11 | Time since last msg | >= 30 hours | RfpIdsForFollowUpAssessmentQuery:82-86 |
| 12 | Time since last msg | <= 15 days | RfpIdsForFollowUpAssessmentQuery:88-92 |
| 13 | Assessment lock | NOT assessed in last 12 hours | FollowUpAssessmentTracker:11 |

### When WILL the system send automatically?

| # | Condition | Value | What happens if NOT met |
|---|-----------|-------|------------------------|
| 1 | Generation type | AUTOMATIC | Waits for manual sending |
| 2 | Manager setting | `hasAutomaticAiFollowUpsEnabled() = true` | Waits for manual sending |
| 3 | Client email | Can be determined | Error, follow-up deleted |
| 4 | Working hours | 9:00-17:00 (venue timezone) | **POSTPONED** to next 9:00 AM |
| 5 | Recent auto-followup | Client didn't receive one in last 25 min | **DELAYED** - will retry later |

---

## Main Flow

```mermaid
flowchart TB
    subgraph Phase1["PHASE 1: QUALIFICATION"]
        A[Scheduled Job] --> B[RfpIdsForFollowUpAssessmentQuery]
        B --> C{13 conditions met?}
        C -->|NO| D[Skip]
        C -->|YES| E[Emit event for assessment]
    end

    subgraph Phase2["PHASE 2: AI ASSESSMENT"]
        E --> F{Recently assessed? 12h lock}
        F -->|YES| G[Skip - already assessed]
        F -->|NO| H[AI evaluates conversation]
        H --> I{Follow-up needed?}
        I -->|NO| J[Mark assessed, no action]
        I -->|YES| K[Generate follow-up]
    end

    subgraph Phase3["PHASE 3: SENDING"]
        K --> L[Create AIFollowUp record]
        L --> M[GPT-4 generates message]
        M --> N{Working hours 9-17?}
        N -->|NO| O[POSTPONE to next 9:00 AM]
        N -->|YES| P{Manager has auto enabled?}
        O --> Q[SendPostponedFollowUpsCommand]
        Q --> P
        P -->|NO| R[Wait for manual sending]
        P -->|YES| S{Can get client email?}
        S -->|NO| T[Delete follow-up - error]
        S -->|YES| U{Client got auto-FU in 25 min?}
        U -->|YES| V[SKIP now - retry later]
        U -->|NO| W[SEND AUTOMATICALLY]
    end

    style W fill:#90EE90
    style R fill:#FFE4B5
    style O fill:#87CEEB
    style V fill:#87CEEB
    style T fill:#FFB6C1
```

---

## PHASE 1: Qualification

### All 13 Conditions for RFP Qualification

```mermaid
flowchart TB
    subgraph DB["Database-Level Filters (SQL Query)"]
        A1["1. RFP status = SENT or ACTIVE<br/>OR still_in_progress_at IS NOT NULL"]
        A2["2. Enquiry NOT archived"]
        A3["3. Enquiry NOT deleted"]
        A4["4. Enquiry status = SENT or ACTIVE"]
        A5["5. RFP created within last 3 months"]
        A6["6. Event date > NOW (future)"]
        A7["7. Thread NOT deleted"]
        A8["8. NO unsent follow-up exists for this RFP"]
    end

    subgraph PHP["PHP-Level Validation (per RFP)"]
        B1["9. Last manager message EXISTS"]
        B2["10. Last message is NOT a follow-up"]
        B3["11. Last message sent >= 30 hours ago"]
        B4["12. Last message sent <= 15 days ago"]
    end

    subgraph Lock["Assessment Lock"]
        C1["13. NOT assessed in last 12 hours"]
    end

    DB --> PHP --> Lock --> D[QUALIFIES FOR ASSESSMENT]

    style D fill:#90EE90
```

### Conditions for Direct Enquiry Qualification

| # | Condition | Value |
|---|-----------|-------|
| 1 | DE status | NEW or IN_PROGRESS |
| 2 | DE | NOT deleted |
| 3 | Created | within last 3 months |
| 4 | Event date | in the future |
| 5 | Thread | NOT deleted |
| 6 | Pending follow-up | NO unsent follow-up exists |
| 7 | Last manager message | EXISTS |
| 8 | Last message | is NOT a follow-up |
| 9 | Time since last msg | >= 30 hours |
| 10 | Time since last msg | <= 15 days |
| 11 | Assessment lock | NOT assessed in last 12 hours |

### Time Window Explanation (30h - 15 days)

```mermaid
timeline
    title Follow-up Qualification Window

    section Too Early - Client may respond
        0h : Manager sends message
        12h : Too early
        24h : Too early
        29h : Still too early

    section Ideal Window
        30h : START - can assess
        48h : Good timing
        72h : Good timing
        7 days : Still OK
        10 days : Still OK
        14 days : Last chance

    section Too Late - Would be stale
        15 days : END - too late
        20 days : Too late
```

---

## PHASE 2: AI Assessment

### Assessment Flow

```mermaid
sequenceDiagram
    participant Queue as Assessment Queue
    participant Lock as FollowUpAssessmentTracker
    participant AI as FollowUpLlmAssessmentService
    participant DB as Database

    Queue->>Lock: Check if recently assessed
    Lock-->>Queue: wasRfpRecentlyAssessed(rfpId)?

    alt Already assessed in last 12h
        Queue->>Queue: Skip - don't re-assess
    else Not recently assessed
        Queue->>Lock: markRfpBeingAssessed(rfpId)
        Note over Lock: Lock for 12 hours
        Queue->>AI: assessFollowUpNeeded(rfpId)
        AI->>AI: Analyze conversation with GPT-4
        AI-->>Queue: true/false

        alt Follow-up needed
            Queue->>DB: Create AIFollowUp record
            Queue->>AI: Generate follow-up message
        else Not needed
            Queue->>Queue: Done - no action
        end
    end
```

---

## PHASE 3: Automatic Sending

### All Sending Conditions (in order)

```mermaid
flowchart TB
    A[Follow-up with AI message ready] --> B{1. Generation impulse = AUTOMATIC?}
    B -->|NO - ON_DEMAND| C1[Wait for manual sending<br/>Manager decides when to send]
    B -->|YES| D{2. Manager has auto-followups enabled?}

    D -->|NO| C2[Wait for manual sending<br/>Manager can enable setting later]
    D -->|YES| E{3. Can determine client email?}

    E -->|NO| F[ERROR: Delete follow-up<br/>to prevent retry loop]
    E -->|YES| G{4. Within working hours 9-17?}

    G -->|NO| H[POSTPONE<br/>Set toBeSentAt = next 9:00 AM<br/>Will be retried by scheduled job]
    G -->|YES| I{5. Client received auto-FU<br/>in last 25 minutes?}

    I -->|YES| J[DELAY - Skip this attempt<br/>Will be retried by scheduled job<br/>if follow-up is postponed]
    I -->|NO| K[SEND AUTOMATICALLY]

    style K fill:#90EE90
    style C1 fill:#FFE4B5
    style C2 fill:#FFE4B5
    style F fill:#FFB6C1
    style H fill:#87CEEB
    style J fill:#87CEEB
```

---

## Working Hours

### Working Hours Definition

| Parameter | Value | Source |
|-----------|-------|--------|
| Start | 9:00 AM | WorkingHoursCalculator:12 |
| End | 5:00 PM (17:00) | WorkingHoursCalculator:13 |
| Timezone | Venue's timezone (or server default) | SendAIFollowUpAutomaticallyOrPostponeListener:49 |

```mermaid
gantt
    title Working Hours for Automatic Follow-ups
    dateFormat HH:mm
    axisFormat %H:%M

    section Outside Hours (POSTPONE)
    Night 00:00-09:00     :crit, 00:00, 9h

    section Working Hours (CAN SEND)
    Can send 09:00-17:00 :active, 09:00, 8h

    section Outside Hours (POSTPONE)
    Evening 17:00-24:00 :crit, 17:00, 7h
```

### Postponement Logic

```mermaid
sequenceDiagram
    participant FU as Follow-up
    participant Listener as SendAIFollowUpAutomaticallyOrPostponeListener
    participant Calc as WorkingHoursCalculator
    participant DB as Database

    Note over FU: Generated at 10:00 PM

    Listener->>Calc: isWithinWorkingHours(venueTimezone, now)?
    Calc-->>Listener: false (22:00 > 17:00)

    Listener->>Calc: calculateNextWorkingHour(venueTimezone, now)
    Calc-->>Listener: Tomorrow 9:00 AM UTC

    Listener->>FU: sendLaterAt(tomorrow 9:00 AM)
    Listener->>DB: Save follow-up with toBeSentAt

    Note over FU: Next day at 9:00+ AM

    rect rgb(200, 230, 200)
        Note over FU: SendPostponedFollowUpsCommand runs
        Note over FU: Picks up follow-ups where toBeSentAt <= now
        Note over FU: Attempts to send again
    end
```

---

## Anti-Spam

### The 25-Minute Protection - IMPORTANT EXPLANATION

**What it is**: Protection against sending multiple automatic follow-ups to the same client in a short time.

**What it checks**: Did this client's email receive ANY automatic follow-up (from any RFP or Direct Enquiry) in the last 25 minutes?

```mermaid
flowchart TB
    subgraph Check["25-Minute Check"]
        A[New follow-up ready to send] --> B[Get client email]
        B --> C[Search database:<br/>Any auto-followups sent to this email<br/>in last 25 minutes?]

        C --> D{Found?}
        D -->|YES from RFP| E[SKIP this attempt]
        D -->|YES from Direct Enquiry| E
        D -->|NO| F[OK - proceed to send]
    end

    subgraph Retry["What happens after SKIP?"]
        E --> G{Does follow-up have<br/>toBeSentAt set?}
        G -->|YES - was postponed| H[Will be retried by<br/>SendPostponedFollowUpsCommand<br/>= DELAY, not block]
        G -->|NO - was in working hours| I[Remains as 'ready'<br/>Manager can send manually<br/>or wait for next assessment cycle]
    end

    style F fill:#90EE90
    style H fill:#87CEEB
    style I fill:#FFE4B5
```

### Key Point: 25 Minutes is a DELAY, not a BLOCK

| Scenario | What happens | Result |
|----------|--------------|--------|
| Follow-up generated **OUTSIDE** working hours | Has `toBeSentAt` set → `SendPostponedFollowUpsCommand` will retry | **DELAY** - will be sent later |
| Follow-up generated **INSIDE** working hours, 25min check fails | No `toBeSentAt` → stays as "ready" | Manager can send manually, or new follow-up will be generated in next cycle |

**Why 25 minutes?**
- Same client might have multiple enquiries to different venues
- These venues might be managed by the same manager
- Without this check, client could receive 3-4 follow-ups within minutes

---

## Deletion

### All Deletion Conditions for RFP Follow-ups

```mermaid
flowchart TB
    A[Pending RFP follow-up] --> B{Check deletion conditions}

    B --> C{1. No manager message > 15 days?}
    C -->|YES| DELETE

    B --> D{2. RFP is successful?}
    D -->|YES| DELETE

    B --> E{3. RFP is unsuccessful?}
    E -->|YES| DELETE

    B --> F{4. RFP has declined custom offer?}
    F -->|YES| DELETE

    B --> G{5. RFP is accepted?<br/>acceptedAt != null}
    G -->|YES| DELETE

    B --> H{6. Event in < 7 days<br/>or in the past?}
    H -->|YES| DELETE

    B --> I{7. Client wrote AFTER<br/>follow-up was created?}
    I -->|YES| DELETE

    DELETE[DELETE FOLLOW-UP]

    style DELETE fill:#FFB6C1
```

### All Deletion Conditions for Direct Enquiry Follow-ups

```mermaid
flowchart TB
    A[Pending DE follow-up] --> B{Check deletion conditions}

    B --> C{1. No manager message > 15 days?}
    C -->|YES| DELETE

    B --> D{2. DE is confirmed?}
    D -->|YES| DELETE

    B --> E{3. DE is lost?}
    E -->|YES| DELETE

    B --> F{4. Event in < 7 days<br/>or in the past?}
    F -->|YES| DELETE

    B --> G{5. Client wrote AFTER<br/>follow-up was created?}
    G -->|YES| DELETE

    DELETE[DELETE FOLLOW-UP]

    style DELETE fill:#FFB6C1
```

### Deletion Reasons Summary

| # | Condition | Applies to | Why delete? |
|---|-----------|------------|-------------|
| 1 | No manager message > 15 days | RFP, DE | Conversation probably dead |
| 2 | RFP successful | RFP | Deal won - no need for follow-up |
| 3 | RFP unsuccessful | RFP | Deal lost - no need for follow-up |
| 4 | RFP declined custom offer | RFP | Client rejected offer |
| 5 | RFP accepted | RFP | Deal in progress |
| 6 | DE confirmed | DE | Booking confirmed |
| 7 | DE lost | DE | Deal lost |
| 8 | Event in < 7 days or past | RFP, DE | Too late for follow-up |
| 9 | Client wrote after follow-up created | RFP, DE | Client responded - follow-up unnecessary |

---

## Time Thresholds

### Complete List of All Time Values

| Value | Where Used | Purpose | Source File |
|-------|------------|---------|-------------|
| **25 minutes** | Anti-spam check | Max 1 auto-followup per client email in this window | ClientRecentlyReceivedAiAutoFollowUpQuery:13 |
| **30 hours** | Qualification | Minimum time since last manager message | RfpIdsForFollowUpAssessmentQuery:82 |
| **15 days** | Qualification | Maximum time since last manager message | RfpIdsForFollowUpAssessmentQuery:88 |
| **24 hours** | Cooldown | Gap between follow-ups to same RFP/DE | CooldownStatus:12 |
| **12 hours** | Assessment lock | Don't re-assess same enquiry | FollowUpAssessmentTracker:11 |
| **3 months** | Enquiry filter | Only recent enquiries considered | RfpIdsForFollowUpAssessmentQuery:63 |
| **15 days** | Deletion | Delete if no manager activity | DeleteObsoleteAIFollowUpsCommand:23 |
| **7 days** | Deletion | Delete if event is soon | DeleteObsoleteAIFollowUpsCommand:24 |
| **9:00-17:00** | Working hours | Automatic sending window | WorkingHoursCalculator:12-13 |

---

## State Diagrams

### Follow-up Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Created: Generate

    Created --> MessageGenerated: AI creates content

    MessageGenerated --> Scheduled: Outside working hours
    MessageGenerated --> ReadyToSend: Within working hours

    Scheduled --> ReadyToSend: Scheduled time arrived

    ReadyToSend --> Sent: Auto send successful
    ReadyToSend --> Sent: Manual send by manager
    ReadyToSend --> Deleted: Conditions no longer met
    ReadyToSend --> ReadyToSend: 25min check failed (stays ready)

    Sent --> [*]
    Deleted --> [*]

    note right of Scheduled
        toBeSentAt set
        Waiting for working hours
        Will retry if 25min fails
    end note

    note right of ReadyToSend
        aiMessage populated
        sentAt = null
        Can be sent manually
    end note
```

### Generation Types

```mermaid
flowchart LR
    subgraph AUTOMATIC["AUTOMATIC (system-initiated)"]
        A1[System finds qualifying enquiry]
        A2[AI assesses need]
        A3[System generates message]
        A4[System sends if all conditions OK]
    end

    subgraph ON_DEMAND["ON_DEMAND (user-initiated)"]
        B1[Manager clicks 'Generate']
        B2[System generates message]
        B3[Manager reviews and edits]
        B4[Manager decides when to send]
    end

    A1 --> A2 --> A3 --> A4
    B1 --> B2 --> B3 --> B4
```

---

## FAQ

### Why wasn't the follow-up generated?

Check these conditions in order:
1. Is RFP/DE status correct? (SENT/ACTIVE for RFP, NEW/IN_PROGRESS for DE)
2. Is enquiry not archived/deleted?
3. Was enquiry created within last 3 months?
4. Is event date in the future?
5. Does a manager message exist?
6. Is the last manager message NOT a follow-up?
7. Is last message between 30 hours and 15 days old?
8. Was this enquiry not assessed in the last 12 hours?

### Why wasn't the follow-up sent automatically?

Check in order:
1. Is `Automatic AI Follow-ups` setting enabled for this manager?
2. Was it during 9:00-17:00 in venue's timezone?
3. Did the client receive another auto-followup in the last 25 minutes?

### Why was the follow-up deleted?

For RFP:
- 15 days passed without manager message
- RFP marked as successful or unsuccessful
- RFP has declined custom offer
- RFP was accepted
- Event is less than 7 days away
- Client wrote a new message

For Direct Enquiry:
- 15 days passed without manager message
- DE marked as confirmed or lost
- Event is less than 7 days away
- Client wrote a new message

### What happens if 25-minute check fails?

**It depends on when the follow-up was generated:**

1. **Outside working hours** → Follow-up was postponed (has `toBeSentAt`) → Will be retried by `SendPostponedFollowUpsCommand` → **This is a DELAY**

2. **Inside working hours** → Follow-up stays as "ready" → Manager can send manually → **Or wait for new follow-up in next cycle**

### How many follow-ups can a client receive daily?

- Max 1 per 25 minutes automatically (anti-spam)
- Max 1 per RFP/DE per 24 hours (cooldown)
- Only during 9:00-17:00 (working hours)
- Theoretically many, but these limits prevent spam

### Can a manager disable auto-followups?

Yes - setting `Automatic AI Follow-ups` to OFF:
- Follow-ups will still be generated
- But will NOT be sent automatically
- Manager must review and send manually

---

## Key Source Files

| File | Responsibility |
|------|----------------|
| `src/AI/FollowUp/Query/RfpIdsForFollowUpAssessmentQuery.php` | 13 conditions for RFP qualification |
| `src/AI/FollowUp/Query/DirectEnquiriesForFollowUpAssessmentQuery.php` | 11 conditions for DE qualification |
| `src/AI/FollowUp/Service/FollowUpAssessmentTracker.php` | 12-hour assessment lock |
| `src/AI/FollowUp/Service/AutomaticAIFollowUpSenderService.php` | 5 conditions for automatic sending |
| `src/AI/FollowUp/Listener/SendAIFollowUpAutomaticallyOrPostponeListener.php` | Working hours check + postponement |
| `src/AI/FollowUp/Service/WorkingHoursCalculator.php` | 9:00-17:00 working hours |
| `src/AI/FollowUp/Specification/ClientRecentlyReceivedAiAutoFollowUpQuery.php` | 25-minute anti-spam check |
| `src/AI/FollowUp/Console/DeleteObsoleteAIFollowUpsCommand.php` | 7 RFP + 5 DE deletion conditions |
| `src/AI/FollowUp/ValueObject/CooldownStatus.php` | 24-hour cooldown |
| `src/AI/FollowUp/Console/SendPostponedFollowUpsCommand.php` | Retries postponed follow-ups |
