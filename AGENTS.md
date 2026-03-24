# AGENTS.md

This document describes my coding philosophy and preferences, for the benefit of AI coding assistants and humans working in this repo.

## Principles

* Follow Unix philosophy.
* Data > protocol > code
* Code should be minimalist, human readable. Make tradeoffs in favour of readability/simplicity over performance/edge cases. Mark edge cases as TODO. Raise complexity only on request, or when unavoidable.
* Occam's razor: Do not multiply entities beyond necessity.
  * State: don't shadow or duplicate state (caches are fine). Custom state lifecyle is often undocumented, forgotten and abandoned. e.g. To identify a client to a server, prefer already-existing ssh host keys to inventing new client certificates, or per-host tokens.
  * Code: Pipelines of shell tools > small, high quality, MIT/BSD licensed code > custom code.
* Keep code and trees shallow, comfortable for ls, vim, diff and grep. Typically just one or two levels deep: README, code, config at the top level, subdirectories for docs, tests, static assets. Avoid Java-like sprawl.

## File and data conventions

* Prefer human readable line-oriented UTF-8 text for config files, data interchange between programs. One record per line, easily processed with grep/awk/cut. e.g. CSV > XLS
* CSV: header row, RFC4180-ish, commas only.
* When parsing would be complex, prefer JSON to XML; pretty-print JSON (2 spaces) with stable key ordering.
* For atomic changes to a single text file, use the copy/edit/rename trick. For atomic changes across multiple files, use a database, sqlite for choice.
* Date format: Choose formats like YYYYMMDDHHMMSSZ (Z for UTC) for storing in text files - this enables easy numeric comparison in shell scripts.
* Logging: one line per record, UTC timestamps.

## Shell scripts

* Lean on `bash`, coreutils, `jq`, `curl`, `openssl`, `sqlite3`.
* Keep scripts as small and readable as possible. Don't overengineer 100 line scripts where 2-3 lines would suffice.
* Have a single line comment showing the usage.
* It's OK to not validate and fail fast, let the failure bubble upwards via exit status

e.g. if asked to create a script to concatenate pdfs using qpdf, this is what I expect:
```
#!/bin/bash
# Usage: pdfcat 1.pdf 2.pdf 3.pdf > combined.pdf
qpdf --empty --pages "$@" -- -
```
What I don't want: 100 lines of checking OS, checking if qpdf is installed, checking if the files exist, checking if output path exists, complaining loudly if any validation fails, etc.

Another example: script to check if a given user is present in the active user list
```
#!/bin/bash

USERLIST=/etc/freeradius/active-users.txt
USER="${1:-nonexistent}"
grep "$USER"  $USERLIST >/dev/null 2>&1
res=$?
echo "$(date): got $USER $res" >>/var/log/freeradius/check_active_user.log
exit $res
```
Note the lack of validation of arguments, existence of userlist file - failure bubbles via exit code.

## Documentation

* Inline comments should explain high level intent, non-obvious things, not obvious mechanics.
  * Good comment: `# Return password page with download link (no-cache to prevent stale tokens)`
  * Good comment: `session['id_token'] = token.get('id_token')  # Needed for step-ca OIDC provisioner`
  * Bad comment: `session.clear()  # Clear session`
  * Bad comment:
    ```
    # Base64 encode the P12 bundle
    p12_base64 = base64.b64encode(p12_data).decode('utf-8')
    ```

## Web apps

* For simple portals, prefer Python/Flask (fits the simplicity and code layout requirement above).
  * Use flask-session and a tmpfs like /run for session data
  * Use Talisman, Flask-WTF, enforce https for production, configure CSP with nonces for inline scripts and styles
  * Use gunicorn, nginx reverse proxy for production deployment
* For simple service liveness checks, prefer an unauthenticated `/ping` endpoint that returns fixed `200 OK` with a small fixed plain-text body like `ok\n`.

## Testing

* For Python: pytest -q, no mocks unless necessary, golden files for line-oriented tools.

## Commit/PR

* For commit messages, use "area: verb in present tense" (e.g., flask: add CSP nonces). Keep commit messages brief.
