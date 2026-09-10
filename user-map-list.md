# GitHub user map

Maps a GitHub login to a person's real name, so the card reads
`Obie Munoz (obiemunozjr)` instead of just `obiemunozjr`.

## Why a list is necessary

The webhook payload can't supply the name. `pull_request.user` is GitHub's *simple user*
object - `login`, `id`, `avatar_url`, `html_url`, `type`. The real `name` lives only on
the full profile from `GET /users/{login}`, which is an API call, which needs a premium
connector.

The Office 365 Users connector is Standard and would resolve a display name happily, but
it needs an email or UPN to search on and the payload has neither. Circular.

So the mapping has to be maintained by hand somewhere. A list means anyone with access
can maintain it without opening the flow.

## Columns

| Display name | Internal name | Type | Purpose |
|---|---|---|---|
| GitHub Login | `Title` | Single line of text | The key. Index it. |
| DisplayName | `DisplayName` | Single line of text | `Obie Munoz` |
| Email | `Email` | Single line of text | Unused today. Enables @mentions later - the Teams connector's `Get an @mention token for a user` takes an email. |

Same internal-name rules as the card index: create `DisplayName` and `Email` with **no
spaces** in the name, and use the built-in `Title` column for the key.

Example rows:

| GitHub Login | DisplayName | Email |
|---|---|---|
| `obiemunozjr` | Obie Munoz | obie@example.com |
| `dependabot[bot]` | Dependabot | |
| `somenewdev` | *(blank - self-seeded, needs filling in)* | |

## No janitor needed

Unlike the card index, this is a slowly-growing reference table: one row per person,
permanent, a few hundred rows at most. Nothing to clean up.

## Self-seeding

When the flow meets a GitHub login that isn't in the list, it writes a row with a blank
`DisplayName`. The card still renders correctly - an unmapped or blank name falls back to
the bare login - but the row now exists as a to-do.

Create a list view filtered to `DisplayName is empty` and that view *is* your backlog.
Nobody has to work out who's missing.

Bot accounts seed too. `dependabot[bot]` will appear as a row; give it a friendly
DisplayName and it renders like anyone else.

## Rendering rules

Verified against every case:

| DisplayName | Card shows |
|---|---|
| `Obie Munoz` | Obie Munoz (obiemunozjr) |
| *no row at all* | somenewdev |
| *blank* | seeded-user |
| `Siobhan O'Brien` | Siobhan O'Brien (sobrien) |
| `Bob "Bobby" Tables` | Bob "Bobby" Tables (btables) |
| `José Muñoz` | José Muñoz (jmunoz) |
| `Dependabot` | Dependabot (dependabot[bot]) |

The quote case works for free because the author line goes through the same
`slice(string(array(...)))` serialiser as the PR title. Names are user-supplied text and
get the same treatment.

## Flow changes

Two actions, both immediately after `Parse JSON` and before `first-time-seeing-pr`, so
the post and update branches both see the result.

| Step | Action | Config |
|---|---|---|
| 1 | Get items → rename to **`Get user`** | List: user map. Filter Query: `expressions/get-user-filter-query.txt`. Top Count `1`. |
| 2 | Condition **`seed-user`** | `expressions/seed-user-condition.txt` · is equal to · `0` |
| 2a | ↳ True → Create item → rename to **`Create user row`** | List: user map. Title = `body('Parse_JSON')?['pull_request']?['user']?['login']`. Leave DisplayName and Email blank. |
| 2b | ↳ False → *(empty)* | |

Then in the card, the author `TextBlock` uses `expressions/author-expression.txt`, with
**no surrounding quotes** - the expression supplies its own.

### Naming

**Do not rename the existing `Get items`.** Many expressions reference
`body('Get_items')` and renaming it breaks all of them at once.

Rename the *new* actions instead - `Get user`, `Create user row` - so you don't end up
with `Get items 1` and `Create item 1` and have to remember which is which.

### Ordering

The seed writes a row *after* `Get user` has already read, so the freshly seeded row
isn't in `Get_user`'s output. That's fine and deliberate: a new row has a blank
DisplayName anyway, so the card falls back to the bare login either way. No second read
needed.
