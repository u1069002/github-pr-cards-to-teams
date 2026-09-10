# GitHub PR cards to Teams

Posts an Adaptive Card to a Teams channel when a pull request opens, then keeps that
same card current as the PR is pushed to, renamed, drafted, closed and merged.

One card per pull request. No duplicates, no notification spam.

```
┌────────────────────────────────────────────┐
│ NEW PULL REQUEST · acme/widgets #42        │
│ Fix null check in parser                   │
│                                            │
│ (o) Obie Munoz (u1069002)                  │
│     feature/parser → main                  │
│                                            │
│ +124  -38    6 files · 3 commits           │
│                                            │
│ Opened Thu, Jul 30th, 2026 at 2:22 PM      │
│                                            │
│ [ View pull request ]  [ Files changed ]   │
└────────────────────────────────────────────┘
```

The header line and its colour change with state: `New pull request` and `Updated` in
blue, `Ready for review` blue, `Back to draft` amber, `Merged` green, `Closed` red.

Once the PR is merged or closed the card **collapses to a single line**, so finished work
stops competing for space with work that still needs eyes:

```
┌────────────────────────────────────────────┐
│ Merged · #42 — Fix null check in parser    │
│ [ View pull request ]  [ Files changed ]   │
└────────────────────────────────────────────┘
```

Green for merged, red for closed-without-merging. Nothing is deleted - GitHub is the
permanent record, and the buttons still go there.

The summary drops the `owner/repo` scope that the header carries, because Teams gives a
channel card a fixed ~448px and every character spent is a character the title loses.

## Design constraints

Built entirely on **Standard connectors**. No Power Automate Premium licence is
required, which rules out the HTTP action and the GitHub connector and shapes most of
what follows.

| Need | Obvious approach | Why not | What we do instead |
|---|---|---|---|
| Receive a webhook | `When an HTTP request is received` | Premium trigger | `When a Teams webhook request is received` (Teams connector, Standard) |
| Remember which card belongs to which PR | Dataverse | Premium connector | SharePoint list, two columns |
| Read merge-conflict state | `GET /repos/.../pulls/{n}` | Needs HTTP action | Not implemented. See Limitations. |

## Architecture

```
GitHub  ──webhook──▶  Teams webhook trigger
  pull_request              │
  events                    ▼
                       Parse JSON
                            │
                            ▼
                       Get user   ◀──────┐  SharePoint list
                      seed if missing    │  GitHub login ⇄ real name
                            │            │
                            ▼            │
                       Get items  ◀──────┼─┐  SharePoint list
                    Title eq 'repo#42'   │ │  Title ⇄ MessageId
                            │            │ │
                            ▼            │
                  first-time-seeing-pr   │
                    │              │     │
         no row ────┘              └──── row exists
              │                            │
     should-post-card?                Update adaptive card
       │          │                        │
      yes         no                  is-closed?
       │          │                     │      │
   Post card   (nothing)              yes     no
       │                               │
   Create item ────────────────────▶ Delete item
```

Everything hangs off the SharePoint row. It exists for exactly one reason: Power
Automate runs are stateless, so the Teams message ID produced when the card is first
posted has to be written down somewhere for later runs to find.

## Components

### 1. SharePoint list

See [`sharepoint-list.md`](./sharepoint-list.md).

Two columns. `Title` (built in) holds `owner/repo#number`, `MessageId` holds the Teams
message ID. Rows are created and deleted, never updated.

### 2. GitHub user map

See [`user-map-list.md`](./user-map-list.md).

Maps an opaque GitHub login to a real name so the card reads `Obie Munoz (u1069002)`
rather than `u1069002`. Self-seeding: unknown logins get a blank row automatically, and a
view filtered to `DisplayName is empty` becomes the fill-in-the-blanks backlog.

Unmapped or blank falls back to the bare login, so the card is never wrong, only less
informative.

### 3. Main flow (in Power Automate)

