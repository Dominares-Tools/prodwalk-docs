# Changelog

## 1.0.1

- Write guard: a `sendBeacon` or `keepalive` fetch sent as a page closes is now stopped and fails the walk in
  Chromium too (it was not routed there). `SharedWorker` is switched off unless `service_workers` is `allow`.
- Write guard: an `allow_writes` entry applies to the walked site only. A `307`/`308` answer to an allowed write
  is followed hop by hop by prodwalk, each hop held to the allow list, instead of re-sent by the browser unrouted.
- An anonymous role (no `sign_in`) no longer counts as signed in, so `signed_out` ignore rules apply to it.
- CSP violations are reported as they happen, so a walk step or hook that leaves the page navigating no longer
  aborts the run, and a violation on an earlier page is no longer forgotten.
- `w.fetch` is bounded from outside the page as well: a page that stops answering fails that one check and the walk
  goes on with a fresh page.
- Secrets are also cut in the forms JavaScript's `encodeURIComponent` and `encodeURI` produce, and with `/` kept. The
  full list: as written; percent-encoded strictly, with `/` kept, as `encodeURIComponent` does it, as `encodeURI`
  does it; form-encoded; JSON-escaped; HTML-escaped.

## 1.0.0

First release.

- `prodwalk run`: walks a live site in Chromium, WebKit or Firefox as several roles on several viewports.
- Write guard: every non-read request is aborted before the network, recorded as `METHOD /path`, and fails the run.
  Explicit `allow_writes` entries for the sign-in. Service workers blocked by default.
- Secrets read from the environment only, cut from every printed and stored line (as written, percent-encoded,
  form-encoded, JSON-escaped, HTML-escaped); sign-in errors withheld; screenshots mask password fields.
- Watchdog: exit 3 with the last finished step when no step finishes for `stall_seconds` or the walk passes `walk_minutes`.
- Console, page-error and CSP-violation checks, with exact forgiveness for WebKit's navigation-cancelled reads.
- Declarative steps (`goto`, `click`, `fill`, `press`, `wait`, `see`, `url_contains`, `url_matches`,
  `title_contains`, `fetch`, `screenshot`), plain HTTP checks, and a `walk(w)` hook for code.
- `prodwalk check` validates a walk file; `prodwalk demo` walks a built-in stand-in site (`--leaky` shows the guard).
- `--report` writes the results as JSON; `-q` prints only failures.
