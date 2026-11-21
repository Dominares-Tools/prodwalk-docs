# prodwalk — Smoke-test production as each user role

**Paid tool: $49 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/prodwalk**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

prodwalk drives a real browser (Chromium, WebKit or Firefox) through your production site as several roles on several screen sizes. Every non-read request except the writes you allow, such as the sign-in, is stopped before the network and fails the run; a beacon sent as a page closes is caught too, except on the walk's final page. Secrets come from the environment and are cut from every line it prints or stores. A watchdog ends a stalled walk with the last step that finished.

## Get it

Buy prodwalk at https://dominares.org/tools/prodwalk: one payment of $49, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the prodwalk command and Python hooks, a stand-in demo site, example walk files and the test suite.

Requirements: Python 3.10 or newer and Playwright, with a browser it can launch (pip install "./prodwalk[browser]" then playwright install chromium). Point it only at sites you are entitled to test.

This page documents version 1.0.1. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

### What it does

Smoke tests against production have two ways to go wrong. A click lands on "Delete", or a page quietly
sends a write of its own (a heartbeat, a read receipt, an analytics beacon), and now your check changed
live data. Or a failing sign-in step prints an error that quotes the password it was typing, and the
password is now in your CI log. prodwalk is built around not doing either.

- **It writes nothing.** Every HTTP request the browser makes goes through a guard. `GET`, `HEAD` and
  `OPTIONS` go on. Anything else (`POST`, `PUT`, `PATCH`, `DELETE`, a form submit, `sendBeacon`) is
  aborted inside the browser before it reaches the network, is recorded as `METHOD /path` (never the
  query string), and fails the run. You name the one or two writes that may pass, such as `POST /login`;
  an entry applies to the walked site only, so the same path on another site is still stopped. Two
  things the browser's request routing never sees get their own guard: a `sendBeacon` or `keepalive`
  fetch sent as a page closes (Chromium sends those without routing them), and a `SharedWorker` (switched
  off, because no engine routes what it sends).