| Step | Action | Config |
|---|---|---|
| Trigger | When a Teams webhook request is received | Who can trigger: **Anyone**. Trigger condition from `expressions/trigger-condition.txt`. **Concurrency Control on, parallelism 1.** |
| 1 | Parse JSON | Content: trigger `Body`. Schema: `expressions/parse-json-schema.json`. Must be named `Parse JSON`. |
| 2 | Get items → name it **`Get user`** | User map list. Filter Query: `expressions/get-user-filter-query.txt`. Top Count `1`. |
| 2a | Condition `seed-user` | `expressions/seed-user-condition.txt` · is equal to · `0` |
| 2a-i | ↳ True → Create item → name it **`Create user row`** | User map list. Title = the GitHub login. DisplayName and Email blank. |
| 2a-ii | ↳ False → *(empty)* | |
| 3 | Get items | Card index list. Filter Query: `expressions/get-items-filter-query.txt`. Top Count `1`. **Leave this one named `Get items`** - many expressions reference it. |
| 4 | Condition `first-time-seeing-pr` | `length(body('Get_items')?['value'])` · is equal to · `0` |
| 4a | ↳ True → Condition `should-post-card` | `expressions/should-post-condition.txt` · is equal to · `yes` |
| 4a-i | ↳ True → Post card in a chat or channel | Flow bot / Channel. Card: `expressions/adaptive-card.json` |
| 4a-ii | ↳ Create item | Title = same `concat(...)` as the filter query. MessageId = the post action's message ID. |
| 4a-iii | ↳ False → *(empty)* | Intentional. Swallows pushes and closes for PRs that predate the flow. |
| 4b | ↳ False → Update an adaptive card | Message Id: `first(body('Get_items')?['value'])?['MessageId']`. Same Team, Channel and card as the post. On `closed` the same template renders itself collapsed - no separate action needed. |
| 4b-i | ↳ Condition `is-closed` | `body('Parse_JSON')?['action']` · is equal to · `closed` |
| 4b-ii | ↳ True → Delete item | Id: `first(body('Get_items')?['value'])?['ID']` |

### 4. Janitor flow

Separate scheduled flow. Deletes rows orphaned by close events that never arrived.

| Step | Action | Config |
|---|---|---|
| Trigger | Recurrence | Weekly, Sunday 03:00 |
| 1 | Get items | Filter Query: `expressions/janitor-filter-query.txt`. Top Count `5000`. |
| 2 | Apply to each | `body('Get_items')?['value']` |
| 3 | ↳ Delete item | Id: `items('Apply_to_each')?['ID']` |

90 days is deliberately generous. Deleting a row early is benign: the next event finds
no row and posts a fresh card instead of updating the old one.

### 5. GitHub webhook

Repo → Settings → Webhooks → Add webhook.

| Field | Value |
|---|---|
| Payload URL | the trigger's POST URL |
| Content type | **`application/json`** |
| Secret | blank (see Security) |
| Events | Let me select individual events → **Pull requests** only |

The same URL can be reused across every repo. `repository.full_name` is on the card while
the PR is open, so the source is visible where it matters. Note the collapsed summary
drops it - if you do point several repos at one flow, finished cards show a bare `#42`
and two repos can produce the same number. Put `full_name` back in the `concat(...)` if
that lands.

## Behaviour matrix

Verified against every transition:

| User action | Flow runs? | Result | Card header |
|---|---|---|---|
| Open a PR (not draft) | yes | post card, create row | New pull request (blue) |
| Push a commit | yes | update card | Updated (blue) |
| Rename the PR | yes | update card | Updated (blue) |
| Mark as draft | yes | update card | Back to draft (amber) |
| Push while draft | **no** | filtered at trigger | - |
| Mark ready for review | yes | update card | Ready for review (blue) |
| Close without merging | yes | **collapse card**, delete row | Closed (red, one line) |
| Reopen | yes | **post a new card**, create row | Reopened (blue) |
| Merge | yes | **collapse card**, delete row | Merged (green, one line) |
| Open as a draft | yes | nothing | - |
| Close a PR that was only ever a draft | yes | nothing | - |
| Mark a draft ready (no card yet) | yes | post card, create row | Ready for review (blue) |

