# CKP Payroll & AP Deadline Calendar

Turns the payroll and accounts-payable schedules buried in CKP's planning workbook into dated Lark Base calendars with automated deadline notifications.

The planning sheet stores each client's schedule as free text — `22th`, `Last working day of the month`, `Mid-Month and Month-End`, `Weekly: Tue & Thu`. Those strings are readable but not actionable: nobody can see the week ahead, and nothing reminds a PIC that a deadline is today.

**Status:** deployed and running.

---

## Deployment

Lark Base: **CKP Planning Sheet R&D - Snapshot 02.09** (CKP → Zaim Syahmi)

The calendar work lives in two sections added alongside the original `Client Listing` / `BD` / `Account Staff Planning` / `Payroll` / `AP` tables.

### `Payroll_Calendar`

| Object | Type | Notes |
|---|---|---|
| `Payroll_Calendar_Events` | Table | One record per client per step per month. Views: **Grid**, **Calendar**, **Gantt** |
| `Payroll_Calendar_Period` | Table | Period-level companion table |
| `Input_Reminder` | Automation | Advance reminder |
| `Process` | Automation | Advance reminder |
| `Due-day push` | Automation | Due-day notification |
| `overdue escalation` | Automation | End-of-day escalation to Team Lead |

### `AP_Calendar`

| Object | Type | Notes |
|---|---|---|
| `AP_Calendar_each-day` | Table | One record per client per AP run date |
| `Due-day_push` | Automation | Due-day notification |
| `Overdue` | Automation | End-of-day escalation to Team Lead |

The two payroll tables are not duplicates. `Payroll_Calendar_Events` answers "what must I do today" — one deadline, one assignee, one status, which is what a notification needs. `Payroll_Calendar_Period` answers "whose cycle is running right now" — the calendar renders each client's month as a horizontal bar, which makes crunch weeks visible in a way a wall of single-day dots cannot. Automations fire from Events only.

AP has no Period table by design: AP runs are independent point events, not a sequence, so a span bar would carry no information.

**See `docs/lark_base_tables.md` for the full breakdown** — grain, fields, views, which table to use for which question, and the maintenance warning about Events and Period drifting apart.

The **Gantt view** is a deployment addition not in the original spec. It reads the same `Start Date` / `End Date` pair as the calendar view and is the better view for seeing where a client's five payroll steps sit relative to each other.


---

## Repo contents

```
data/
  payroll_calendar_workflow.csv   1,452 events · 49 clients · 2026-09 → 2027-02
  ap_calendar_workflow.csv          814 events · 24 clients · same period
  ap_cadence_rules.csv               25 rules · one row per client, for review
  payroll_rules.json                242 rules · month-agnostic (M/22, M/LWD, M+1/7)
docs/
  notification_workflow_spec.md   Field setup, automation configs, message templates
scripts/
  01_extract_payroll_rules.py     Payroll sheet → normalised rule codes + PIC/TL mapping
  02_expand_payroll_dates.py      Rule codes → real dates for a given month range
  03_extract_ap_cadence.py        AP Remarks free text → cadence rules → real dates
  04_add_workflow_fields.py       Adds Task Status, Escalation Owner, Critical, Completed On
```

`data/payroll_rules.json` is the month-agnostic layer. When the six-month window runs out, re-run `02_expand_payroll_dates.py` with new months rather than re-deriving the rules.

---

## Data model

### Payroll — five milestones per client

`Payroll Input` → `Payroll Processing` → `Payroll Review` → `Payroll Submission to Client & Approval` → `Salary Payment`

There is no firm-wide payroll calendar. All five dates are configured per client, so the calendar is built as client × step. Rule codes:

| Code | Meaning |
|---|---|
| `M/17` | Day 17 of the current month |
| `M/EOM` | Last calendar day of the current month |
| `M/LWD` | Last working day of the current month |
| `M+1/7` | Day 7 of the following month |

Assignees are not in the payroll sheet. They are backfilled from `Task - Internal`, taking the most recent `Monthly Payroll` row per client: `PIC` owns Input and Processing, `Team Lead in Charge` owns Review, PIC submits and the client approves. All 49 clients matched.

Salary Payment's executor depends on the engagement, derived from `Payroll Remarks`: `AP` (18 clients, CKP AP team pays), `CLIENT` (16), `POB-STAT` (8), `POB` (3), `CKP-MAKER` (3), `TAX-ONLY` (1).

### AP — cadence, not milestones

The five AP date columns in the workbook are empty for every row. The entire schedule lives in `AP Remarks` as a repeat frequency:

