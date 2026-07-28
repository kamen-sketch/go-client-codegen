# Security findings

A findings pass, as opposed to [`TAINT-MAP.md`](TAINT-MAP.md), which maps *reachability* — which untrusted
input can reach which dangerous operation. This document covers what that map does not: credential handling,
resource exhaustion, CI supply chain, and the security properties of the library's public surface. Findings
here are numbered `S*`; the map's are `F*` and are not repeated.

Everything marked **confirmed** was reproduced against the real client or verified against the dependency
source, with the observed output quoted. The reproductions were removed rather than committed — they assert
the absence of issues that are still present and would fail CI.

| #  | Finding                                                              | Severity | Status |
| -- | -------------------------------------------------------------------- | -------- | ------ |
| S1 | No HTTP request timeout anywhere in the client stack                  | Medium   | confirmed |
| S2 | Unpinned `go install …@latest` in a CI job holding secrets and write scope | Medium | confirmed |
| S3 | API secrets handed to a build step that does not use them             | Low–Med  | confirmed |
| S4 | Server-controlled newlines reach error strings (log forging)          | Low      | confirmed |
| S5 | Credential-bearing DTOs have no redaction                             | Low      | confirmed |
| S6 | `Permissions()` cannot distinguish "none required" from "unknown"     | Low      | confirmed |
| S7 | No security linter runs in the static analysis stack                  | Info     | confirmed |

---

## S1 — No request timeout anywhere — Medium

A server that accepts the connection and then never responds hangs the caller **forever**:

```
--- FAIL: TestSec_NoRequestTimeout (5.01s)
    still hanging after 5s with no ctx deadline: no client timeout is configured
```

The stack sets every timeout except the one that bounds a whole request:

- `retryablehttp.NewClient()` uses `cleanhttp.DefaultPooledClient()`, which sets `Dial: 30s`,
  `TLSHandshakeTimeout: 10s`, `IdleConnTimeout: 90s` — and leaves `http.Client.Timeout` at zero
- `StandardClient()` (`retryablehttp/client.go:920`) returns `&http.Client{Transport: …}` — also zero

So the response-read phase is unbounded. A slow-loris peer, a hung load balancer, or a TCP black hole after
the handshake holds the goroutine and the pooled connection indefinitely. `NewClient` exposes no option to
set one: `RetryMaxOpt`, `RetryWaitMinOpt` and `RetryWaitMaxOpt` govern retry pacing, not request duration.
The only escapes are a per-call `context` deadline on every one of the 395 operations, or replacing the whole
`Doer` via `DoerOpt` — and `DoerOpt` also discards all retry behaviour, which is easy to do by accident.

`RetryMax` defaults to 6, so against a server that is slow but does eventually respond, the worst case
multiplies by seven.

**Fix:** add a `TimeoutOpt` (and `AIVEN_CLIENT_TIMEOUT`) with a non-zero default, applied to the
`retryablehttp` inner client so it bounds each attempt rather than the whole retry sequence.

## S2 — Unpinned tool install in a job with secrets and write access — Medium

`.github/workflows/update.yml`:

```yaml
- run: go install golang.org/x/tools/cmd/goimports@latest    # unpinned
- run: go install github.com/vektra/mockery/v3@v3.5.5        # pinned
- uses: arduino/setup-task@v3
- run: task generate
  env:
    AIVEN_TOKEN: ${{ secrets.AIVEN_TOKEN }}
    AIVEN_PROJECT_NAME: ${{ secrets.AIVEN_PROJECT_NAME }}
```

`task generate` runs `fmt-imports`, which invokes `goimports` — so the binary resolved by `@latest` executes
as a subprocess of the step that has both secrets in its environment, inside a job granted `contents: write`
and `pull-requests: write`.

The module proxy and checksum database protect the integrity of *published* versions; they do not protect
against a malicious *new* version, which is exactly what `@latest` opts into. A compromised release of that
module would run with the Aiven API token in its environment and the ability to push commits and open pull
requests against this repository.

`golang.org/x/tools` is Google-maintained, so the likelihood is low — but the inconsistency is the tell:
mockery is pinned on the very next line. Pin `goimports` to a version and bump it deliberately.

Lower priority, same class: `actions/checkout@v7`, `actions/setup-go@v7`, `arduino/setup-task@v3`,
`peter-evans/create-pull-request@v8` and `anothrNick/github-tag-action@1.75.0` are all mutable tags rather
than commit SHAs. Standard practice, and acceptable for most repos, but this workflow's permissions are above
average.

## S3 — Secrets handed to a step that does not use them — Low–Medium

Verified: **nothing in the generation path reads any `AIVEN_*` variable.** The generator's `envconfig` uses
the `GEN` prefix (`generator/main.go:31`), and `get-openapi-spec` fetches a public URL with no auth:

```bash
curl -s -o openapi.json https://api.aiven.io/doc/openapi.json
```

A grep for `AIVEN` across `generator/`, `Taskfile.yml` and `.mockery.yaml` returns nothing. Both secrets are
therefore inherited by every subprocess of `task generate` — `task` itself, `go run`, `mockery`, and the
unpinned `goimports` from S2 — for no functional reason.

**Fix:** delete the `env:` block. It is a one-line change that removes the credential from S2's blast radius
entirely, which is why it is worth doing even though the workflow is not currently broken.

## S4 — Server-controlled newlines reach error strings — Low

