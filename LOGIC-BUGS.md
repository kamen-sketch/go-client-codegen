# Business logic review

Correctness bugs — behaviour that is wrong or surprising regardless of whether an attacker is involved.
Complements [`TAINT-MAP.md`](TAINT-MAP.md), which covers the security-reachability question, and
[`ARCHITECTURE.md`](ARCHITECTURE.md).

Every finding marked **confirmed** was reproduced against the real client with a temporary test file; the
observed output is quoted. Those tests were removed again rather than committed, since they assert the
absence of bugs that are still present and would fail CI. They belong with the fixes.

| #  | Finding                                                        | Status    | Impact |
| -- | -------------------------------------------------------------- | --------- | ------ |
| B1 | `limit=999` is injected into every HTTP method, not just GET     | confirmed | High   |
| B2 | 64 list operations can truncate at 999 with no way to detect it | confirmed | High   |
| B3 | Non-idempotent POSTs are replayed on 5xx and on 404             | confirmed | Medium |
| B4 | An empty 200 body becomes `unexpected end of JSON input`         | confirmed | Medium |
| B5 | Singleflight runs the shared request on the first caller's context | confirmed | Medium |
| B6 | Field name collisions silently drop a field, warning only        | latent    | Medium |
| B7 | Naive depluralization produces misspelled public type names      | confirmed | Low    |
| B8 | `IsAlreadyExists` requires an exact English substring            | confirmed | Low    |

---

## B1 — The default limit is applied to every method — High

`client.go:222` says:

```go
// Add default limit for GET requests if conditions are met
const defaultLimit = "999"
if shouldAddDefaultLimit(operationID, q) {
```

`shouldAddDefaultLimit` checks two things: that the caller did not already set `limit`, and that the
operation is not one of two hardcoded names. It never checks the method — and it *cannot*, because
`fmtQuery(operationID string, query ...[2]string)` is never told what the method is. `Do` knows the method
and does not pass it.

Reproduced:

```
DELETE /v1/project/proj/service/svc?limit=999
POST   /v1/project?limit=999
GET    /v1/project/proj/service/svc?limit=999
```

Every mutating request in the client carries `limit=999`. Most backends ignore an unknown query parameter,
which is why this has survived — but the denylist is the tell:

```go
operationsWithoutLimit := []string{
    "ServiceKafkaQuotaDescribe",
    "ServiceKafkaQuotaDelete",   // <- a DELETE
}
```

Those two entries are a symptom-level patch for exactly this bug, and one of them is a DELETE. Any future
endpoint that validates or interprets `limit` breaks in production until someone notices and appends another
string to that slice.

**Fix:** thread the method into `fmtQuery` and gate on `GET`/`HEAD`. That also makes both denylist entries
redundant — `ServiceKafkaQuotaDelete` is covered by the method check, and `ServiceKafkaQuotaDescribe` is the
only genuine exception left.

## B2 — Silent truncation of list results — High

The `limit=999` hack has a second consequence that the `TODO` acknowledges but does not size. Of the list
operations in the client:

- **64** return a bare slice (`ListClouds(ctx) ([]CloudOut, error)`), carrying no count and no cursor
- **11** return a struct that can expose `total_count`, `next`, `first`, `last`, `prev`

For the 64, a caller whose project holds more than 999 of something receives exactly 999 items and **no
indication whatsoever** that the list was cut off. There is no error, no flag, no count to compare against.
The result is indistinguishable from a complete list, so callers that iterate to reconcile state — the
sweeper pattern in `CONTRIBUTING.md` is exactly this — will silently skip resources.

The bare-slice shape comes from `getResponse` (`generator/main.go:586`): a response object with exactly one
property is unwrapped and the inner field returned directly. That is good ergonomics for `{"clouds": [...]}`,
but it is also what discards any sibling pagination metadata the moment the API grows some.

Only two operations expose a cursor at all (`OrganizationGovernanceAccessListCursor`,
`UpgradePipelineStepListCursor`), and both were reachable only because their responses happen to have more
than one property.

**Fix:** this needs a real decision, not a patch. Either surface truncation (return the count alongside, or
error when `len(results) == limit`), or implement cursor following in `Do`. At minimum, document the 999 cap
in the README so callers know to pass their own `limit` and paginate.

