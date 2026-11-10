---
title: Automation Webhooks
section: Automation
order: 20
---

# Automation Webhooks

An [automation workflow](automation.md) can be linked to your own system
in two directions:

| Direction | Feature | Use it to… |
|----|----|----|
| **Your system → Alaya** | The **Webhook trigger**: the workflow's webhook URL. | Start a workflow when something happens on your side. |
| **Alaya → your system** | The **Webhook process**: an HTTP request sent during a run. | Tell your system that a run reached a point, and give it the run's files. |

```
  Your system                                   Alaya
  ───────────                                   ─────
  GET  …/api/automation/webhook?c=…&co=…&aw=…  ─►  queues a run (202)
                                                    │
                                                    ▼  Process 1 … n
  POST https://your-host/hook   ◄─────────────────  Webhook process
  (headers, body, placeholders, outputs)
  answer 2xx ───────────────────────────────────►   process passes
```

## Starting a workflow with its webhook URL

### Setting it up

1. Open the workflow and click the trigger node at the start of the
   diagram. In the **Trigger** dialog, set **Trigger** to **Webhook**.
2. Optionally enter a **Secret Key**.
3. Copy the **Webhook URL** from the dialog, and click **Update**.
4. Tick **Active** and save the workflow. The URL starts only the
   **saved** workflow, so save again after changing its name or secret
   key.

### Request

```
GET https://{alaya web}/api/automation/webhook?c={client code}&co={company code}&aw={automation name}&sk={secret key}
```

| Parameter | Required | Description |
|----|----|----|
| `c` | Yes | The client code. |
| `co` | Yes | The company code of the workflow's company. |
| `aw` | Yes | The automation name, URL-encoded (for example `Daily%20Sales%20Report`). |
| `sk` | When set | The trigger's secret key. Leave it out when the trigger has none. |

- Only **HTTPS** is accepted.
- The call **queues** a run and answers at once. It does not wait for the
  run to finish. Follow the run in the **Automation Log**, where its
  trigger source is *Webhook*.
- The run's business date is the day of the call.

### Responses

Every response body has the same shape:

```json
{ "Result": true, "Message": "The automation run has been queued." }
```

| Status | `Result` | Meaning |
|----|----|----|
| **202 Accepted** | `true` | The run has been queued. |
| **400 Bad Request** | `false` | `c`, `co` or `aw` is missing. |
| **403 Forbidden** | `false` | The call was not made over HTTPS. |
| **404 Not Found** | `false` | The client, company or workflow was not found, the workflow's trigger is not Webhook, or `sk` is wrong. These cases cannot be told apart, so a caller without the key learns nothing. |
| **409 Conflict** | `false` | The workflow is inactive, outside its effective date range, or the client has no automation credit left. |
| **503 Service Unavailable** | `false` | The run could not be queued. Try again later. |
| **500 Internal Server Error** | `false` | An unexpected error. |

### Example

```bash
curl -i "https://{alaya web}/api/automation/webhook?c=C0001&co=HQ&aw=Daily%20Sales%20Report&sk=my-secret"
```

### Security notes

- **Treat the whole URL as a secret.** Anyone who has it can start the
  workflow. Always set a secret key. Without one, the URL needs only the
  client code, company code and name.
- **Don't paste the URL into chats or emails.** Link previews and email
  scanners open links automatically, and opening the URL starts a run.
- **Change the secret key if the URL leaks**, then save the workflow. The
  old URL stops working at once.
- A run never repeats on its own. If you call the URL twice, two runs are
  queued.
