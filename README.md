# Jev Go SDK

Go client for TypeSafe AI's System One API. This is an independently audited source fork of [`Stumble/jev-go`](https://github.com/Stumble/jev-go/tree/a475dc925ba68602be93f4478e1381cf5ec27ee4). It is not an official TypeSafe AI release. The `github.com/RyanJarv/jev` module path is reserved for publication; the repository has not been created yet.

The SDK supports Noul, Choice, and Score questions, batched requests, model listing, bounded HTTP responses, context cancellation, retries, and structured API errors. It also supports the Vercel AI Gateway evaluation dialect. It has no third-party Go dependencies and requires Go 1.22 or newer.

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"

    jev "github.com/RyanJarv/jev"
)

func main() {
    client, err := jev.NewClient(jev.Config{APIKey: os.Getenv("TYPESAFE_API_KEY")})
    if err != nil { log.Fatal(err) }

    response, err := client.SystemOne(context.Background(), jev.Request{
        State: "Help! My payout has failed for three days.",
        Questions: map[string]jev.Question{
            "urgent": jev.Noul("Does this message convey urgency?"),
            "department": jev.Choice("Which team should handle this?", map[string]any{
                "billing": "Payments and payouts",
                "technical": "Bugs and integrations",
            }),
            "severity": jev.Score("How severe is the problem?", []string{
                "Minor inconvenience", "Service disruption", "Critical outage",
            }),
        },
    })
    if err != nil { log.Fatal(err) }
    fmt.Println(response.Answers["urgent"].Noul)
    fmt.Println(response.Answers["department"].Choice)
    fmt.Println(response.Answers["severity"].Score)
}
```

`Config` accepts an explicit key by default. To read `TYPESAFE_API_KEY`, `TYPESAFE_BASE_URL`, and `TYPESAFE_DEFAULT_MODEL` as fallbacks, set `ReadFromEnvironment: true`. A caller supplied `http.Client` is copied; redirects are blocked to prevent credential forwarding. Base URLs must use HTTPS, except for local loopback HTTP during testing. Keep keys on the server side.

The default timeout is 10 seconds per attempt and the default retry policy makes up to two additional attempts for connection failures, timeouts, 408, 429, and 5xx responses. Set a total deadline with `context.WithTimeout` when required. See [AUDIT.md](AUDIT.md) for selection evidence, the security review, and verification limits.

## Verification

```sh
go test ./...
go test -race ./...
go vet ./...
```

The tests use local HTTP servers and do not require an API key. A separate live smoke check using an ignored local token successfully listed models and evaluated a batch containing Noul, Choice, and Score questions on 2026-09-26. The token is not part of the repository.