Reopening posts a *new* card rather than reviving the old one, because the row was
deleted on close and there is no message ID left to update. The old card stays in the
channel, collapsed to its one-line `Closed` summary, which is why collapsing matters
here beyond tidiness: the stranded card is a footnote above the live one rather than a
full-size duplicate competing with it. To get one card per PR forever instead, stop
deleting on close and shorten the janitor window to about 14 days - at the cost of the
list holding every PR from the last fortnight rather than only open ones.

## Design notes

Decisions that are not obvious from reading the flow, each of which cost real
debugging time.

### Let the platform escape JSON, never hand-roll it

The Adaptive Card field is a **string template**. Power Automate splices dynamic values
into it with no escaping whatsoever, so a PR titled `Fix "off by one"` produces invalid
JSON and the action fails with `InvalidBotRequestMessageBody`.

Chained `replace()` calls are a trap. Each one fixes a single character class and the
next hostile input finds the gap - quotes, then backslashes, then tabs, then control
characters. Instead, hand the value to the platform's own JSON serialiser:

```
slice(string(array(<value>)), 1, -1)
```

`array()` wraps it, `string()` serialises the array to JSON with correct escaping, and
`slice()` strips the brackets - leaving the value **complete with its surrounding
quotes**. So in the card the field carries no manual quotes:

```json
"text": @{slice(string(array(body('Parse_JSON')?['pull_request']?['title'])), 1, -1)},
```

This handles quotes, backslashes, tabs, newlines, control characters and surrogate
pairs correctly by construction. Consequence: the card file no longer parses as JSON on
its own, only after the expressions resolve.

### Never compare booleans in a Condition

Typing `true` into a Condition's value box stores the **string** `"true"`. An expression
like `empty(...)` returns the **boolean** `true`. `equals(true, 'true')` is false, so the
condition silently inverts and takes the wrong branch without erroring.

Every condition here compares a number to a number or a string to a string:

- `length(body('Get_items')?['value'])` → `0`
- `if(..., 'yes', 'no')` → `yes`
- `body('Parse_JSON')?['action']` → `closed`

### Collapse finished cards, don't delete them

A merged PR's card is dead weight in a review channel. Deleting it is tempting - GitHub is
the permanent record, so nothing is actually lost - but a delete takes any **threaded
replies** with it, silently, and discussion on the card is exactly the thing that has no
copy in GitHub. Collapsing keeps that thread anchored to something.

So the card collapses instead. The detail blocks live in a `Container` whose `isVisible`
goes false on `closed`, and a one-line summary with the opposite `isVisible` takes their
place. **The flow is untouched** - step 4b already fires `Update an adaptive card` on every
event, so re-rendering the same template with a different `action` does the whole job.
Putting the collapse in the `Container` rather than on each block also means blocks added
later inherit it automatically.

### Emit `isVisible` as a quoted string, never a real boolean

The obvious spelling of the collapse toggle is wrong:

```
"isVisible": @{if(equals(body('Parse_JSON')?['action'], 'closed'), false, true)}
```

Logic Apps stringifies booleans with a capital first letter, so that resolves to
`"isVisible": True` - which is not valid JSON, and the action fails with
`InvalidBotRequestMessageBody`. Return **quoted strings** instead:

```
"isVisible": @{if(equals(body('Parse_JSON')?['action'], 'closed'), 'true', 'false')}
```

The template carries no quotes around the `@{...}`, so `'true'` lands as the bare token
`true` and parses as a JSON boolean. Same trick as the title escaping: let the value reach
the JSON as text you control, not as a type the platform will reformat on you.

This is the card-template twin of "never compare booleans in a Condition" - both are the
same underlying hazard, that a boolean crossing a string boundary does not survive as one.

### Guard drafts at the post decision, not at the trigger

Putting `draft != true` on the trigger makes a draft PR invisible to the flow entirely -
including *transitions into* draft and closes that happen while drafted. A card that
already exists then freezes and can never be updated again.

The draft check belongs on `should-post-card`, which decides whether to create a *new*
card. Once a card exists, every event should reach it.

### The author name needs the same escaping as the title

Real names are user-supplied text too. `Bob "Bobby" Tables` would break the card exactly
like a quoted PR title, so the author line goes through the same
`slice(string(array(...)))` serialiser. Verified against apostrophes, quotes, accents and
bracketed bot logins like `dependabot[bot]`.

