# Lark Base tables — what each one is for

Three tables were added to **CKP Planning Sheet R&D - Snapshot 02.09**. They are not three copies of the same data. Each answers a different question, and mixing them up is the main way this setup can go wrong.

| Table | Answers | Grain | Records |
|---|---|---|---|
| `Payroll_Calendar_Events` | "What must I do today?" | client × step × month | 1,452 |
| `Payroll_Calendar_Period` | "Whose payroll cycle is running right now?" | client × payroll period | 294 |
| `AP_Calendar_each-day` | "Which AP runs land today?" | client × run date | 814 |

---

## `Payroll_Calendar_Events`

The operational table. One record per client, per payroll step, per month, each a single-day event.

Event titles read `Client - Step`, e.g. `KIA Asia Pacific Sdn. Bhd. - Payroll Input`, `11 Dali Sdn. Bhd. - Salary Payment`. Colour is set by step, so a month view shows the shape of the cycle at a glance: input and processing early, review and client approval mid-cycle, salary payment at the end.

**This is the only payroll table the automations may fire from.** Every notification needs three things a step-level record has and a period-level record does not: one deadline, one assignee, and one `Task Status` to check. `Input_Reminder`, `Process`, `Due-day push` and `overdue escalation` all read this table.

Views:
- **Grid** — the working list. Currently filtered to a single client (see README known issues).
- **Calendar** — every deadline in the firm, by day.
- **Gantt** — step sequence per client.

Key fields: `Event Title`, `Step`, `Client`, `Payroll Period`, `Start Date`, `End Date`, `Assignee`, `Role`, `Team Lead`, `Relationship Manager`, `Payment Mode`, `Headcount`, `Task Status`, `Description`, `Weekend Flag`.

`Start Date` and `End Date` hold the same value — these are point-in-time deadlines, not spans. The pair exists so the calendar and Gantt views bind cleanly.

## `Payroll_Calendar_Period`

The capacity table. One record per client per payroll period, spanning from that client's first milestone to their salary payment date.

Event titles read `Client (YYYY-MM)`, e.g. `CTP Infotech Malaysia Sdn Bhd (2026-09)`. On the calendar these render as horizontal bars, so the month view shows how many clients' cycles are open on any given day and where they stack up. In September 2026 the bars pile up heavily in the last week — that is the crunch window, and it is invisible in the Events table because there each deadline looks like an isolated dot.

Use it for: workload planning, spotting weeks where one PIC has four cycles running at once, and answering a client's "where are you up to on ours?"

**Do not build automations on it.** A period record has no single owner and no single deadline, so any notification fired from it would be addressed to nobody in particular about a deadline that has not arrived.

Key fields: `Event Title`, `Client`, `Payroll Period`, `Start Date`, `End Date`, `Milestones` (all five dates as a compact string), `Assignee`, `Team Lead`, `Relationship Manager`, `Payment Mode`, `Headcount`.

**Maintenance warning:** this table was imported separately and is not linked to `Payroll_Calendar_Events`. Editing a date in Events does not update the matching Period bar. Both are regenerated together by `02_expand_payroll_dates.py`, so treat the scripts as the source of truth and re-import both whenever either changes. If someone starts editing dates directly in Lark, the two tables will silently diverge.

## `AP_Calendar_each-day`

The AP equivalent of Events. One record per client per run date, single-day events titled `Client - Step`.

AP has no milestone chain the way payroll does. The workbook's five AP date columns are empty for every client; what exists is a repeat cadence in `AP Remarks`, which the scripts expand into one record per occurrence — hence "each-day". Most clients produce a single `AP Run` per occurrence; NS0 Malaysia is the exception, splitting into `AP Listing Update` (Mon/Thu) and `AP Bank Upload` (Tue/Fri).

**There is deliberately no AP Period table.** A period bar is only meaningful when a client's work is a sequence with a start and an end. AP runs are independent point events, so a bar from the first to the last run of the month would just be a bar across the whole month.

The calendar view makes the volume problem obvious: Framework Studio, Nu Best and Oneness Concept Wellness appear on every single weekday, and together account for roughly 390 of the 814 records. Weekly clients (Arctic Green and DKM Kids on Wednesdays, Ez Fundamental and Jobbie on Fridays, Wunder Group on Tue/Thu) sit on top of that, and the 15th and month-end are the heaviest days once the mid-month clients land.

Unlike the payroll calendar, no AP events fall on a weekend — every cadence resolves to a weekday by construction.

`Due-day_push` and `Overdue` fire from this table.

Key fields: `Event Title`, `Step`, `Client`, `Start Date`, `End Date`, `Cadence Rule`, `Assignee`, `PIC`, `Team Lead`, `Team`, `AP Software`, `Task Status`, `Description`, `Assumption Flag`.

---

## Which table for which job

| You want to | Use |
|---|---|
| Send a reminder or escalation | `Payroll_Calendar_Events` / `AP_Calendar_each-day` |
| See one person's workload this week | Events / each-day, filtered by `Assignee` |
| See which clients are mid-cycle | `Payroll_Calendar_Period` |
| Find the crunch weeks before assigning new work | `Payroll_Calendar_Period` |
| Check whether a deadline was met | Events / each-day, `Task Status` |
| Tell a client where their payroll stands | `Payroll_Calendar_Period`, then Events for the specific step |
