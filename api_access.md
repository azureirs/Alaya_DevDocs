---
title: Alaya API
section: Customization
order: 13
---

# API Access

The **Alaya ERP** platform exposes a REST API surface, allowing approved
developers to integrate with client environments programmatically.
Access requires a valid API licence, a dynamically resolved API base
URL, and credentials.

## Prerequisites

Before making any API call, the following must be in place:

### API Licence

API access is tied to a per-client licence add-on. Ensure the following
conditions are met:

- The client has purchased the **API addon** for their Client ID.
- The client has been issued an **ApiCode** and an **ApiKey**:

| Credential | Description |
|----|----|
| **ApiCode** | The licence identifier. Typically the same value as the client's Alaya Client ID. |
| **ApiKey** | A secret key in GUID format (e.g. `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`). Treat this as a password — do not share or commit it to source control. |

```
Note: If the client has not received their ApiCode or ApiKey, contact the Alaya support team to provision the API addon for that Client ID.
```
## Getting the API URL

Each client may be assigned to a different database server, so the API
base URL varies per client. Use the following public endpoint to resolve
the correct API URL before making any other calls.

### Endpoint

| Field            | Value                                            |
|------------------|--------------------------------------------------|
| **URL**          | `https://idealer.irs-alaya.com/api/Alaya/GetUrl` |
| **Method**       | `POST`                                           |
| **Content-Type** | `application/json`                               |

### Request Body

    {
      "ClientCode": "your_client_id"
    }

### Response

    {
      "Result": true,
      "Message": "message if there is issue",
      "Obj": {
        "SiteUrl": "http://www.irs-alaya.com/",
        "ApiUrl": "http://api.irsalaya.com/v3/"
      }
    }

| Field | Description |
|----|----|
| **Result** | `true` if the lookup succeeded; `false` if the Client ID was not found or another error occurred. |
| **Message** | Human-readable error detail when `Result` is `false`. Empty on success. |
| **Obj.SiteUrl** | The client's Alaya web application URL. |
| **Obj.ApiUrl** | The base URL for all subsequent API calls. Use this as the root for every endpoint path. Cache this value on the client side to avoid repeated calls to the `GetUrl` endpoint — only re-fetch if a request fails with a connectivity or host error. |

```
Note: Always resolve the API URL dynamically via this endpoint rather than hardcoding it. The assigned server may change when a client is migrated.
```
## Documentation
API Endpoints Documentation can be accessed [here](https://documenter.getpostman.com/view/6739034/2sA3sAhnyd#4b4691ac-a0b6-409f-a274-45c95b600ee0/).

## Related Pages

- [Module Editor](module_editor.md)
