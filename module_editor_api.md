---
title: Module Editor API
section: Customization
order: 15
---

# Module Editor API

The `Udf` controller of the [Alaya REST API](api_access.md) can create a whole
[Module Editor](module_editor.md) module in one call, and read and write the records of its
Master Data menus. This page lists the rules a request must follow, so that a program or an
AI assistant can build a correct request the first time.

All endpoints are under `{ApiUrl}api/Udf/` and need the API token of a user
(see [API Access](api_access.md)). Every reply is:

```json
{ "Result": true, "Message": null, "Obj": { } }
```

`Result` is `false` when the request is refused, and `Message` says why.

## Concepts

| Term | Stored as | Notes |
|---|---|---|
| Module | `UdfApp` | A ribbon tab. Its code is `UA_Name`. |
| Menu | `UdfTable` | A ribbon button inside the module. Its code is `UT_TableName`. |
| Field | `UdfColumn` | A column of a Master Data menu. Its name is `UC_FieldName`. |
| Record | a row of `ZZT_<menu code>` | Each field is a column named `ZZC_<field name>`. |

Menu types:

| MenuType | Name | What it is |
|---|---|---|
| `2` | Master Data | A list and a detail form of records. Every record has `Code` and `Description`. |
| `1` | Plugin | An external `https://` page shown in an iframe. See [Plugin](plugin.md). It has no records table. |

Document menus (`3`) can't be created through this API.

## Create a module

`POST api/Udf/CreateModule`

Creates the module and its menus the same way the Module Editor screen does. The whole request is
checked before anything is saved, so a refused request saves nothing. The module and its
menus are active, and saving each menu adds it to the ribbon, as the Module Editor does.

```json
{
  "Name": "Fleet",
  "Caption": "Fleet",
  "Menus": [
    {
      "TableName": "Vehicle",
      "Caption": "Vehicles",
      "MenuType": 2,
      "AssociateGroup": "CMP",
      "Fields": [
        { "FieldName": "PlateNo", "Label": "Plate No", "LayoutType": "textbox", "Size": 20, "IsRequired": true, "IsUnique": true },
        { "FieldName": "VehicleType", "Label": "Type", "LayoutType": "selection", "Options": ["Van", "Lorry", "Car"], "DefaultValue": "Van" },
        { "FieldName": "Capacity", "Label": "Capacity (kg)", "LayoutType": "decimal", "Decimal": 2 },
        { "FieldName": "RoadTaxExpiry", "Label": "Road Tax Expiry", "LayoutType": "datetime", "DateTimeType": "Date" },
        { "FieldName": "VehicleActive", "Label": "Active", "LayoutType": "checkbox" },
        { "FieldName": "VehicleNotes", "Label": "Notes", "LayoutType": "multiline", "Size": 2000 }
      ]
    },
    {
      "TableName": "FleetMap",
      "Caption": "Fleet Map",
      "MenuType": 1,
      "ExternalUrl": "https://maps.example.com/fleet",
      "UseSystemToolbar": false
    }
  ]
}
```

Reply `Obj`:

```json
{
  "AppId": 12,
  "Name": "Fleet",
  "Menus": [
    { "TableId": 40, "TableName": "Vehicle", "QueryTableName": "ZZT_Vehicle", "SyncedToDb": true, "SyncedToDbErrorMsg": null },
    { "TableId": 41, "TableName": "FleetMap", "QueryTableName": null, "SyncedToDb": true, "SyncedToDbErrorMsg": null }
  ]
}
```

`SyncedToDb` is `false` when the menu was saved but its table couldn't be created. Open the menu
in the Module Editor and save it again.

### Module rules

| Property | Rule |
|---|---|
| `Name` | Required. 3 to 80 characters, a letter first, letters and digits only. Unique. Can't be changed later. |
| `Caption` | Up to 80 characters, unique among modules. The code is used when it's empty. |
| `Menus` | The menus in ribbon order. May be empty. A module without menus doesn't show on the ribbon. |

### Menu rules

| Property | Rule |
|---|---|
| `TableName` | Required. Same rule as the module code. Unique across all modules. |
| `Caption` | Up to 80 characters, unique in the module. The code is used when it's empty. |
| `MenuType` | `2` Master Data or `1` Plugin. |
| `AssociateGroup` | Master Data only. `NONE` (default): one set of records for all companies. `CMP`: records per company. `AIG`, `ACG`, `AVG`: records per item, customer or vendor associated group. |
| `Fields` | Master Data only. The fields after `Code` and `Description`, in form order. |
| `ExternalUrl` | Plugin only, required. Must begin with `https://`. |
| `UseSystemToolbar` | Plugin only. Shows Alaya's toolbar above the page. |

### Field rules

| Property | Rule |
|---|---|
| `FieldName` | Required. 1 to 50 letters, digits or underscores. Unique across **all** modules, not just this menu. |
| `Label` | Required. The form label. |
| `LayoutType` | One of the layout types below. |
| `Size` | `textbox` and `multiline`: maximum length. Default 100. Above 8000 means unlimited. |
| `Decimal` | `decimal`: decimal places, 1 to 6. Default 2. |
| `IsRequired` | The form won't save without a value. |
| `IsUnique` | No two records may have the same value. A unique field can't have a default value. |
| `Options` | `selection`: its values. Required for a selection. |
| `SelectionType` | `selection`: `Selection` (drop-down, default) or `Radio`. |
| `DateTimeType` | `datetime`: `Date` (default), `Time` or `Datetime`. |
| `Placeholder`, `Tooltip` | Optional texts. |
| `DefaultValue` | The value of a new record. `textbox`, `multiline`, `integer`, `decimal` (a number), `selection` (one of `Options`). |

