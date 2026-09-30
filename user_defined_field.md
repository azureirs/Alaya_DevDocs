---
title: User Defined Field
section: Customization
order: 11
---

# User Defined Field

**User Defined Fields** (**UDF**) is a feature in
Alaya ERP
that allows admin users to extend built-in system entities — such as
Item Master, Customer, Vendor or transaction based modules like Sales
Invoice or Purchase Invoice — with additional custom fields, without
modifying application code. The system displays these fields within the
relevant module under a dedicated **User Defined Field** tab.

Do not confuse with
[Module Editor](module_editor.md). Although Module Editor uses the same underlying UDF stack,
it is a separate feature, which instead of extending existing entities
it creates entirely new standalone modules with their own screens.

## Overview

Every deployment has data requirements that go beyond the fields
provided by standard system modules. The UDF feature bridges this gap by
letting administrators attach extra fields directly to existing system
entities. Once configured, the custom fields appear automatically under
the **User Defined Field** tab of the relevant module screen, and are
stored and retrieved alongside the standard record data.

Supported system entities include, but are not limited to:

- Item Master
- Customer
- Vendor
- Sales Invoice
- Purchase Invoice
- and many more...

```
Note: The full list of supported entities varies by deployment. Check the 'User Defined Field' configuration screen ('Utility' → 'Customization' → 'User Defined Field') to see which entities are available in the current installation.
```
## Managing UDFs

UDF fields are configured through the system's **User Defined Field**
administration screen. Administrators select a system entity, then
define one or more fields to attach to it. Once saved, the fields appear
immediately under the **User Defined Field** tab in the relevant module
for all records of that entity type.

```
Note: If logged in user already opened the relevant detail screen before the administrator update / save the UDF, the user may not be able to save and have to redo data input after a screen refresh.
```
## UDF Objects

### UdfTable (System Table)

For the UDF feature, each supported system entity has a corresponding
`UdfTable` record where `UT_SystemTable_YN = true`. These are
pre-defined by the system and are identified by a `sys.` prefix in
`UT_TableName` (e.g. `sys.ItemMaster`). Administrators do not create
these — they only attach `UdfColumn` records to them.

Key attributes relevant to UDF:

| Property | Description |
|----|----|
| `UT_TableName` | Unique code for the entity. System tables use a `sys.` prefix. |
| `UT_Caption` | Display name shown in the UDF configuration screen. |
| `UT_SystemTable_YN` | `true` for system/UDF tables; `false` for Module Editor menus. \<!--\|- |

### UdfColumn

`UdfColumn` is the core object of the UDF feature. Each instance
represents one custom field attached to a `UdfTable`. The physical
database column is created with a `ZZC_` prefix on `UC_FieldName`.

| Property | Description |
|----|----|
| `UC_TableName` | The `UT_TableName` of the parent `UdfTable`. |
| `UC_FieldName` | Unique field code across all modules. The physical database column is named `ZZC_<UC_FieldName>`. |
| `UC_FieldType` | Database column type (eg. `nvarchar`, `int`, `bit`, `decimal`, `datetime`). |
| `UC_LayoutType` | UI field type used by the form editor (e.g. `text`, `integer`, `checkbox`). Determines which input control is rendered. |
| `UC_DefaultCaption` | Display label shown under the **User Defined Field** tab. |
| `UC_DefaultSeq` | Display order of the field within the tab when generated UI is using simple editor. |
| `UC_SystemColumn_YN` | `true` for system-generated columns (ID, audit fields, etc.). Not user-configurable. |
| `UC_RfDataTable` | For **Related Field** type: the physical database table to look up from. |
| `UC_RfDataToShow` | For **Related Field** type: the database column whose value is displayed. |
| `UC_Properties` | JSON blob (`UdfColumnProperties`) storing designer-time configuration. See below. |

```
Note: 'UC_FieldName' must be unique across all modules and all tables in the system — not just within the parent 'UdfTable'. The system validates this on save.
```
**Reserved `UC_FieldName` values** (cannot be used for user-defined
columns): `ID`, `DateCreated`, `CreatedBy`, `DateModified`,
`ModifiedBy`, `CompanyId`, `IAGID`, `CustomerGroup`, `VendorGroup`,
`DocNo`, `Code`, `Description`, `Status`. Field names using the prefix
`RFT_` are also reserved (used internally for Related Field companion
columns).

## Relationship to Module Editor

The
[Module Editor](module_editor.md) uses the same underlying stack as UDF but serves a different
purpose:

|  | UDF | Module Editor |
|----|----|----|
| **Purpose** | Extend existing system entities with extra fields. | Create entirely new standalone modules and screens. |
| **Appears in UI as** | A **User Defined Field** tab inside an existing system module. | A new top-level entry under "My Module" in the main navigation. |
| **Core object** | `UdfTable` (`UT_SystemTable_YN = true`) → `UdfColumn` per field. | `UdfApp` → `UdfTable` (`UT_SystemTable_YN = false`) → `UdfColumn` per field. |

Both features share the same field types and the same API controller
(`Udf`).

## UDF Data

UDF column values are stored per system entity record. When a user opens
an existing record (e.g. an Item Master entry), Alaya retrieves the
`UdfColumn` definitions for that entity's `UdfTable` and displays their
current values under the **User Defined Field** tab. Saving the record
persists both the standard fields and the UDF values together.

The physical UDF column values are stored in the same database table as
the entity, in the `ZZC_`-prefixed columns added when the `UdfColumn`
was created.

## API Access

UDF data and structure are accessible programmatically via the **Udf**
controller in the Alaya REST API. This allows external systems to
interact with custom modules without going through the UI.

Refer to [Module Editor: API Access](module_editor.md#api-access) for the
base URL pattern and authentication requirements, and to
[API Access](api_access.md)
for the full API setup guide.

The Swagger documentation portal (accessible at the client's `ApiUrl`)
lists all available Udf endpoints with their request and response
schemas.

## Related Pages

- [Module Editor](module_editor.md)
- [API Access](api_access.md)