| Remark | Rule type | Clients |
|---|---|---|
| `Mid-Month and Month-End` | `MIDMONTH_EOM` | 12 |
| `Every Wednesday`, `Weekly: Friday`, `Weekly: Tue & Thu` | `WEEKLY` | 7 |
| `Everyday` | `WEEKDAYS` | 3 |
| `Month-End`, `Month-End - together with salary` | `EOM_WD` | 2 |
| `15th & 29th (Prior working day)` | `DAYS_PRIOR_WD` | 1 |
| `Listing: Mon & Thu. Upload: Tue & Fri` | split into two steps | 1 |

Do not read the workbook's `Frequency` column. It says `Monthly` on all 33 rows, including the ones that run every Wednesday.

---

## Automations

Six of the ten automations in the spec were deployed. The spec IDs map to the Lark object names as follows:

| Spec ID | Deployed as | Trigger | Recipient |
|---|---|---|---|
| P1a | `Input_Reminder` | Advance reminder | Assignee |
| P1b | `Process` | Advance reminder | Assignee |
| P2 | `Due-day push` | Due date, morning | Assignee |
| P3 | `overdue escalation` | Due date, end of day, not `Done` | Assignee + Team Lead |
| A1 | `Due-day_push` | Due date, morning | Assignee |
| A3 | `Overdue` | Due date, end of day, not `Done` | Assignee + Team Lead |

Not deployed: `P4` (salary payment pre-check), `P5` (weekly digest), `A2` (bank cut-off reminder), `A4` (TBC cadence chase). Their configurations remain in `docs/notification_workflow_spec.md`.

Message templates are in the same spec file.

---

## Assumptions

These were not defined in the workbook and were chosen to make the dates computable. Each should be confirmed before the next rebuild.

- `Mid-Month` = the 15th
- AP `Month-End` = last working day, because AP involves a bank upload. The payroll sheet distinguishes `Month end` from `Last working day`; the two modules do not currently share one definition.
- `Everyday` = Monday to Friday
- Weekend and public-holiday shifting is **not** applied. The payroll team's position is that shift handling is negotiated per client, so dates are placed literally. The `Weekend Flag` column marks the 416 payroll events affected.

---

## Known issues

**Daily-run clients flood the AP channel.** Framework Studio, Oneness Concept Wellness and Nu Best run every working day and generate roughly 390 of the 814 AP events. With `Due-day_push` firing per record, that is three messages every morning from those clients alone. A consolidated daily digest would be a better fit.

**No payment-specific safeguard.** `Salary Payment` and AP bank upload are the steps where money moves, and they get the same treatment as every other step. `P4` and `A2` were designed for exactly this and were not deployed. A missed payroll approval is not the same class of miss as a late listing update.

**No completion enforcement.** `overdue escalation` and `Overdue` depend on PICs setting `Task Status` to `Done`. If the field is not maintained, both become noise and get muted.

**Weekend notifications.** Two sources: events that fall on a weekend, and calendar-day reminder offsets that land on one. Expect some Saturday and Sunday messages until the shift rule is agreed.

**No public holiday calendar.** Nothing in the schedule or the triggers accounts for Malaysian public holidays.

**Fixed six-month window.** Ends 2027-02. Re-run `02_expand_payroll_dates.py` and `03_extract_ap_cadence.py` before then.

**Eight AP clients out of scope.** NVD Logistics, NVD Asia Logistics Taiwan, Juice Work, Leax Malaysia, Leax Asia (Gamma), Master Jaya Environment (Sigma), Shyunmija, The Lush Clinic (Omega). Their `AP Remarks` records an owner to confirm with rather than a schedule, so no dates could be derived. Excluded from the deployed tables.


---

## Rebuilding the data

```bash
pip install -r requirements.txt
python scripts/01_extract_payroll_rules.py
python scripts/02_expand_payroll_dates.py     # edit `months` to change the window
python scripts/03_extract_ap_cadence.py
python scripts/04_add_workflow_fields.py
```

Scripts expect the planning workbook at the path set at the top of each file. The workbook is deliberately not committed.

To re-import: create the table, import the CSV, set `Start Date` and `End Date` to Date type, convert `Assignee` and `Escalation Owner` to Member fields, set `Task Status` to a single select. Then bind the calendar and Gantt views to `Start Date` / `End Date`.

---

## Confidentiality

The source workbook contains client names, monthly fees, staff assignments and payment arrangements. **Use a private repository.** `.gitignore` excludes `*.xlsx`, but the CSVs in `data/` also carry client names and assignee names — review before sharing outside the firm. The Base itself is marked External, so check who it is shared with before adding anything further to it.
