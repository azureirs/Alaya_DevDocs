---
title: Automation
section: Automation
order: 10
---

# Automation

**Automation** runs a company's routine work for it: reports that are
generated and emailed, purchase orders and stock transfer requests raised
from the reorder advice, item lead times and reorder points kept up to
date, and calls to other systems. Each piece of work is set up once as an
**automation workflow** and then runs on a schedule, when another system
calls its URL, or on demand.

This page describes every part of a workflow. To start a workflow from
your own system, or to receive calls from one, see
[Automation Webhooks](automation_webhooks.md).

## Concepts

| Term | Meaning |
|----|----|
| **Workflow** | A named automation of one company: its trigger, its settings and an ordered list of processes. |
| **Trigger** | What starts the workflow automatically: **None**, **Schedule**, **Webhook** or **Module Editor Record**. Only an **active** workflow is started by its trigger. |
| **Process** | One unit of work in the workflow, such as a System Report or a Webhook. Processes run one after another, in order. |
| **Action** | What a process does with its result: **Send Email** and/or **Alaya Notification**. |
| **Run** | One execution of a workflow. Every run is recorded in the **Automation Log**, with its processes, emails, files and documents. |
| **Business date** | The date a run is for. Relative date filters (for example *Last month*) are worked out from it. |

```
 Trigger ──► Process 1 ──► Process 2 ──► … ──► Process n
 (None /      │ actions:     │
  Schedule /  │ Send Email   │
  Webhook /   │ Notification │
  Module Editor Record)
              ▼              ▼
          Automation Log (run, processes, emails, files, documents)
```

## Screens and access rights

All screens are in the **Automation** module.

| Screen | Access rights | Use |
|----|----|----|
| **Automations** | View, Add, Save / Edit, Delete, Export Grid, Copy | Lists the workflows with their company, trigger, schedule, active flag and last run. |
| **Automation Detail Page** | View, Add, Save / Edit, Copy | Edits one workflow: its settings, trigger and processes, drawn as a diagram or listed in a grid. |
| **Automation Log** | View, Export Grid | Lists the runs. Filter them by date and status, and open a run to see its details. |

A workflow can also be started from a report screen. **New Automation**
on a report format opens a new workflow that already has a System Report
process for that report and format, with the filters as they are on the
screen.

## Workflow settings

| Setting | Description |
|----|----|
| **Automation Name** | Unique within the company. The webhook URL uses it, so changing the name changes the URL. |
| **Description** | Free text. |
| **Company** | The company whose data the workflow uses. A workflow belongs to exactly one company. |
| **Active** | Only an active workflow is started by its trigger. An inactive workflow runs only with **Run Now**. |
| **Trigger** | **None**, **Schedule**, **Webhook** or **Module Editor Record**. Click the trigger node at the start of the diagram to open the **Trigger** dialog, where you choose the trigger and set it up. |
| **Runtime Limit** | The longest a whole run may take, in seconds: 300 by default, at most 86,400 (24 hours). A run that goes over the limit is stopped and marked **TimedOut**. |
| **Effective Date Range** | The trigger starts the workflow only on business dates inside this range. Leave a date blank for no limit. **Run Now** is not limited. |

## Triggers

### None

The workflow runs only when a user clicks **Run Now**.

### Schedule

The workflow runs at a time of day, on the days its frequency allows.

| Frequency | Runs on |
|----|----|
| **Daily** | Every day. |
| **Weekly** | One weekday. |
| **Bi-Weekly** | One weekday every two weeks. The week containing the **anchor date** is the first week. |
| **Monthly** | A day of the month (1–31), or the last day of the month. |
| **Quarterly** | A day in the 1st, 2nd or 3rd month of each quarter. |
| **Yearly** | A day of one month. |

- Times are in the server's time zone, in whole minutes (00:00–23:59).
- A month that does not have the selected day is skipped. To run at
  every month end, select **Last day**.
- A schedule runs a workflow at most once per business date. If a
  scheduled run is delivered again for the same date, it is skipped.
- The trigger dialog previews the next runs.

### Webhook