- **It never prints a secret.** Passwords are read from environment variables you name in the walk file
  and nowhere else; the file holds the variable's name, never its value. Every line prodwalk prints, and
  every line it stores in a report, has each secret cut out (as written; percent-encoded, strictly or the way
  JavaScript's `encodeURIComponent` and `encodeURI` do it; form-encoded; JSON-escaped; and HTML-escaped). Sign-in steps run inside one `try` whose error is never read, because a failed
  Playwright `fill` writes the value it was typing into its call log. Screenshots mask password fields,
  and a screenshot step is refused inside sign-in.
- **It walks as several roles.** Each role signs in on its own browser context, then walks the pages
  meant for it, on each viewport and in each browser you list (Chromium, WebKit, Firefox). A page can
  be limited to some roles and some viewports, so "a viewer is offered no Save button" and "the phone
  menu is there" are one line each.
- **It cannot hang.** A watchdog on its own thread ends the run with exit 3 and the last step that
  finished when no step finishes for `stall_seconds` (300), or the whole walk runs past `walk_minutes`
  (90). Every in-page `fetch` is bounded by `fetch_seconds` (and, from outside the page, by one second
  more), and every browser action by `step_seconds`.
- **It reads what the page says.** Console errors, uncaught page errors and Content-Security-Policy
  violations fail the walk. WebKit's noise for a read that the walk's own navigation cancelled in flight
  is forgiven, and only that: the rule is exact (see [Console noise](#console-noise)).
- **One failing step does not end the run.** A step that throws fails its page with a one-line reason
  and the walk goes on to the next page, so one stuck click does not take every later check with it.

### The walk file

A walk is one JSON file. Start from `examples/site-roles.json` (three roles) or
`examples/anonymous-smoke.json` (no sign-in). `prodwalk check walk.json` validates the file, prints what it
would walk and which secret variables are set (never their values), and opens no browser.

```json
{
  "host": "https://example.com",
  "allow_writes": ["POST /login"],
  "http_checks": [{"name": "health", "path": "/health", "json": {"build.short": "${WALK_SHA}"}}],
  "roles": [
    {"name": "admin", "secrets": {"password": "ADMIN_PASSWORD"},
     "sign_in": {"path": "/login", "steps": [
        {"fill": "#user", "value": "admin"},
        {"fill": "#pw", "secret": "password"},
        {"click": "button[type=submit]"},
        {"see": "#app"}]}}
  ],
  "pages": [
    {"name": "Home", "steps": [{"goto": "/"}, {"see": "h1", "text": "Welcome"}]},
    {"name": "Phone menu", "viewports": ["phone"], "steps": [{"goto": "/"}, {"see": "#menu"}]}
  ]
}
```

With no `roles`, one anonymous role named `visitor` walks the pages. `${NAME}` in `host` and in an
`http_checks` JSON value reads the environment variable `NAME`; a missing one is a usage error.

#### Settings

| Key | Default | Meaning |
|---|---|---|
| `host` | required | The origin to walk, such as `https://example.com`. `--host` overrides it. |
| `browsers` | `["chromium"]` | Any of `chromium`, `webkit`, `firefox`. |
| `viewports` | laptop 1440x900, phone 390x844 | `{name, width, height}`. Phone means narrower than 600 px. |
| `allow_writes` | `[]` | Writes that may pass: `"POST /login"`, `"PUT /session/*"` (`*` matches within the path). |
| `guard_scope` | `all` | `all` stops writes to any site (a third-party beacon too); `origin` stops only writes to `host`. |
| `service_workers` | `block` | `allow` lets the site register a service worker and use a `SharedWorker`. See the limits. |
| `stall_seconds` | 300 | End the run (exit 3) when no step finishes for this long. |
| `walk_minutes` | 90 | End the run (exit 3) when the whole walk runs this long. |
| `fetch_seconds` | 20 | Bound on every in-page `fetch` and every plain HTTP check. |
| `step_seconds` | 30 | Bound on every browser action and every wait for a selector. |
| `ignore_console` | `[]` | Console lines that are not faults. See below. |
| `cancelled_reads` | `[]` | Readers whose "Load failed" is forgiven when their own address was cancelled. |
| `http_checks` | `[]` | Plain reads before any browser opens. See below. |
| `python` | none | A file defining `walk(w)`, run after the pages for each role. See below. |

#### Steps

Each step has exactly one action key. Add `"name"` to label its check.

| Step | Does | Records a check? |
|---|---|---|
| `{"goto": "/path"}` | Opens the page on `host` (a full URL on another site is refused). `"status": 404` expects that status; otherwise it must be below 400. | yes |
| `{"click": "sel"}` | Clicks. Any write it causes is stopped. | no, unless it throws |
| `{"fill": "sel", "value": "x"}` or `{"fill": "sel", "secret": "password"}` | Types a literal, or a secret declared on the role. | no, unless it throws |
| `{"press": "Enter", "selector": "sel"}` | Presses a key (on the page when no selector). | no, unless it throws |
| `{"wait": 1.5}` | Waits up to 60 seconds. | no |
| `{"see": "sel"}` | The selector is visible. Add `"state"` (`visible`, `attached`, `hidden`, `detached`), `"text"`, `"matches"` (regex), `"count"` (exact; `0` means absent), `"min"`, `"timeout"` (seconds). | yes |
| `{"url_contains": "x"}`, `{"url_matches": "re"}`, `{"title_contains": "x"}` | The address or the title, waited for like a selector (`"timeout"` in seconds, default `step_seconds`). | yes |
| `{"fetch": "/api/me", "status": 200, "json": {"a.b": 1}, "contains": "x"}` | A GET from inside the page, with its cookies. `json` maps a dotted path to the value it must hold. | yes |
| `{"screenshot": "name"}` | Saves a PNG under `--screenshots`; a note when no folder was given. | no |

An absence check (`"count": 0`) passes at once if the element is not there, so put a presence check
for something on the same page before it.

#### Roles and sign-in

```json
{"name": "viewer", "secrets": {"password": "VIEWER_PASSWORD"},
 "sign_in": {"path": "/login", "steps": [ ... ]}}
```

`secrets` maps a name you use in `fill` to the environment variable that holds it. A variable that is
not set fails that role by name (`... VIEWER_PASSWORD is in the environment`) and the other roles still
walk. The sign-in passes when every step in it completes, so end it with a `see` for something only a
signed-in page shows. A sign-in that fails prints `signs in` as FAIL and nothing about why beyond that
the error is withheld.

#### Plain HTTP checks

`http_checks` run first, with no browser and no cookies, as an anonymous visitor's first request would.
Each takes `path` and any of `name`, `status` (default 200), `headers` (name to regex), `body_contains`,
`body_matches` (regex), `json` (dotted path to value; a `${NAME}` value reads the environment) and
`max_seconds`. They follow redirects.

#### Console noise

`ignore_console` rules: `{"message": "regex", "url": "regex on the path", "phase": "any | signed_out | signed_in"}`.
A boot script that probes the session before sign-in and gets a 401 is not a fault; the rule above
excuses exactly that, before sign-in, for that path.

Separately, WebKit logs `Fetch API cannot load <url> due to access control checks` for a read the
outgoing page starts after the walk's own navigation began, and `TypeError: Load failed` for a read the
navigation cancelled. Both are forgiven only inside one of the walk's own navigations, for this origin,
and only when every send of that address inside it was then cancelled. The same message at any other
moment, a refusal for another origin, or a send nothing cancelled still fails the walk. `cancelled_reads`
names the readers whose own `Load failed` message you want to forgive, each tied to the one address it reads:
`{"message": "^palette: could not load settings", "path": "/api/settings"}`.

#### A walk in code

For anything the steps cannot say, set `"python": "mywalk.py"`. The file defines `walk(w)`, which runs
after the pages for each browser, viewport and role, on the same guarded, watched page:

```python
def walk(w):
    w.goto("/reports")
    status, body = w.fetch("/api/reports")          # GET from inside the page, bounded
    w.check("three reports are listed", status == 200 and len(body["items"]) == 3)
    w.screenshot("reports")
```

`w` also has `page`, `context`, `role`, `tag`, `phone`, `host`, `note(name, why)` and `secret(name)`.
The file is ordinary Python and runs with your permissions: only point `python` at code you wrote.

### Options

```
prodwalk run WALK.json  [--host URL] [--browsers a,b] [--screenshots DIR] [--report FILE]
                        [--stall-seconds N] [--walk-minutes N] [--quiet] [--headed]
prodwalk check WALK.json [--host URL]
prodwalk demo           [--leaky] [--port N] [--print-spec] [same options as run, minus --host]
```

| Option | Meaning |
|---|---|
| `--host` | Walk this origin instead of the file's `host`. |
| `--browsers` | Overrides the file's list. |
| `--screenshots` | Saves PNGs here, named `<browser>-<WxH>-<role>-<name>.png`. Keep the folder out of git: a signed-in page can show real data. |
| `--report` | Also writes `{exit_code, total, failed, results, notes}` as JSON, secrets already cut out. |
| `--stall-seconds`, `--walk-minutes` | Override the file's watchdog limits. |
| `-q`, `--quiet` | Prints only FAIL, NOTE and the summary. |
| `--headed` | Shows the browser window. |
| `--leaky` | `demo` only: the stand-in page sends a write, to show the guard. |
| `--print-spec` | `demo` only: prints the walk file the demo uses. |
| `--port` | `demo` only: the stand-in site's port (default: any free one). |

Exit codes: `0` every check passed, `1` a check failed, `2` the walk file or the setup is wrong (also: Playwright not
installed), `3` the watchdog ended a stalled or overlong run.

Output is one line per check, then `N/M passed`:

```
PASS  chromium-1440x900 viewer: Viewer is read-only: a viewer is offered no Save button
FAIL  chromium-1440x900 admin: the walk wrote nothing (a write is stopped before the network)  (POST /api/ping)
```

### Limits

- **It stops what a browser sends over HTTP.** It does not see WebSocket messages, and a `GET` that
  changes data on the server (a link that deletes) is a read to the guard. Walk a site whose reads are reads.
- **Service workers and shared workers are blocked by default**, because Playwright cannot route the
  requests they make (a page that checks for `SharedWorker` sees it missing). With
  `"service_workers": "allow"` the guard covers the page's own requests but not those workers'.
- **An allowed write is followed by prodwalk, not by the browser.** A `307` or `308` answer would make
  the browser send the same write again to the new address without routing it, so prodwalk follows up
  to five such hops itself and holds each one to the allow list; a hop to a path that is not allowed is
  stopped and fails the run. WebKit refuses to hand a redirect back to the page, so there an allowed write
  that is answered with a redirect reaches the page as a short page that refreshes to the new address: a
  form post behaves as before, a script's `fetch` of it sees a 200.
- **A write attempted only as the last page closes is not heard.** The guard reports what a page tries
  as it closes when a later page of the walk loads, so the walk's final page has no later page. Nothing is
  sent for it either way (the request is a plain fetch that a closing page abandons).
- **`guard_scope: origin` lets writes to other sites through.** The default, `all`, stops them.
- **An error elsewhere on the page can still contain a secret.** prodwalk cuts out every secret it was
  given, in the forms listed above. A value the site derives from a secret (a hash, a token it issues)
  is not a secret prodwalk knows.
- **Screenshots are real pages.** Password fields are masked; everything else is not.
- **A page that stops answering costs one check, not the run.** `fetch_seconds` aborts a fetch inside the
  page; a renderer that stops answering altogether fails that one fetch after one more second, the page is
  closed and a fresh one (`w.page`) takes its place in the same session. Any other browser call that never
  returns is caught by `stall_seconds`, with exit 3.
- **It needs a browser it can launch.** `playwright install chromium` (and `webkit`, `firefox`) fetch them.
  A browser that cannot start is a FAIL line that names the command to run.
- It runs one role, viewport and browser at a time, signing in for each. It does not parallelise.
- Python 3.10 or newer. Walking needs the `browser` extra (Playwright); validating a file, the plain
  HTTP checks and the reports need only the standard library.

### Development

```bash
playwright install chromium webkit firefox
python -m pytest
```

The tests run real browsers against stand-in sites on 127.0.0.1.

---

Docs mirror of the private repository Dominares-Tools/prodwalk; updated with each release.
