# CKP Payroll & AP — Lark Notification Workflow Spec

Source tables:
- `CKP_Payroll_Calendar_Workflow.csv` — 1,452 events, 49 clients, 2026-09 to 2027-02
- `CKP_AP_Calendar_Workflow.csv` — 862 events, 32 clients, same period

Both tables use the same workflow fields, so the two automation sets are structurally identical.

---

## 1. Field setup (do this before building any automation)

| Field | Type | Notes |
|---|---|---|
| `Start Date` | Date | Drives the calendar view and the due-day triggers |
| `Task Status` | Single select | `Not Started` / `In Progress` / `Done` / `Blocked` — default `Not Started` |
| `Completed On` | Date | Filled by the PIC when marking `Done` |
| `Assignee` | Member | Must be converted from text to a Lark member field, otherwise messages cannot be routed |
| `Escalation Owner` | Member | Team Lead — same conversion required |
| `Critical` | Single select | `YES` for `Salary Payment` and `AP Bank Upload` |
| `Status` (AP only) | Single select | `CONFIRMED` / `TBC` |

All advance reminders use Bitable's built-in "N days before" offset on `Start Date`. The offset counts calendar days, so a reminder whose offset lands on a Saturday or Sunday will still fire that day. There is no pre-computed reminder column — `Start Date` is the only date field the automations trigger on.

Because one automation carries one offset value, steps with different lead times need separate automations (see P1a / P1b below).

---

## 2. Payroll automations

### P1a — Advance reminder (Payroll Input)
- **Trigger:** `Start Date`, 2 days before, 09:00
- **Condition:** `Step` = `Payroll Input` AND `Task Status` is not `Done`
- **Action:** Send Lark message to `Assignee`
- **Message:**
  > **[Payroll] Due {Start Date} — {Client}**
  > Step: {Step} · Payroll period: {Payroll Period}
  > {Description}
  > Mark this record `In Progress` once you start.

### P1b — Advance reminder (all other steps)
- **Trigger:** `Start Date`, 1 day before, 09:00
- **Condition:** `Step` is not `Payroll Input` AND `Task Status` is not `Done`
- **Action:** Send Lark message to `Assignee`
- **Message:** same template as P1a.

### P2 — Due-day push
- **Trigger:** `Start Date` arrives, 08:30
- **Condition:** `Task Status` is not `Done`
- **Action:** Send Lark message to `Assignee`
- **Message:**
  > **[Payroll] DUE TODAY — {Client} · {Step}**
  > Payroll period: {Payroll Period} · Headcount: {Headcount}
  > {Description}

### P3 — Same-day overdue escalation
- **Trigger:** `Start Date` arrives, 17:30
- **Condition:** `Task Status` is not `Done`
- **Action:** Send Lark message to `Assignee` and `Escalation Owner`
- **Message:**
  > **[Payroll] OVERDUE — {Client} · {Step}**
  > Due {Start Date}, still `{Task Status}`. Assignee: {Assignee}.
  > Downstream steps in this payroll cycle are now at risk.

### P4 — Salary payment pre-check
- **Trigger:** `Start Date`, 1 day before, 14:00
- **Condition:** `Critical` = `YES`
- **Action:** Send to a Lark group chat `CKP Payroll — Payment Run`
- **Message:**
  > **[Payroll] Salary payment tomorrow — {Client}**
  > Pay date: {Start Date} · Mode: {Payment Mode} · Headcount: {Headcount}
  > Confirm client approval is received and the bank file is ready.
  > RM: {Relationship Manager} · Team Lead: {Team Lead}

### P5 — Weekly team digest
- **Trigger:** Scheduled, every Monday 08:30
- **Condition:** `Start Date` within the next 7 days
- **Action:** Send grouped summary to `CKP Payroll — Theta` group
- **Message:** count by `Assignee`, list of `Client` + `Step` + `Start Date`, plus any records still `Not Started` from last week.

---

## 3. AP automations

### A1 — Due-day push
- **Trigger:** `Start Date` arrives, 08:30
- **Condition:** `Status` = `CONFIRMED` AND `Task Status` is not `Done`
- **Action:** Send Lark message to `Assignee`
- **Message:**
  > **[AP] DUE TODAY — {Client} · {Step}**
  > Cadence: {Cadence Rule} · System: {AP Software}
  > {Description}

### A2 — Bank cut-off reminder
- **Trigger:** `Start Date` arrives, 15:00
- **Condition:** `Critical` = `YES` AND `Task Status` is not `Done`
- **Action:** Send to `Assignee` and `Escalation Owner`
- **Message:**
  > **[AP] Bank cut-off approaching — {Client}**
  > {Step} due today and not yet completed. Upload before the bank cut-off or the payment rolls to the next working day.

### A3 — Overdue escalation
- **Trigger:** `Start Date` arrives, 17:30
- **Condition:** `Status` = `CONFIRMED` AND `Task Status` is not `Done`
- **Action:** Send to `Assignee` and `Escalation Owner`
- **Message:**
  > **[AP] OVERDUE — {Client} · {Step}**
  > Due {Start Date}, still `{Task Status}`.

### A4 — TBC cadence chase
- **Trigger:** Scheduled, first working day of each month 09:00
- **Condition:** `Status` = `TBC`
- **Action:** Send to `Escalation Owner`, grouped by `Team`
- **Message:**
  > **[AP] Cadence still unconfirmed — {Team} team**
  > These clients have no agreed AP schedule; the calendar is running on a placeholder date only:
  > {Client list}
  > Please confirm the listing and payment cadence with the account team and update `Cadence Rule`.

### A5 — Daily-run clients
Framework Studio, Oneness Concept Wellness and Nu Best run every working day and generate roughly 390 events over six months. Excluding them from A1 and sending one consolidated 08:30 message instead keeps the channel usable.

---

## 4. Escalation ladder

| Level | Trigger point | Recipient |
|---|---|---|
| 1 | 1–2 days before due date, 09:00 | Assignee (PIC) |
| 2 | Due date 08:30 | Assignee (PIC) |
| 3 | Due date 17:30, not `Done` | Assignee + Team Lead |
| 4 | Critical step, not `Done` next morning | + Relationship Manager |

Level 4 is not built as an automation yet — it needs a rule for what counts as a payment failure, which should come from Anna.

---

## 5. Open items before go-live

1. **Weekend firing.** 416 Payroll events fall on a weekend (`Weekend Flag`), and the calendar-day offset means advance reminders can land on a weekend too. Both are accepted for now; recipients should expect some Saturday and Sunday messages until the shift rule is agreed.
2. **Public holidays.** No Malaysian public holiday calendar is applied anywhere in the schedule or the triggers.
3. **Mid-month / month-end definitions.** AP `Mid-Month` assumed to be the 15th; `Month-End` assumed to be the last working day. Payroll distinguishes `Month end` from `Last working day` — the two modules need one shared definition.
4. **Assignee field conversion.** Names are currently plain text. They must be mapped to real Lark accounts or no message will route.
5. **Automation run quota.** Roughly 2,300 events over six months, with three to four automations each. Check the workspace's monthly automation run limit before enabling everything at once.
