Developer portal documentation (rendered at /DevPortal/Home/Docs)

- One page per file: <slug>.md. The slug (lowercase letters, digits, - and _) is the URL: /DevPortal/Home/Docs/<slug>
- "index.md" is the landing page; if it does not exist the first page in the menu is shown.
- Optional front matter at the top of the file controls the menu:

    ---
    title: User Defined Field
    section: Alaya API
    order: 10
    ---

  Pages are grouped by section (alphabetical), then sorted by order, then title.
- Link to another page:  [UDF](user-defined-field.md)
- Images go in img/ (png, jpg, jpeg, gif only) and are referenced as ![caption](img/name.png).
  They are served only to logged-in, Active developers.
- Tables, fenced code blocks and other GitHub-style Markdown features are supported.
- This folder is under App_Data, so files are never served directly by IIS.