These names are reserved and can't be used as field names: `ID`, `Code`, `Description`, `DocNo`,
`Status`, `CompanyId`, `DateCreated`, `CreatedBy`, `DateModified`, `ModifiedBy`, `IAGID`,
`CustomerGroup`, `VendorGroup`, and anything starting with `RFT_`.

### Layout types

| LayoutType | Column type | Control |
|---|---|---|
| `textbox` | `nvarchar(Size)` | One-line text |
| `multiline` | `nvarchar(Size)` | Text area |
| `integer` | `int` | Whole number |
| `decimal` | `decimal(18, Decimal)` | Number |
| `datetime` | `datetime` | Date, time or both |
| `checkbox` | `bit` | Tick box |
| `selection` | `nvarchar(200)` | Drop-down or radio buttons |

A related field (a lookup to another table) and the layout-only Tabs and Button controls can
only be added in the Module Editor's detail editor (**Table Details**).

### The detail form

Each Master Data menu gets a detail form straight away. Its fields are laid out on the editor's
12-column grid, three to a row, in this order: `Code`, `Description`, then `Fields` in request
order. A `multiline` field takes a whole row and is two rows high. The form can be rearranged
later in **Table Details**. The layout is saved in the format that editor writes (`UT_Layout`), and
every field's settings in `UC_Properties`.

### What the system adds

A Master Data menu's table `ZZT_<TableName>` also has these columns, which the server fills in:

| Field | Filled with |
|---|---|
| `ID` | The record ID |
| `Code` | Required, unique. A new record without one gets the next number (`000001`, ...). |
| `Description` | Required. A new record without one gets its `Code`. |
| `CompanyId` | `CMP` menus: the record's company |
| `IAGID`, `CustomerGroup`, `VendorGroup` | `AIG`, `ACG`, `AVG` menus: the record's associated group |
| `DateCreated`, `CreatedBy`, `DateModified`, `ModifiedBy` | Set on every save |

New menus get the Module Editor's default action rights (list: view, add, delete, copy and the
grid tools; Master Data form: view, copy, save / edit). Which user groups can open them is set on
the **User Group** screen, as for any menu.

## Read modules

| Endpoint | Returns |
|---|---|
| `GET api/Udf/GetUdfAppList` | All modules (`UdfApp`). |
| `GET api/Udf/GetUdfAppUIById?id={UA_ID}` | One module with its menus in `UdfTables`. |
| `GET api/Udf/GetUdfTableByCode?code={UT_TableName}` | One menu (`UdfTable`). `UT_QueryTableName` is its records table. |
| `GET api/Udf/GetUdfColumnList?tableName={UT_TableName}` | The menu's fields (`UdfColumn`). `UC_Properties` holds each field's settings as JSON: `isRequired`, `isUnique`, `optionValues` (one option per line), and so on. |

Fields with `UC_LayoutType` `rftext` are system copies of a related field's display text. Don't
write them.

## Read and write records

`tableName` is always the menu code (`UT_TableName`), not the `ZZT_` table name. Rows come back
with the `ZZC_` column names (`ZZC_ID`, `ZZC_Code`, ...). Field values are sent without the prefix.

| Endpoint | Body | Returns |
|---|---|---|
| `POST api/Udf/ZZTGetDataTable` | `{ "tableName", "companyId", "searchKeyword", "lstColumn", "exportAction": "", "loadFirstTime": 1 }` | Up to 500 records, or every record matching `searchKeyword`. Records are filtered by the company or its associated group, as the menu keeps them. `lstColumn` lists the columns to search, separated by `\|`, e.g. `ZZC_Code\|ZZC_Description`. |
| `POST api/Udf/ZZTGetRowById` | `{ "tableName", "primaryKey" }` | One record. |
| `POST api/Udf/IsUniqueValue` | `{ "tableName", "primaryKey", "companyId", "fieldName", "value" }` | `true` when no other record has the value. |
| `POST api/Udf/ZZTSaveRow` | `{ "tableName", "primaryKey", "columnValuePairs": { "PlateNo": "WXY 1234", ... } }` | The record ID. `primaryKey` `0` adds a record. |
| `POST api/Udf/ZZTDeleteRow` | `{ "tableName", "primaryKey" }` | `true` when deleted. |

`ZZTSaveRow` saves only the fields it's given. It doesn't check required or unique fields:
check unique values with `IsUniqueValue` first, as the detail form does. When adding a record:

- send `"Code": null` to get the next number, and `"Description": null` to copy the code
- for a `CMP` menu, send `CompanyId`
- for an `AIG`, `ACG` or `AVG` menu, send `IAGID`, `CustomerGroup` or `VendorGroup`

Saving or deleting a record starts the [automations](automation.md) whose Module Editor Record
trigger watches the menu.

Values: dates as `yyyy-MM-dd` (or `yyyy-MM-ddTHH:mm:ss`), checkboxes as `true` / `false`, a
selection as one of its options.

## Related Pages

- [Module Editor](module_editor.md)
- [Plugin (Blank Menu)](plugin.md)
- [Alaya API](api_access.md)
- [Automation API](automation_api.md)