## B3 — Non-idempotent requests are replayed — Medium

Confirmed against a server returning 500 to everything, with `RetryMaxOpt(3)`:

```
POST attempts on repeated 500: 4
```

`retryablehttp`'s `ErrorPropagatedRetryPolicy` retries any 5xx regardless of method, and `checkRetry`
(`retry.go:19`) keeps that behaviour and adds to it. A `POST /project` that fails with 500 *after the
write committed* — a timeout in a downstream component, a proxy 502 on the response leg — is replayed up to
`RetryMax` times and can create duplicate resources.

The client also adds a POST-specific 404 retry:

```
POST attempts on 404 service-lag: 4
```

`isServiceLagError` (`retry.go:84`) retries `POST` when a 404 body matches `Service \S+ does not exist`. That
rule is defensible on its own terms — a 404 means the write did not land, so replaying is safe, and it exists
to paper over service-creation lag. The 5xx case is the risky one, because a 5xx carries no such guarantee.

**Fix:** restrict 5xx retries to idempotent methods (GET/HEAD/PUT/DELETE), or keep POST retries only where
the response proves nothing happened (the 404 rule). If POST-on-5xx must stay, it needs an idempotency key,
which the Aiven API would have to support.

Worth noting for anyone writing tests here: `DoerOpt` replaces the whole `Doer`, so the `retryablehttp`
client is never constructed and **no retry logic runs at all**. My first attempt at this test measured 1
attempt and looked like a pass. Retry behaviour can only be exercised against a real server.

## B4 — An empty 200 body is a parse error — Medium

```go
_, err = cl.ServiceGet(ctx, "proj", "svc")   // server returns 200 with an empty body
// err = unexpected end of JSON input
```

Generated handlers unconditionally `json.Unmarshal(b, out)` on any 2xx. `fromBytes` passes an empty body
through as success, so the failure surfaces from the JSON decoder with no operation context — the caller sees
`unexpected end of JSON input` and cannot tell which call produced it or that the request actually succeeded.

The generator has `// todo: support 204` at `main.go:219` and skips operations declaring neither a 200-JSON
nor a 204 response. But an operation *declared* with a 200 JSON schema that returns an empty body at runtime
— common for accepted-but-no-content and no-op updates — hits this path.

**Fix:** in the generated method, return the zero value when `len(b) == 0`; or have `fromBytes` normalise an
empty 2xx body to `null` so the unmarshal succeeds.

## B5 — Singleflight uses the first caller's context — Medium

```go
v, serr, sh := d.singleflight.Do(key, func() (any, error) {
    statusCode, body, err := d.do(ctx, method, path, in, queryString)   // ctx is whoever got here first
    return result{statusCode: statusCode, body: body}, err
})
```

Confirmed: with caller A and caller B sharing one flight, cancelling only A fails B —

```
B (never cancelled) -> err = Get "http://…/v1/project/proj/service/svc?limit=999": context canceled
```

The same applies to deadlines: a caller with a 100 ms timeout that happens to start the flight imposes that
timeout on every caller sharing it, including ones with a generous one. Because `EnableSingleFlight` defaults
to `true`, this is on by default, and it makes an unrelated request's cancellation look like a spurious
failure in yours. Under load — the exact situation dedup exists for — the odds of sharing go up.

`client_test.go` covers dedup, HTTP-level dedup, project isolation and method isolation, but no test cancels
one participant.

**Fix:** run the shared call on `context.WithoutCancel(ctx)` with a deadline derived from client config
rather than from the caller, so a departing caller cannot poison the flight.

## B6 — Field collisions drop a field with only a warning — latent, Medium

`fmtStruct` (`main.go:545-555`) maps Go field name → JSON name, last writer wins:

```go
if exist, ok := uniqueNames[goName]; ok {
    log.Warn().Msgf("Found field collision: %q overrides %q -> %q", p.path(), exist, jsonName)
}
// WARNING: This is a hack to avoid name collisions in Go fields
uniqueNames[goName] = jsonName
```