### Use the built-in `Title` column as the key

A new SharePoint column named "PR Key" gets the internal name `PR_x0020_Key`, which then
has to appear in every filter query and expression. Internal names are set at creation
and never change. `Title` is already there, already indexable, and clean.

## Limitations

- **Merge conflicts are not shown.** `mergeable` and `mergeable_state` are computed
  asynchronously by GitHub and arrive `null` in webhook payloads. Getting a real value
  requires polling the REST API, which needs a premium connector.
- **Fork PRs.** Works for the PR card itself. Would break a future CI-status feature,
  since `check_suite.pull_requests` and `workflow_run.pull_requests` come back empty for
  forks.
- **Reopen creates a second card**, as described above. The stranded one is collapsed, so
  the cost is a line rather than a full card.
- **A collapsed card keeps its buttons.** `isVisible` is a property of card *elements*; no
  schema version has an equivalent for entries in `actions`. Conditionally emitting the
  whole `actions` array would mean hand-rolling JSON around a URL, which the escaping rule
  above exists to prevent. Both buttons stay, and on a merged PR both still point somewhere
  useful. Adaptive Cards 1.5 adds `Action.mode`, which could push `Files changed` into an
  overflow menu on collapsed cards - untested here, and see the version note below.
- **Channel cards are a fixed ~448px wide.** Teams pins the width of a card in a channel
  message and ignores `msteams.width`, which only applies in tabs, task modules and
  stageview. There is no card-level width in the schema at any version, so this is not a
  thing a version bump fixes. Practically the summary holds about 70 characters before it
  wraps to a second line - still far short of the eight the full card occupies.
- **The card declares `version: 1.4`, which nobody chose.** It has been there since the
  first commit. Teams itself renders up to 1.6, but 1.6 is reported broken for Teams cards
  sent through Power Automate, so 1.5 is the only bump worth trying and it needs testing
  against the connector rather than against the Teams docs.
- **The webhook URL is the credential.** GitHub signs requests with HMAC-SHA256 in
  `X-Hub-Signature-256`, and Power Automate has no HMAC function, so the signature can't
  be verified. The SAS signature in the flow URL is the de facto shared secret. Treat it
  accordingly. It can't be rotated in place - you'd delete and re-add the trigger, then
  update GitHub.
- **Card content is invisible to Teams search, quote replies and notification toasts.**
  Adaptive Cards are attachments, not message text. The `summary` property that would fix
  this sits at the attachment level, which the Teams connector action doesn't expose.

## Scale

The list holds **open** PRs, not all PRs, because rows are deleted on close. Steady-state
size is throughput × time-open:

| PRs/week | Days open | Rows | vs 5,000 list view threshold |
|---|---|---|---|
| 50 | 2 | 14 | 0.3% |
| 200 | 2 | 57 | 1.1% |
| 500 | 3 | 214 | 4.3% |
| 2,000 | 3 | 857 | 17.1% |

Storage is irrelevant - SharePoint caps at 30 million items. The constraint that matters
is the 5,000 List View Threshold, which needs roughly 4,000 simultaneously-open PRs to
approach. Indexing `Title` keeps queries working even past it.

The real risk is orphan accumulation, not volume: at 500 PRs/week with the janitor never
running, the list crosses 5,000 rows within about ten weeks.

## Testing

See [`TESTING.md`](./TESTING.md). Four test PRs, roughly 20 minutes, covers every
transition plus a symptom-to-cause table.

## Possible future work

**CI status on the card.** GitHub Actions results arrive on the `workflow_run` event, not
`pull_request`, and its payload carries `pull_requests[].number` - which is exactly the
key the SharePoint row is indexed by.

The complication: an Adaptive Card update replaces the whole card, and a `workflow_run`
payload has no PR title, author, branches or diff stats. Rendering the full card from a
CI event means caching all of that in SharePoint, taking the list from 2 columns to
roughly 10 and making every PR event update rows rather than only create them.

The lighter alternative is a threaded reply on failure, using `Reply with an adaptive
card in a channel` with the stored message ID as the parent. About five actions, no
schema change.