The workflow runs when its **webhook URL** is called with an HTTPS GET.
The URL carries the client code, company code, automation name and,
optionally, a **secret key**. The trigger dialog shows the URL and copies
it for you. See [Automation Webhooks](automation_webhooks.md#starting-a-workflow-with-its-webhook-url).

### Module Editor Record

The workflow runs when a record of a [Module Editor](module_editor.md)
table is created, updated or deleted.

| Setting | Description |
|----|----|
| **Table** | The Module Editor menu whose records run the workflow, listed as *module / menu*. |
| **Events** | Any of **Created**, **Updated** and **Deleted**. A document menu's record is cancelled rather than deleted; that counts as Deleted. |
| **Runs** | **One run per record** and/or **One run per import**. At least one must be selected. |

Which saves start a run:

| Save | One run per record | One run per import |
|----|----|----|
| The record's detail screen, the list's delete, `api/Udf/ZZTSaveRow` and `api/Udf/ZZTDeleteRow` | One run for that record. | — |
| An import into the table | — | One run for all the records it created, and one for all it updated. |

- A run carries its records: a [Webhook process](#webhook-process) can
  send them with `{{TableName}}`, `{{Event}}`, `{{RecordID}}` and
  `{{RecordIDs}}`, and the Automation Log shows them.
- The run is made by the user who saved the records.
- The trigger watches the table for every company. The workflow's
  processes still run for the workflow's own company.
- To stop a workflow from running itself without end (for example a
  Webhook process whose receiver saves the same table again), a workflow
  is started at most **30 times a minute** by this trigger. Further
  records in that minute don't start it.
- Saving a record never waits for automation or fails because of it. If
  the automation server can't be reached, the records are sent again
  for up to a day.
- The Module Editor's **Module Query** (seed data) does not start the
  trigger.

## Processes

Each process has an **Enabled** switch. A disabled process keeps its
settings but is skipped when the workflow runs, together with its Send
Email and Alaya Notification.

| Process | What it does | Actions |
|----|----|----|
| [System Report](#system-report) | Generates a report as PDF and/or Excel. | Send Email, Alaya Notification |
| [Reorder Advice](#reorder-advice) | Generates purchase orders or stock transfer requests from the reorder advice. | Send Email, Alaya Notification |
| [Lead Time Advice](#lead-time-advice) | Updates items' lead time to the recommended lead time. | Send Email, Alaya Notification |
| [Reorder Point Advice](#reorder-point-advice) | Updates items' reorder point to the recommended reorder point. | Send Email, Alaya Notification |
| [Send Email](#send-email-process) | Sends an email, optionally with the files of the run's earlier processes. | None (it is a Send Email) |
| [Alaya Notification](#alaya-notification-process) | Notifies users that the run reached it. | None (it is an Alaya Notification) |
| [Webhook](#webhook-process) | Sends an HTTP request to another system. | None |
| [Workflow](#workflow-process) | Runs another workflow's processes in its place. | None (the other workflow's processes keep their own) |

### System Report

Runs one of Alaya's reports, the same way the report screen does.

- **Report** and **Report Format**: the active system formats and the
  company's own formats, as on the report screen. Some filters list only
  the formats made for them (for example *Show Project* lists the Project
  formats). **Apply** and **Preview** on the Set Filter page refuse a
  format that the report screen would not list with those filters.
- **Output**: PDF, Excel, or both. The files are kept with the run and
  linked from the Automation Log.
- **Set Filter** opens the report's own filter screen. Its values are
  saved with the process and used on every run.
- **Relative dates.** Date filters are relative to each run's business
  date. A **date range** can be *Today*, *Yesterday*, *Past week / 2 weeks /
  month / quarter / year* (the last 7, 14, 30, 90 or 365 days, ending
  yesterday), *Last week* (Sunday to Saturday), *Last month*, *Last quarter*
  or *Year to date*; the default is *Last month*. A **single date** can be
  *Today*, *Yesterday*, or the end of last week, month or quarter.
- **Field filter** (optional). Filters the report's data on the report
  format's fields after the report query has run, for example *Balance
  greater than 0*. Text fields support *Contains*, *Does not contain*,
  *Starts with*, *Ends with*, *Equals* and *Does not equal*. Number and date
  fields support *Equals*, *Does not equal*, *Less than*, *Greater than*,
  *Less than or equal*, *Greater than or equal* and *Between*. Switched
  off, the rows are kept but not applied.
- **Run report** in the process dialog runs the report with its filters,
  as the report screen does. Nothing is emailed or logged.

### Reorder Advice

Uses the Reorder Advice screen's filters (opened with **Set Filter**) to
find the items to reorder, then generates documents from them.

| Setting | Options |
|----|----|
| **Generate** | **Purchase Order** or **Stock Transfer Request**. A stock transfer request needs a **Reason**, chosen in Set Filter. |
| **Save as** | **Draft** (the default) or **Post**. |
| **Skip items** | *Skip item without preferred vendor* (purchase orders) or *Skip item without preferred warehouse* (stock transfer requests). |
| **Exclude** | *Exclude if PO exists* / *Exclude if STR exists*, as on the Reorder Advice screen. |

- Document lines take the same defaults as the Purchase Order or Stock
  Transfer Request screen.
- All documents are checked before any is saved. If any item cannot be
  generated, the process fails and **nothing** is generated. Matrix and
  package items always fail.
- If saving fails part-way, documents that were already saved stay saved.
  The error message lists them.
- The generated documents are listed in the Automation Log and open from
  there.

### Lead Time Advice

Uses the Lead Time Advice screen's filters. Each run updates the listed
items' lead time to the recommended lead time.

- **Calculate lead time from** *Purchase Order* or *Stock Transfer*.
- **Recommendation method**: *Median*, *Average*, *Longest*, *Shortest* or
  *Latest*.
- **Include if recommended lead time is zero**: those items' lead time is
  cleared as well.
- The analysis period is the 12 months before the run's business date,
  from the first day of that month to the business date.

### Reorder Point Advice

Uses the Reorder Point Advice screen's filters. Each run updates the
listed items' reorder point to the recommended reorder point.

- **Calculate demand from** any of *Sales*, *Delivery Order*, *Stock
  Transfer Out* and *Production Consumption*.
- **Analysis period**: a relative date range, worked out from each run's
  business date.
- **Service level** (percent), **rounding**, and whether to include items
  with zero lead time or zero demand.

### Send Email process

The [Send Email](#send-email) action as a process of its own, for an
email at a chosen point of the workflow, for example after several
reports. Its settings are the action's, with these differences:

- **Attach** attaches the PDF and Excel files that the run's **earlier**
  processes made (the files a Webhook process's *Send outputs* would
  link). With none, the email is sent without attachments.
- There is no **Send by** and no **Send when empty**: it has no report.
- `{{ReportName}}` is the workflow's name, so the default subject is
  *"{workflow} - {business date}"*. The default body says that the
  workflow reached this process for the business date, and that the
  files are attached when there are any. It is also `{{Summary}}`.
- A failed email fails the process and stops the run, unless
  [Continue the run if this process fails](#when-a-send-email-or-alaya-notification-process-fails)
  is checked.

### Alaya Notification process

The [Alaya Notification](#alaya-notification) action as a process of its
own. The selected users are told *"Process N (Alaya Notification) of
{workflow} reached for {business date}."*, and the notification opens
the run in the Automation Log.

- At least one user is required.
- A notification that cannot be sent fails the process and stops the
  run, unless *Continue the run if this process fails* is checked. As an
  action, it would only be marked failed.

#### When a Send Email or Alaya Notification process fails

| Continue the run if this process fails | The process | The run |
|----|----|----|
| Unchecked (the default) | Fails. | Stops. |
| Checked | Passes. Its email or notification is marked failed, with the error, in the Automation Log. | Goes on with the next process. |

A stop from the Automation Log, or the runtime limit, ends the run
either way.

### Webhook process

Sends one HTTP request to another system and passes when the answer is a
2xx status. Any other status, a connection error or no answer in time
fails the process. See
[Automation Webhooks](automation_webhooks.md#receiving-calls-from-a-webhook-process)
for the request format.

| Setting | Description |
|----|----|
| **Method** | GET, POST, PUT, PATCH or DELETE. A GET has no body. |
| **URL** | A full `http://` or `https://` address. It may contain placeholders. |
| **Headers** | One `Name: value` per line, for example an API key. Headers are saved encrypted. |
| **Content type** | JSON (`application/json`) or text (`text/plain`). |
| **Body** | Not used with GET. A JSON body must be valid JSON when you save it. |
| **Send outputs** | Adds links to the PDF and Excel files made by the run's earlier processes. |
| **Timeout** | 1–300 seconds, 30 by default. It is also capped by the time left in the workflow's runtime limit. |

### Workflow process

Runs the processes of another workflow of the same company in its place,
each with its own Send Email and Alaya Notification.

- The other workflow's own **Active** flag, trigger and runtime limit are
  ignored. The run uses the calling workflow's settings.
- A workflow cannot run itself, directly or through other workflows. The
  picker hides any workflow that would create such a loop, and saving
  checks again.
- A workflow that another workflow runs cannot be deleted.

## Actions

### Send Email

| Setting | Description |
|----|----|
| **To / Cc / Bcc** | One or more addresses, separated by commas or semicolons. |
| **Send by** | *Customer* or *Vendor*, offered when the report can be filtered by that party. Each customer or vendor in the report gets their own email with their own report, instead of the To addresses. The Automation Log shows the outcome per recipient. |
| **Subject / Body** | Leave blank to use the report's **Report email template** from Email Account Setting, or else the defaults: subject `{{ReportName}} - {{BusinessDate}}`, body *"Please find the {{ReportName}} report attached."* |
| **Attach** | The PDF and/or Excel file. |
| **Send when empty** | Off by default: a report with no data is not emailed, and the Automation Log shows the email as skipped. |

The subject and body can contain these placeholders, written as
`{{Name}}` or `{Name}`:

| Placeholder | Value |
|----|----|
| `{{ReportName}}` | The report's name (for an advice process, the process type's name). |
| `{{BusinessDate}}` | The run's business date, `yyyy-MM-dd`. |
| `{{CompanyID}}`, `{{ClientCode}}` | The workflow's company ID and client code. |
| `{{RecipientCode}}`, `{{RecipientName}}` | With Send by: the customer's or vendor's code and name. |
| `{Customer_Name}`, `{Customer_Attention}`, `{Customer_Salutation}` | The Report email template's customer placeholders (filled with Send by). |
| `{Sender_Company_Name}`, `{Sender_Contact_Number}` | The company's name and phone number. |
| `{{Summary}}` | Advice processes: what was generated or updated. It is also their default body. |

A Reorder Advice, Lead Time Advice or Reorder Point Advice email is not
sent when nothing was generated or updated.

### Alaya Notification

The selected users get an Alaya notification (the bell) when the process
finishes or fails. The notification opens the run in the Automation Log.
Only users with access to the workflow's company can be selected.

## Runs

### Starting a run

| Trigger source | Started by | Limits |
|----|----|----|
| **Manual** | **Run Now** on the workflow screen. | Any workflow, active or not, on any date. |
| **Schedule** | The workflow's schedule. | Active workflows with the Schedule trigger, inside the effective date range. |
| **Webhook** | A call of the workflow's webhook URL. | Active workflows with the Webhook trigger, inside the effective date range. |
| **Module Editor Record** | A record saved in the trigger's Module Editor table. | Active workflows with the Module Editor Record trigger, inside the effective date range, at most 30 runs a minute. |

Runs are queued and run in the background, so a run can start a little
after it was requested. A run has no logged-in user, so its reports
format dates the way the workflow's creator has set them.

The Automation Log shows who made each run: the user who clicked
**Run Now**, or, for a Module Editor Record run, the user who saved the records.

### Run status

| Status | Meaning |
|----|----|
| **Queued** | Waiting to start. |
| **Running** | In progress. |
| **Success** | Every enabled process finished. |
| **Failed** | A process failed. The processes after it did not run. |
| **Partial** | Shown for a failed run that still sent at least one email. |
| **Cancelled** | Stopped by a user. |
| **TimedOut** | Went over the workflow's runtime limit. |
| **Skipped** | Shown for a process or email that did not need to run, for example an empty report with Send when empty off. |

### Stopping a run

**Stop** on a running run's log detail marks it **Cancelled** at once.
A report or email already in progress finishes first; the processes
after it are not run.

### Automation Log detail

Each run shows:

- **Run Snapshot**: the workflow as it was when the run started (trigger,
  schedule, effective dates, runtime limit) and each process's report,
  format, outputs, email settings, notified users and filters, with
  relative dates already worked out.
- **Execution Processes**: each process's status, timing and error. For a
  Webhook process this includes the answer's status and the start of its
  body.
- **Per-recipient Outcomes**: each email recipient with Send by.
- **Files and documents**: the PDF and Excel files, and the purchase
  orders and stock transfer requests the run generated, each with a link.

## Automation credit

When usage control is switched on for the client, each run's processing
time is charged to the client's **automation credit**:

- A run that starts with no credit fails before any process runs, and is
  not charged.
- Credit is checked between processes. A run whose credit runs out stops
  before its next process.
- A webhook call is refused while there is no credit (see
  [responses](automation_webhooks.md#responses)).

## Rules at a glance

- A workflow belongs to one company, and its name is unique within that
  company.
- Only the selected trigger's settings are saved. Switching from Webhook
  to Schedule discards the secret key.
- Every trigger needs the workflow to be **Active** and inside its
  **Effective Date Range**. Run Now needs neither.
- The processes of a run run one at a time, in order. The first failure
  stops the run.
- A whole run is never retried automatically, because that would repeat
  emails and documents that were already sent or saved. Run it again with
  **Run Now** instead.
