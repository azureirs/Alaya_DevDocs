---
title: Module Editor
section: Customization
order: 12
---

# Module Editor

The **Module Editor** is a built-in low-code customisation tool in
Alaya ERP
that allows authorised users to create, configure, and manage **custom
modules** without writing application code. Each custom module can
contain one or more **menus**, and each menu exposes a user-designed
form built from a library of **field types**.

## Overview

Most ERP deployments require forms and data structures that go beyond
the system's standard modules. The Module Editor bridges that gap by
giving administrators a visual workspace to:

- Define new business entities as standalone modules.
- Organise data entry screens into logical menus within each module.
- Compose form layouts by dragging and dropping field types onto a
  canvas.
- Publish the resulting module so end-users can access it from the main
  navigation.

## Accessing the Module Editor

1.  Log in as a user with the **Admin** role.
2.  Navigate to **Utility** → **Customization** → **Module Editor**.
3.  The Module Editor workspace opens, listing all existing custom
    modules.

## Concepts

### Module

A **module** is the top-level container for a custom business entity. It
appears as a dedicated entry in the main application menu once
published. Internally, it is identified with the object
**[UdfApp](user_defined_field.md#relationship-to-module-editor)**. A module has:

| Property | Internal (`UdfApp`) | Description |
|----|----|----|
| **Module Code** | `UA_Name` | Unique identifier, it is not editable after saved. Maximum 80 characters. |
| **Module Name** | `UA_Caption` | Display name shown in the navigation menu. Maximum 80 characters. |
| **Sequence** | `UA_RibbonSeq` | Position of module shown in navigation menu under "My Module". |
| **Active** | `UA_Active_YN` | *Unchecked* (hidden from users) or *Checked* (visible to permitted users). |

### Menu

Each module can contain one or more **menus**. A menu represents a
distinct view or data entry screen within the module — analogous to a
sub-menu item in the navigation hierarchy. Examples: *Requests*,
*Approvals*, *Archive*. Internally, it is identified with the object
**[UdfTable](user_defined_field.md#udftable-system-table)**.

| Property | Internal (`UdfTable`) | Description |
|----|----|----|
| **Menu Code** | `UT_TableName` | Unique identifier for the menu within the module. Alphanumeric only (no `sys.` prefix). Not editable after saved. |
| **Menu Name** | `UT_Caption` | Display name shown as the sub-menu item in the navigation hierarchy. |
| **Status** | `UT_Active_YN` | *Unchecked* (hidden from users) or *Checked* (visible to permitted users). |
| **Associated Group** | `UT_AssociateGroup` | Data grouping scope for records in this menu: `NONE` (no grouping), `CMP` (by Company), `AIG` (by Inventory Group), `ACG` (by Customer Group), `AVG` (by Vendor Group). |

### Field Types

When designing a menu's form layout, admins place **fields** onto the
canvas. The following field types are available:

| Field Type | Description | Example Use |
|----|----|----|
| **Text** | Single-line plain text input. | Employee name, Reference number |
| **Multiline Text** | Multi-line text area for longer content. | Notes, Description, Address |
| **Integer** | Whole-number numeric input. | Quantity, Count, Age |
| **Decimal** | Fixed-decimal or floating-point numeric input. | Price, Weight, Tax rate, Percentage |
| **Date** | Calendar date picker for selecting a single date. | Start Date, Due Date, Expiry Date, Date of Birth |
| **Checkbox** | Boolean toggle (checked = true, unchecked = false). | Approved, Active, Is Required, Completed |
| **Selection** | Drop-down list of user-defined options. | Status, Category, Priority |
| **Related Field** | Lookup field that references a record from another module or master data entity. | Employee, Customer, Vendor, Item |

```
Note: Additional field types will be documented here as they are confirmed.
```
### Menu Templates

Each menu has a **Menu Type** (`UT_MenuType`) that determines the form
template, the system-generated columns, and the deletion behaviour:

| Type | `UT_MenuType` | System columns | Deletion behaviour |
|----|----|----|----|
| **Blank** | `1` | N/A | N/A |
| **Master Data** | `2` | ID, Code, Description, DateCreated, CreatedBy, DateModified, ModifiedBy | Hard delete (record removed permanently). \<!-- |
| **Document** | `3` | ID, DocNo, Status, DateCreated, CreatedBy, DateModified, ModifiedBy | Soft delete — Status is set to `CANCEL`. Record is retained. Requires `UT_DocNoPrefix` (must start with `Z`) for document numbering. --\> |

```
Note: 'Blank' type is used for 'External Views'. No physical database table or columns will be generated.
```
```
Note: Values '100' (system_default) and '101' (system_simple) are reserved for system modules and are not available in the Module Editor.
```
### UdfColumn

Each field placed on a menu form is stored as a `UdfColumn` record. The
UI field type (Text, Date, etc.) maps to `UC_LayoutType`; the underlying
database storage type is recorded separately in `UC_FieldType` (e.g.
`nvarchar`, `int`, `bit`).

The physical database column for each field is named
`ZZC_<UC_FieldName>`, and the physical database table for the menu is
named `ZZT_<UT_TableName>`. Database triggers are prefixed `ZZZ_`. These
are created automatically by the system when a field or menu is saved
via `SP_SyncUdfToDbSchema`.

`UC_FieldName` must be **unique across all modules**, not just within a
menu.

**Reserved `UC_FieldName` values** (system-generated, not
user-configurable): `ID`, `DateCreated`, `CreatedBy`, `DateModified`,
`ModifiedBy`, `CompanyId`, `IAGID`, `CustomerGroup`, `VendorGroup`,
`DocNo`, `Code`, `Description`, `Status`.

**Related Field companion column:** When a `relatedfield`-type column is
saved, the system automatically creates a paired companion column
prefixed `RFT_` (e.g. `RFT_EmployeeId`) of type `nvarchar(MAX)` to store
the display text of the referenced record. This companion column is
system-managed (`UC_SystemColumn_YN = true`) and is deleted
automatically when the related field is deleted.

## Creating a Custom Module

### Step 1 — Add a New Module

1.  In the Module Editor listing, click **+ New**.
2.  Enter a **Module Code** (e.g. *FranchiseManagement*).
3.  Enter a **Module Name** (e.g. *Franchise Management*).
4.  Enter **Sequence**.
5.  Uncheck **Active** if you do not want to publish it immediately.
6.  Click **Confirm**. The module is created in *Draft* status.

### Step 2 — Add a Menu

1.  Click **+** button for the module.
2.  Enter the **Menu Code** (e.g. *Franchise*). Alphanumeric only; used
    as `UT_TableName` and as the suffix of the physical database table
    (`ZZT_Franchise`).
3.  Enter the **Menu Name** (e.g. *Franchise*).
4.  Check or uncheck **Status**.
5.  Choose the **Associated Group** (data grouping scope — see the
    [Menu](#menu) concept table
    for valid values).
6.  Click **Confirm**.

### Step 3 — Design the Form Layout

1.  Click on the menu **Details** to open the **Detail View Editor**.
2.  From the **New Fields** panel on the left, drag a field onto the
    form canvas.
3.  Configure the field's properties (label, required, default value,
    etc.).
4.  Repeat for all required fields.
5.  Click **Save**.

### Step 4 — Publish the Module

1.  Return to the module's settings page.
2.  Set **Status** to **Checked**.
3.  Click **Confirm**. The module now appears in the main navigation for
    users with appropriate access (configurable from **Staff** → **User
    Group**).

## API Access

Custom module data is accessible programmatically via the **Udf**
controller in the
[Alaya REST API](api_access.md). This allows external systems and integrations to read and write
records in any published custom module without going through the UI.

### Controller

All Module Editor API endpoints are grouped under the `Udf` controller.
The base path follows the pattern:

    {ApiUrl}api/Udf/{endpoint}

Where `{ApiUrl}` is the client-specific base URL resolved via the
[GetUrl](api_access.md#getting-the-api-url) endpoint.

### Authentication

Requests to the Udf controller require the same API credentials (ApiCode
and ApiKey) used across the Alaya API. Refer to
[API Access](api_access.md)
for the full authentication setup.

### Browsing Available Endpoints

The complete list of Udf endpoints — including request/response schemas
— is available in the Swagger documentation portal at the client's
`ApiUrl`. See
[Accessing the API Documentation](api_access.md#documentation) for login instructions.

## Permissions

Access to the Module Editor and to published custom modules is governed
at two levels:

### Module Editor Access

Only users assigned the **Admin** role can open the Module Editor,
create or modify modules, and publish them. Regular users have no access
to the editor interface.

### Published Module Access

Once a module is published (**Active** checked), visibility and access
for end-users is controlled through **Staff** → **User Group**. Only
members of that group can see and interact with the corresponding menu.
This follows the same role-based access control model used across all
standard Alaya ERP modules.

## Related Pages

- [Plugin (Blank Menu)](plugin.md)
- [User Defined Field](user_defined_field.md)
- [API Access](api_access.md)
