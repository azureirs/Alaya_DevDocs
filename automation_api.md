---
title: Automation API
section: Automation
order: 30
---

# Automation API

The `automation` routes of the [Alaya REST API](api_access.md) list, read, create and run
[automation](automation.md) workflows, and read their runs from the Automation Log. A workflow is
created exactly as the Automation screen saves it, with the same checks.

All routes are under `{ApiUrl}api/automation/` and need the API token of a user
(see [API Access](api_access.md)). The workflow is made by, and runs as, that user. Every reply is:

```json
{ "success": true, "msg": "", "data": { } }
```

`success` is `false` when the request is refused, and `msg` says why.

`companyID` picks the company. Without it, the token's default company is used.

## Endpoints

| Method and route | What it does |
|---|---|
| `GET workflows?companyID=` | The company's workflows: `WorkflowID`, `Name`, `Description`, `Active_YN`, `TriggerType`, `LastRunBusinessDate`. |
| `GET workflows/{id}?companyID=` | One workflow with its trigger and processes, in the shape `POST workflows` takes (below). |
| `POST workflows` | Creates a workflow. The reply's `data` is `{ WorkflowID, Schedule }`. |
| `POST workflows/{id}/run` | Queues a run now, as **Run Now** does. Body `{ "CompanyID": 5, "BusinessDate": "2026-11-12T00:00:00" }`; `BusinessDate` is optional (today). The reply's `data` is `{ HangfireJobID }`. |
| `GET workflows/{id}/schedule?companyID=` | The workflow's scheduled job and its next runs. |
| `POST workflows/{id}/schedule` | Registers the schedule again (after it was saved directly in the database). Body `{ "CompanyID": 5 }`. |
| `GET runs?companyID=&workflowID=&status=&dateFrom=&dateTo=&top=` | Runs, newest first, as the Automation Log lists them. `top` is 1 to 500 (default 50). The dates filter the business date. |
| `GET runs/{runId}?companyID=` | One run as the Automation Log detail shows it: `Run`, `Steps`, `Actions` (emails and notifications) and `Outputs`. |

A run's status (`AE_DisplayStatus`) is one of `Queued`, `Running`, `Success`, `Failed`,
`Partial` (failed after sending at least one email), `Cancelled`, `TimedOut` or `Skipped`.
A PDF or Excel output's `AO_FilePath` is the file's link. A Purchase Order or Stock Transfer
Request output's `AO_FilePath` is the document's ID, with its number in `AO_DocNo`.

A run is queued, so `POST workflows/{id}/run` returns before the run starts. Find the run with
`GET runs?workflowID=`.

## The workflow

```json
{
  "Name": "Monthly sales to manager",
  "Description": "",
  "Active_YN": false,
  "TriggerType": "Schedule",
  "Schedule": { "Frequency": "Monthly", "TimeOfDay": "08:00:00", "DayOfMonth": 1 },
  "RuntimeLimitSeconds": 300,
  "EffectiveFrom": null,
  "EffectiveTo": null,
  "Steps": [
    {
      "StepType": "SystemReport",
      "Enabled_YN": true,
      "Configuration": { "ReportKey": "...", "StoredProcedureName": "...", "ReportFormatID": 123, "Pdf_YN": true, "Excel_YN": false, "Filters": { } },
      "Actions": [
        {
          "ActionType": "SendEmail",
          "Sequence": 1,
          "Active_YN": true,
          "Configuration": { "To": "manager@example.com", "AttachPdf_YN": true, "IsHtml": true }
        }
      ]
    }
  ]
}
```

| Property | Rule |
|---|---|
| `Name` | Required. |
| `Active_YN` | The trigger starts the workflow only while it's active. **Run Now** and `POST workflows/{id}/run` run it either way. Create it inactive until it has been checked. |
| `TriggerType` | `None`, `Schedule`, `Webhook` or `ModuleEditorRecord`. Only that trigger's settings are kept. |
| `RuntimeLimitSeconds` | 1 to 86400. Default 300. A run is stopped when it takes longer. |
| `EffectiveFrom`, `EffectiveTo` | Optional. The first and last business dates the trigger may run on. |
| `Steps` | At least one. They run in list order. A step with `Enabled_YN` false keeps its settings and is skipped. |

`GET workflows/{id}` also returns `WorkflowID` and `CompanyID`, and each step's `Sequence`.
`POST workflows` ignores them: it always creates a new workflow in the request's company, and
numbers the steps in list order.

### Triggers

**Schedule**: `Schedule` is

| Property | Rule |
|---|---|
| `Frequency` | `Daily`, `Weekly`, `Bi-Weekly`, `Monthly`, `Quarterly` or `Yearly` |
| `TimeOfDay` | `"HH:mm:00"`, in the API server's time zone |
| `Weekday` | Weekly and Bi-Weekly: 0 (Sunday) to 6 (Saturday) |
| `AnchorDate` | Bi-Weekly: it runs on the first `Weekday` on or after this date, then every 14 days |
| `DayOfMonth` | Monthly, Quarterly, Yearly: 1 to 31, unless `IsLastDayOfMonth` |
| `IsLastDayOfMonth` | Monthly, Quarterly, Yearly: run on the month's last day |
| `QuarterMonthGroup` | Quarterly: 1 (Jan, Apr, Jul, Oct), 2 (Feb, May, Aug, Nov) or 3 (Mar, Jun, Sep, Dec) |
| `Month` | Yearly: 1 to 12 |