- In a run started by its webhook URL, a
  [Webhook process](#receiving-calls-from-a-webhook-process) that calls
  that same URL (a GET with the same `c`, `co` and `aw`, on any host) is
  not sent, and passes. Otherwise each run would start the next one. Its
  log says it was skipped. In a run started any other way, the call is
  sent and queues one more run.

## Receiving calls from a Webhook process

A **Webhook process** sends one HTTP request each time the workflow
reaches it. Settings are described in
[Automation](automation.md#webhook-process).

### The request

| Part | What Alaya sends |
|----|----|
| **Method** | GET, POST, PUT, PATCH or DELETE, as set. A GET has no body. |
| **URL** | As set, with its placeholders filled in and URL-encoded. |
| **Headers** | The headers you set (for example `Authorization: Bearer …` or `X-Api-Key: …`), with placeholders filled in. |
| **Content-Type** | `application/json` or `text/plain` (UTF-8). |
| **Body** | The body you set, with placeholders filled in, plus the run's outputs when **Send outputs** is on. |

### Placeholders

URL, headers and body can contain these placeholders:

| Placeholder | Value |
|----|----|
| `{{WorkflowName}}` | The automation name. |
| `{{BusinessDate}}` | The run's business date, `yyyy-MM-dd`. |
| `{{RunID}}` | The run's execution ID, as shown in the Automation Log. |
| `{{ClientCode}}` | The client code. |
| `{{CompanyID}}` | The workflow's company ID. |
| `{{TableName}}` | [Module Editor Record](automation.md#module-editor-record) runs: the Module Editor table (`UT_TableName`). Empty in other runs. |
| `{{Event}}` | Module Editor Record runs: `Created`, `Updated` or `Deleted`. |
| `{{RecordID}}` | Module Editor Record runs: the record's ID (`ZZC_ID`); the first record's in a run for an import. |
| `{{RecordIDs}}` | Module Editor Record runs: all the records' IDs, comma-separated (for example `12,13,14`). |

How a value is inserted depends on where the placeholder is:

| In | Encoding |
|----|----|
| URL | URL-encoded. |
| JSON body | JSON-escaped, **without** quotes. Put text placeholders inside quotes (`"{{WorkflowName}}"`). Numbers can be bare (`{{RunID}}`). |
| Text body, headers | As is. |

A JSON body is checked when the workflow is saved, with every
placeholder replaced by `0`. So `{"run": {{RunID}}}` is accepted.

### Outputs

With **Send outputs** on, the request carries links to the PDF and Excel
files that the run's **earlier** processes made:

- **JSON body**: an `outputs` array is added to the body, so the body must
  be a JSON object (`{ … }`) or empty.
- **Text body**: one link per line, after the body.

Example: the body

```json
{ "event": "daily-sales", "workflow": "{{WorkflowName}}", "date": "{{BusinessDate}}", "run": {{RunID}} }
```

is sent as:

```json
{
  "event": "daily-sales",
  "workflow": "Daily Sales Report",
  "date": "2026-11-05",
  "run": 10452,
  "outputs": [
    {
      "process": 1,
      "processName": "Sales Summary",
      "type": "PDF",
      "fileName": "Sales Summary.pdf",
      "url": "https://…/Sales Summary.pdf"
    },
    {
      "process": 1,
      "processName": "Sales Summary",
      "type": "Excel",
      "fileName": "Sales Summary.xlsx",
      "url": "https://…/Sales Summary.xlsx"
    }
  ]
}
```

| Field | Description |
|----|----|
| `process` | The sequence number of the process that made the file. |
| `processName` | That process's name, as in the Automation Log. |
| `type` | `PDF` or `Excel`. |
| `fileName` | The file's name. |
| `url` | A download link for the file. Download it promptly, and don't publish it. |

### Your answer

| Your endpoint… | The process… |
|----|----|
| answers **2xx** | passes, and the run goes on to its next process. |
| answers any other status | fails, and the run stops. |
| cannot be reached, or doesn't answer within the timeout | fails, and the run stops. |

- The answer's status and the first 500 characters of its body are shown
  in the Automation Log. Return a short, readable message.
- The timeout is 1–300 seconds (30 by default), and never longer than the
  time left in the workflow's runtime limit.
- The request is **not retried**. Answer quickly with 2xx and do slow work
  afterwards. If your endpoint can receive the same `{{RunID}}` twice (for
  example when a user starts the run again), make it safe to repeat.

### Authenticating Alaya's calls

Alaya doesn't sign its requests. To check that a call comes from your
workflow:

- Add a secret header in the process, for example
  `X-Alaya-Webhook-Key: <long random value>`, and reject requests that
  don't carry it. Headers are saved encrypted.
- Use an `https://` URL.
- Optionally, check `{{RunID}}` against the run you expect.

## Patterns

### Report delivery to another system

**Schedule trigger → System Report (PDF/Excel) → Webhook process with
Send outputs.** Your endpoint receives the file links every day and
downloads the files.

### Plugin-driven automation

Combine with a [Plugin](plugin.md) and the
[Module Editor Record trigger](automation.md#module-editor-record):

1. Your plugin, your service or a user saves a record of your Module
   Editor table (on its screen, or through `api/Udf/ZZTSaveRow`).
2. The workflow whose Module Editor Record trigger watches that table runs.
3. Its Webhook process calls your service with the record, for example:

   ```json
   { "table": "{{TableName}}", "event": "{{Event}}", "id": {{RecordID}}, "run": {{RunID}} }
   ```

If your service saves the same table again when it is called, keep it
from running the workflow over and over: only save when something
really changed, or watch only the events you need. The trigger starts a
workflow at most 30 times a minute.

To start a workflow from your system **without** saving a record, call
its [webhook URL](#starting-a-workflow-with-its-webhook-url) instead.
That URL carries no data, so the workflow has to find its work from the
data itself. A common way is a status or *processed* field: pick up the
rows that are not processed yet, then mark them done.

### Chaining with other workflows

To run several workflows one after another from one URL, use a
[Workflow process](automation.md#workflow-process) instead of calling
several URLs.
