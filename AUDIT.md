# Source selection and security audit

Reviewed 2026-09-26. The HTTP contract was checked against TypeSafe's [API reference](https://docs.typesafe.ai/api), [official JavaScript SDK](https://github.com/typesafe-ai/typesafe-sdk-js/tree/66880cc), and [official Python SDK](https://github.com/typesafe-ai/typesafe-sdk-python/tree/0ffd094). Repository and maintainer metadata was read with `gh api`; source was cloned at the commits below. The selected maintainer's GitHub account dates to 2013 and has 65 public repositories; that is a limited continuity signal, not proof of safe code or future maintenance. `mattn` has a much larger established profile, but the reviewed SDK has the transport risks noted below. Stars are context, not evidence of code safety. Every candidate repository was created in September 2026, so none has a long maintenance record.

| Candidate | Reviewed commit | Go minimum | Dependencies | Evidence and decision |
| --- | --- | --- | --- | --- |
| [Stumble/jev-go](https://github.com/Stumble/jev-go) | `a475dc925ba68602be93f4478e1381cf5ec27ee4` | 1.22 | none | Selected. Covers direct API and optional Gateway, validates answers, bounds response bodies, permits HTTPS or loopback HTTP, and blocks redirects even with a supplied client. Contract tests passed. Created Sep 17; 5 stars, no forks at review time. |
| [mattn/go-jev](https://github.com/mattn/go-jev) | `85f5994` | 1.27 | none | Small API and 30 stars, but arbitrary endpoint URLs and unbounded response reads make it a weaker security baseline for a credentialed SDK. |
| [HomayoonAlimohammadi/jev-sdk-go](https://github.com/HomayoonAlimohammadi/jev-sdk-go) | `4ec00b2` | 1.24 | none | Extensive tests and features, but accepts plaintext HTTP for remote hosts; a supplied HTTP client retains its redirect policy, which can forward credentials. Larger surface to audit. |
| [kazz187/jev-sdk-go](https://github.com/kazz187/jev-sdk-go) | `30a619c` | 1.27 | none | Typed Go 1.27 API and passing tests, but permits remote plaintext HTTP and uses caller/default redirect behavior. |

## Scope and findings

The copied files are `client.go`, `types.go`, `retry.go`, `errors.go`, and `vercel.go`, plus their tests and MIT license. The upstream CLI, agent skills, and workflow files were excluded; they are unnecessary for a library and were not installed or executed. The module import path changed to `github.com/typesafe-ai/typesafe-sdk-go` provisionally. The source remains attributable to Stumble under MIT.

- **No malicious behavior found in the reviewed production files.** Imports are from the Go standard library. The code does not execute shell commands, load plugins, use `unsafe`, start background workers, or send telemetry. Its only HTTP destinations are the configured TypeSafe or Vercel endpoint; environment reads occur only with explicit opt-in.
- **Credential exposure controls:** The constructor rejects non-HTTPS remote URLs, URL credentials, query strings, fragments, and newline-bearing keys. It copies the supplied `http.Client` and refuses redirects. Authorization and protocol headers are managed by the SDK. These controls are covered by local tests.
- **Protocol checks:** Direct requests use `POST /v1/systemone`; model listing uses `GET /v1/models`. Noul, Choice, Score, usage, errors, request IDs, retries, and Gateway conversion have local contract tests. Response bodies are capped at 4 MiB. The SDK rejects malformed or mismatched answers before returning them.
- **Residual risks:** This is a new upstream project with no established maintenance history. An audit of this snapshot cannot guarantee future upstream commits. Gateway behavior was verified against local fixtures, not a live Gateway account. The SDK's strict answer validation may reject a future API response shape until updated.

## Verification performed

`go test ./...`, `go test -race ./...`, and `go vet ./...` passed for this fork using Go 1.27.1. The full `go test ./...` suite also passed under Go 1.22.12 in a local Linux container with networking disabled; only source and tests were streamed into it, with no API token mounted. The tests cover request serialization, all three answer kinds, response validation, model listing, retries, cancellation, errors, configuration, concurrency, redirect blocking, and response size limits. A separate attempt to run Go 1.22.12's test binary on this Mac failed in the macOS loader (`missing LC_UUID load command`) before tests could start.

With a locally supplied, ignored token, `GET /v1/models` returned two models and a live batched `POST /v1/systemone` returned valid Noul, Choice, and Score answers from `jev-1.13.0`, with a request ID. The small smoke request exercised the direct API only. This is not a security certification or exhaustive production test.

Publication is pending because the current GitHub CLI token cannot create a repository under `RyanJarv`. Pin this audited commit in consumers until subsequent versions are reviewed.
