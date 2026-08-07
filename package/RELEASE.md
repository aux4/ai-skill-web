# Release notes

## Fill a form in one call

`aux4 browser fill` fills every field of a form in a single call, reading each field's type off
the page. The skill now leads with it, because choosing between `type`, `select` and `check` per
field is the most common way a form fill goes wrong:

```bash
aux4 browser fill --session <id> \
  --field "Full name=Sally Reed" \
  --field "Role=engineer" \
  --field "Subscribe to the newsletter=yes"
```

The single-field commands are still documented, as what `fill` uses underneath.

## `open` starts a session; `visit` changes page

The skill described `open` as the way to reach a URL and paired it with `visit` as though they
were interchangeable. They are not: `open` always creates a **new** session with no cookies, so
calling it again silently discards whatever you were signed in to.

An agent following the old wording would sign in, call `open` to "go to" the next page, land back
on a login wall, and sign in again — repeatedly, until it ran out of budget. The lifecycle step
now says plainly that `open` happens once and `visit --session <id>` is how you move.

## Read the page before acting, and check what happened after

Two corrections to the same failure — acting on what the task said rather than what the page says:

- **Before**: snapshot first and use the names and refs that are actually there. A task saying
  "full name" does not mean the field is called `full_name`.
- **After**: every command reports its own result. `filled: 0` or an unexpected `finalUrl` means
  the page is not what you assumed — re-read it, rather than continuing the plan or concluding
  the task is impossible. A form is submitted only once you have clicked its button and seen the
  page change.

## What an HTTP status means for the next step

Navigation reports an `httpStatus`, and each one calls for a different response — retry, change
approach, or stop:

| status | meaning | what to do |
|---|---|---|
| `401` | not signed in | sign in; ask for credentials or create an account if you have none |
| `403` | not permitted | that route is closed — do not retry |
| `404` | not there | if you assembled the URL, go back and click your way instead |
| `429` | too fast | wait, then retry |
| `5xx` | server-side | retry once, then stop and say so |

Retrying helps for `429` and `5xx` only.

### Notes

- Signing up does not sign you in. The `401` guidance says so explicitly, because landing back on
  a sign-in page after registering reads as success and is not.
