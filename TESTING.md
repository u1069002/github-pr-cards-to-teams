# Test plan - GitHub PR cards in Teams

Four test PRs cover every path. Roughly 20 minutes.

## Before you start

Open three tabs side by side:

1. The Teams channel
2. The SharePoint list
3. Power Automate → the flow → **28-day run history**

Confirm the SharePoint list is **empty**. Any leftover row will shadow a real PR and produce confusing results. Delete stragglers first.

Note the meaning of each column below:

- **Teams** - what should visibly happen in the channel
- **List** - what the SharePoint row should do
- **Run** - what run history should show

---

## PR A - full lifecycle (9 steps)

Open it with this exact title. It exercises the escaping path at the same time:

```
Fix "off by one" & <edge> case
```

| # | Do this | Teams | List | Run |
|---|---|---|---|---|
| A1 | Open the PR, **not** a draft | New card. Header **New pull request**, blue. Title renders exactly, quotes and `&` and `<>` intact | Row appears. Title = `owner/repo#N`, **MessageId populated** | Success. Post card + Create item green |
| A2 | Push a commit | **Same** card updates. Header **Updated**. Commit and diff counts change. **No second card** | Unchanged | Success. Update green |
| A3 | Rename to `Renamed: "still" quoted` | Same card, title changes | Unchanged | Success |
| A4 | Mark as draft | Same card. Header **Back to draft**, amber | Unchanged | Success |
| A5 | Push a commit while draft | **Nothing changes** | Unchanged | **No run at all** |
| A6 | Mark ready for review | Same card. Header **Ready for review**, blue | Unchanged | Success |
| A7 | Close without merging | Same card **collapses to one line**: `Closed · #N — <title>`, red, no `owner/repo`. Avatar, branches, diff stats and timestamp all gone | **Row disappears** | Success. Delete item green |
| A8 | Reopen | **New second card**, full size. Header **Reopened**. First card still sits above it collapsed at `Closed` | New row, new MessageId | Success. Post + Create |
| A9 | Merge | The **second** card collapses to one line: `Merged · #N — <title>`, green | Row disappears | Success. Delete |

A1 is the escaping test. If the title shows mangled characters or the run fails with `InvalidBotRequestMessageBody`, the `slice(string(array(...)))` wrapper isn't in place on the title field.

A5 is the only step where **no run** is correct. A run that fires and does nothing means the trigger condition's synchronize-plus-draft exclusion isn't right.

A8 producing a second card is intended, not a bug. The row was deleted at A7, so there's no message left to update.

A7 and A9 are the collapse test. The one-line summary must still show the title with its
quotes and `&` and `<>` intact - it goes through the same `slice(string(array(...)))`
serialiser as the full card, so a mangled title here means the `concat(...)` was built
outside the wrapper. Both buttons staying is expected; Adaptive Cards 1.4 can't hide them.

---

## PR B - draft-first path (4 steps)

| # | Do this | Teams | List | Run |
|---|---|---|---|---|
| B1 | Open the PR **as a draft** | Nothing | No row | Run fires, everything skipped |
| B2 | Push a commit | Nothing | No row | **No run** |
| B3 | Mark ready for review | Card appears. Header **Ready for review** | Row appears with MessageId | Success. Post + Create |
| B4 | Merge | Card collapses to one line, **Merged**, green | Row disappears | Success. Delete |

B1 firing a run that does nothing is correct - the trigger lets it through, `should-post-card` returns `no`, everything downstream skips. The run shows as successful.

---

## PR C - concurrency (1 step)

| # | Do this | Teams | List | Run |
|---|---|---|---|---|
| C1 | Open a PR, then push a commit **within ~5 seconds** | Exactly **one** card, ending on **Updated** | Exactly **one** row | Two runs, both successful, running one after the other |

Two cards for the same PR means Concurrency Control isn't set to 1. Both runs read "no row" before either wrote one.

---

## PR D - close a draft (2 steps)

| # | Do this | Teams | List | Run |
|---|---|---|---|---|
| D1 | Open as a draft | Nothing | No row | Run fires, skips |
| D2 | Close it without ever marking ready | Nothing | No row | Run fires, skips |

D2 is the case that used to leak a "Closed" card for a PR nobody ever saw announced.

---

## Janitor flow

Can't be tested honestly without waiting 90 days, so verify the query instead:

1. Open the janitor flow → **Run** → **Run flow**
2. Expand **Get items** → Outputs

Expect `"value": []` and Apply to each doing nothing. An empty array proves the `addDays` filter query is syntactically valid, which is the only real failure risk. A filter error shows up here as a red Get items.

---

## Cleanup

- Delete or close the four test PRs
- Confirm the SharePoint list is back to empty
- Delete the test cards from the Teams channel if you'd rather not keep them

---

## If a step fails

| Symptom | Look here first |
|---|---|
| Two cards for one PR | Concurrency Control not 1; or Create item's Title doesn't exactly match Get items' filter query |
| Card posts but never updates | Get items filter query - check the resolved value in run **Inputs** matches the row's Title byte for byte |
| Update fails on null Message Id | The post action isn't returning an ID, or MessageId's internal column name has `_x0020_` in it |
| `InvalidBotRequestMessageBody` | Copy the resolved card from the failed action's **Inputs** into adaptivecards.io/designer - it gives a line number |
| `InvalidBotRequestMessageBody` only on close/merge | Look for `"isVisible": True` in the resolved card. The expression is returning a real boolean instead of the quoted `'true'`/`'false'` strings |
| Card never collapses on close/merge | `isVisible` comparing `body('Parse_JSON')?['action']` to something other than the bare string `closed` |
| Card collapses but shows `Closed` on a merged PR | The one-liner's `equals(..., true)` on `merged` - it must compare to the boolean `true`, not `'true'` |
| Branch skipped unexpectedly | Open the condition in run results. Check the And/Or chip and look for stray empty rows |
| Row never deleted on close | `is-closed` comparing `body('Parse_JSON')?['action']` to the bare string `closed` |
| Nothing happens at all | Trigger condition. GitHub's **Recent Deliveries** tab should show 202; if it does, the payload arrived and the condition rejected it |