`Error.Error()` interpolates server-supplied text into a single-line format string with no sanitization.
Two of the three paths carry raw newlines through:

```
message field:   "[400 ServiceGet]: bad\nERROR injected-via-message\nlevel=fatal"
non-JSON body:   "[400 ServiceGet]: gateway failure\nERROR injected-via-plaintext\nlevel=fatal"
```

Any consumer that writes `err.Error()` to a line-oriented log (logfmt, syslog, plain `log.Printf`) gets
forged log lines with attacker-chosen severity fields. Not an issue for the client's own debug logger, which
is zerolog and JSON-encodes, but the error is public API and most of its consumers are not.

Worth noting the contrast, because it shows the fix is cheap: the `errors` field is **safe**, because
`Error()` re-marshals it with `json.Marshal`, which escapes newlines. Only `Message` — assigned from the
unmarshalled `message` field at `error.go:65`, or copied verbatim from a non-JSON body at `error.go:74` —
reaches the output raw.

**Fix:** collapse `\r` and `\n` in `Message` when formatting, and bound its length (which also addresses F2).

## S5 — Credential-bearing DTOs have no redaction — Low, wide blast radius

```go
out, _ := cl.ServiceGet(ctx, "proj", "svc")
fmt.Sprintf("%+v", *out)
// → …ServiceUri:postgres://avnadmin:SUPERSECRET@h:1/db…
```

Confirmed. Generated DTOs carry 40 credential-bearing fields — 12 `password`, plus `token`, `tokens`,
`access_key`, `service_uri` and friends — and the generator emits no `String()` or `GoString()` method, so
the default `%v`/`%+v` formatting prints them in full.

Printing a struct is the single most common debugging reflex in Go, and it appears in error paths that ship
to production. The impact is on consumers of this library rather than the library itself, which is precisely
why the library is the right place to fix it: one generator change protects every consumer.

**Fix:** have `writeStruct` emit a redacting `String()` for any struct containing a field whose name matches
a sensitive-name list. The field remains accessible; only the default formatting changes.

## S6 — `Permissions()` conflates "none required" with "not documented" — Low

`Permissions()` returns operation ID → required roles, embedded from `permissions.yaml`. Coverage:

```
operations:            395
with permission data:  275
without any entry:     120
```

The missing 120 include `AccessTokenCreate`, `AccountDelete`, `AccountCreate` and similar — operations that
plainly do require privilege. And `readPermissions` (`generator/permissions.go:44-47`) actively *deletes*
entries whose list is empty, so an operation genuinely requiring nothing is stored identically to one nobody
has documented.

A consumer using this map to gate a UI, pre-flight a call, or drive a policy engine — the obvious reasons to
expose it — will read absence as "no permission required" and get it wrong for 30% of the API.

**Fix:** either document `Permissions()` as advisory and incomplete, or make the distinction explicit by
keeping empty lists rather than deleting them.

## S7 — No security linter runs — Informational

`.golangci.yaml` sets `default: none` and enables 18 linters. `gosec` is not among them; it appears only
inside an exclusion rule scoped to `_test.go`, which has no effect since the linter never runs in the first
place. The exclusion reads as though gosec were enabled, which is how this stays unnoticed.

CodeQL (`codeql-analysis.yml`) does run on every PR and push, so the gap is partial rather than total.

---

## Verified not vulnerable

Recorded so these are not re-investigated:

- **The token is stripped on cross-host redirects.** Confirmed empirically — a redirect from one host to a
  different hostname does not carry `Authorization`, while a same-host redirect does. This is Go's standard
  `shouldCopyHeaderOnRedirect` policy and it applies correctly here, including through
  `retryablehttp.StandardClient()`. Note the standard caveat: the header *is* forwarded to subdomains of the
  original host.
- **No `pull_request_target` anywhere in `.github/`** — the workflow trigger most commonly responsible for
  pwning a repository is not used.
- **Workflow permissions are scoped.** Every workflow declares `permissions: read-all` at the top level and
  elevates per job (`contents: write` only in `release` and `update`, `security-events: write` only in
  CodeQL, `checks: write` only for Trunk annotations).
- **No TLS weakening.** No `InsecureSkipVerify`, no custom `tls.Config`, no `MinVersion` downgrade anywhere
  in the module.
- **The `errors` field cannot forge log lines** — it is re-marshalled through `json.Marshal`, which escapes
  control characters. Only `Message` is raw (S4).
- **Singleflight cannot mix credentials** (repeated from the taint map because it is the first thing a
  reviewer looks for): the key omits the token, but the group is a per-`aivenClient` field.

## Suggested order

| # | Change | Addresses | Size |
| - | ------ | --------- | ---- |
| 1 | Delete the `env:` block from the `task generate` step | S3, shrinks S2 | one line |
| 2 | Pin `goimports` to a version | S2 | one line |
| 3 | Sanitize newlines and bound length in `Error.Message` | S4, F2 | a few lines |
| 4 | Add `TimeoutOpt` / `AIVEN_CLIENT_TIMEOUT` with a non-zero default | S1 | small |
| 5 | Enable `gosec` in `.golangci.yaml` | S7 | one line, then triage |
| 6 | Generate redacting `String()` for secret-bearing structs | S5 | generator change |
| 7 | Document or fix `Permissions()` completeness | S6 | decision first |

Items 1–3 and 5 are each a handful of lines and carry no API impact. Item 4 changes runtime behaviour for
existing callers — a previously-hanging call starts erroring — so it wants a release note.
