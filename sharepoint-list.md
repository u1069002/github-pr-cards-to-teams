# SharePoint list setup

The list exists for one reason: Power Automate runs are stateless. The Teams message ID
generated when a card is first posted has to be written down so a later run - triggered
by a push, a rename, a merge - can find it and update the right card.

It is an index of open pull requests, not a log. Rows are created on first post and
deleted on close.

## Where to put it

Any site you have Member or Owner rights on. Natural home is the site backing the Teams
team you're posting into: in Teams, channel → **Files** → **⋯** → **Open in SharePoint**.

## Columns

| Display name | Internal name | Type | Notes |
|---|---|---|---|
| PR Key | `Title` | Single line of text | Built-in Title column, display name changed |
| MessageId | `MessageId` | Single line of text | Create it with **no space** in the name |

Example rows:

| PR Key | MessageId |
|---|---|
| `acme/widgets#42` | `1753891234567` |
| `acme/widgets#43` | `1753894441201` |

## Creation steps

1. Site → **+ New** → **List** → **Blank list**. Name it `PR Card Index`. Uncheck
   "Show in site navigation".
2. **+ Add column** → **Single line of text** → name it exactly `MessageId`.
3. Rename the Title column's display name: click the **Title** header → **Column
   settings** → **Edit** → `PR Key`.
4. Index it: gear → **List settings** → **Columns** section → **Indexed columns** →
   **Create a new index** → primary column **PR Key**.
5. Versioning off: **List settings** → **Versioning settings** → "Create a version each
   time you edit an item in this list?" → **No**.

Steps 4 and 5 need the Settings gear, which lives in the SharePoint bar at the top right
of the browser window - not in the list's own command bar. It is absent if you're viewing
the list inside Teams, or if you're only a Visitor on the site. Both steps are optional
at normal scale; skip them if the gear isn't available.

## The internal-name trap

SharePoint sets a column's **internal** name from whatever you type at creation and never
changes it, even if you rename the column later. Create it as "Message Id" and the
internal name is permanently `Message_x0020_Id`, which then has to appear in every
expression. The only fix is deleting the column and starting over.

This is also why the key uses the built-in `Title` column rather than a new "PR Key"
column - `Title` is already clean.

## Verifying internal names

Don't read the settings URL. Check what the flow will actually see:

1. Add a row by hand with any values
2. Run the flow once
3. Expand **Get items** → **Outputs**

```json
"value": [
  { "ID": 1, "Title": "acme/widgets#42", "MessageId": "1753891234567" }
]
```

Those are the true internal names. If you see `Message_x0020_Id`, recreate the column.

**Delete the hand-added row afterwards.** A row whose Title matches a real PR but whose
MessageId is blank will shadow that PR permanently and every update will fail on a null
message ID.

## Permissions

The Power Automate SharePoint connection needs write access - the flow creates and
deletes rows.