When two JSON properties camel-case to the same Go identifier, one is **removed from the struct entirely** —
it cannot be sent and cannot be received. The only signal is a `log.Warn` during generation, which in the
nightly `update.yml` run scrolls past in the job log. Nothing fails, and the resulting PR looks like an
ordinary schema bump with a field quietly missing.

I checked the case the comment cites and it is no longer live: `topics.blacklist` and `topics_exclude` map to
`TopicsBlacklist` and `TopicsExclude`, which do not collide, and `topics.exclude` is gone from the current
schema. So there is no field being dropped today — but the mechanism is armed, and the deterministic tie-break
(sort by the name with non-word characters stripped, descending) is subtle enough that which field survives is
not obvious from reading the code.

**Fix:** make a collision fail generation, with an explicit override map in `openapi_patch.yaml` for the cases
that are genuinely intentional. A loud failure in the update PR is far cheaper than a silently missing field.

## B7 — Misspelled public type names from naive depluralization — Low

`toSingle` (`main.go:599`) strips a trailing `ies` → `y`, else a trailing `s`. Applied to array item names,
that yields exported types:

| Source     | Generated type | Should be     |
| ---------- | -------------- | ------------- |
| `access`   | `AccesOut`     | `AccessOut`   |
| `addresses`| `AddresseOut`  | `AddressOut`  |

`AccesOut` is the element type of `OrganizationGovernanceAccessListOut.Access` — it is in the public API
surface of the client. Any word ending in `s` that is not a plural hits this (`status`, `analysis`, `dns`,
`class` would all mangle if they appeared as array item names).

**Fix:** an exception list in `toSingle` for known non-plurals, or drop the depluralization and disambiguate
by parent name instead. Note this is a breaking rename for anyone referencing `AccesOut` today, so it wants
a deliberate release rather than a drive-by change.

## B8 — `IsAlreadyExists` matches on an English substring — Low

```go
return errors.As(err, &e) && strings.Contains(e.Message, "already exists") && e.Status == http.StatusConflict
```

Both conditions must hold. A 409 phrased any other way — "duplicate name", "conflicts with an existing
service", a localized or reworded message — returns `false`, and callers relying on this for create-or-update
flows take the wrong branch. This is a wire-format coupling to prose that no test pins and no contract
guarantees.

Compare `IsNotFound`, which checks status only and is therefore robust. If 409 is unambiguous for this API,
`IsAlreadyExists` should check status only too; if it is not, the substring list needs to be broader and
documented.

---

## Checked and not a bug

Recorded so nobody re-derives them:

- **All-zero request bodies are not dropped.** `client.go:182` skips the body when
  `in == nil || isEmpty(in)`, and `isEmpty` returns true for any zero value — which would silently drop a
  legitimate payload like `{"enabled": false, "acls": null}` (disabling OpenSearch ACLs). It does not,
  because **all 264 request bodies are passed by pointer**, and a non-nil pointer is never zero. Verified by
  sending exactly that payload; the body went out intact.

  This is latent rather than safe, though: whether a body is a pointer is decided by
  `withPointer(o, s.required)` in `getType`. A required request-body schema would be emitted **by value**,
  and an all-zero instance of it would then be silently dropped. Worth an assertion in the generator that no
  body parameter is ever emitted by value.

- **Singleflight cannot mix credentials.** The key omits the token, but `singleflight.Group` is a field on
  `aivenClient` and the token is per-client, so two differently-authenticated clients never share a group.

## Suggested order of work

1. **B1** — one parameter and one condition; removes a whole class of latent breakage, and the fix is
   self-evidently correct.
2. **B4** — a `len(b) == 0` guard; turns a confusing decoder error into correct behaviour.
3. **B5** — `context.WithoutCancel`; small, and removes a nondeterministic failure that is hard to debug
   from the outside.
4. **B6** — turn the collision warning into a generation failure; cheap, and prevents a silent data bug.
5. **B3** — needs a policy decision on which methods may be replayed.
6. **B2** — needs a design decision on pagination; the largest piece of work here and the one with the
   biggest correctness payoff.
7. **B7**, **B8** — breaking or semi-breaking; batch into a release where API changes are expected.

Each of 1–4 should land with the regression test that currently proves the bug.