An active workflow with this trigger is scheduled as soon as it's created. The reply's
`Schedule.NextRuns` shows its next runs.

**Webhook**: `Webhook` is `{ "SecretKey": "..." }` (optional, up to 200 characters). The webhook
URL is shown on the Automation screen. `GET workflows/{id}` returns the key masked as `********`.
Don't send the masked value back.

**ModuleEditorRecord**: `ModuleEditorRecord` is

```json
{ "TableName": "Vehicle", "Events": ["Created", "Updated"], "PerRecord_YN": true, "Batch_YN": false }
```

`TableName` is the [Module Editor](module_editor_api.md) menu code. `Events` are `Created`,
`Updated` and `Deleted`. `PerRecord_YN` starts a run for each record saved on its own, and
`Batch_YN` one run for each import. At least one of the two must be set.

### Processes (steps)

| StepType | Configuration |
|---|---|
| `SystemReport` | The report, its format, outputs and filters |
| `ReorderAdvice` | `{ Settings, FilterState, FilterDisplay }` |
| `LeadTimeAdvice` | `{ Settings, FilterState, FilterDisplay }` |
| `ReorderPointAdvice` | `{ Settings, FilterState, FilterDisplay }` |
| `Webhook` | The HTTP request |
| `Workflow` | `{ "WorkflowID": 7 }` |
| `SendEmail` | `{ "ContinueOnFailure_YN": false }`, and exactly one active `SendEmail` action |
| `AlayaNotification` | `{ "ContinueOnFailure_YN": false }`, and exactly one active `AlayaNotification` action |

Any other process may have a `SendEmail` action (`Sequence` 1, as the Automation screen saves it)
and an `AlayaNotification` action (`Sequence` 2).

**Report and advice processes** need settings that the Automation screen's filter pages build
(the report's filters as its stored procedure's parameters, the advice screens' filters). Copy
them from a workflow made on the Automation screen (`GET workflows/{id}`) rather than writing
them by hand. A `SystemReport` needs `ReportKey`, `StoredProcedureName`, `ReportFormatID` and at
least one of `Pdf_YN` and `Excel_YN`. A filter value can be a relative date, which each run fills
in from its business date: `{{AutomationDateFrom:LastMonth}}` and `{{AutomationDateTo:LastMonth}}`.
Ranges: `Today`, `Yesterday`, `Past7Days`, `Past14Days`, `Past30Days`, `Past90Days`, `Past365Days`,
`LastWeek`, `LastMonth`, `LastQuarter`, `YearToDate`, `Tomorrow`, `NextWeek`, `ThisMonth`,
`NextMonth`, and `DayOfLastMonth7`, `DayOfThisMonth15`, `DayOfNextMonth1` (any day 1 to 31).

**Webhook**:

| Property | Rule |
|---|---|
| `Method` | `GET`, `POST` (default), `PUT`, `PATCH` or `DELETE` |
| `Url` | A full `http://` or `https://` address |
| `Headers` | Optional. One `Name: value` per line. Saved encrypted. `GET workflows/{id}` never returns them; it returns `HeadersSet` instead. |
| `ContentType` | `application/json` (default) or `text/plain` |
| `Body` | Not with `GET`. A JSON body must be valid JSON. |
| `TimeoutSeconds` | 1 to 300. Default 30. |
| `SendOutputs_YN` | Adds the links of the files the earlier processes made: an `outputs` array in a JSON object body, or one link per line after a text body. |

The URL, headers and body may use `{{WorkflowName}}`, `{{BusinessDate}}`, `{{RunID}}`,
`{{ClientCode}}`, `{{CompanyID}}`, and in a run started by a Module Editor Record trigger
`{{TableName}}`, `{{Event}}`, `{{RecordID}}` and `{{RecordIDs}}`.

**Workflow** runs another workflow of the same company. A workflow can't run itself, directly or
through the workflows it runs.

**SendEmail and AlayaNotification processes** fail the run when their email or notification
fails, unless `ContinueOnFailure_YN`.

### Actions

**SendEmail**:

| Property | Rule |
|---|---|
| `To`, `Cc`, `Bcc` | Addresses separated by `,` or `;`. `To` is required unless `SendBy` is set. |
| `SubjectTemplate`, `BodyTemplate` | Optional. Without them the report's email template in Email Account Setting is used, else `{{ReportName}} - {{BusinessDate}}`. |
| `IsHtml` | The body is HTML. The Automation screen always sets it. |
| `AttachPdf_YN`, `AttachExcel_YN` | Attach the process's files. On a `SendEmail` process: the files of the earlier processes. |
| `SendBy` | Report processes only: `Customer` or `Vendor` emails each customer or vendor its own report. It needs the report's Send by filter, which the report-filter page sets. |
| `SendWhenEmpty_YN` | Send even when the report has no rows. |

Leave `SenderEmail`, `SenderPassword`, `SmtpHost` and `SmtpPort` out: the company's email settings
are used when the run sends.

**AlayaNotification**: `{ "UserIDs": [1, 2] }`, the users told when the process finishes.

An action with `Active_YN` false keeps its settings and doesn't run.

## Related Pages

- [Automation](automation.md)
- [Automation Webhooks](automation_webhooks.md)
- [Module Editor API](module_editor_api.md)
- [Alaya API](api_access.md)
