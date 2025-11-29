# go-utils — Reusable Go Utility Library  
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)  
[![Build Status](https://github.com/azargarov/go-utils/actions/workflows/ci.yml/badge.svg)](https://github.com/azargarov/go-utils/actions/workflows/ci.yml)

A collection of modular, reusable Go packages designed to accelerate development of production-grade Go applications.  
This library encapsulates common patterns for logging, worker-pools, network fetching, rate-limiting, string/auto-string utilities.

## Features

- Modular packages: `backoff`, `wpool`, `zlog`, `httpsrv`, `grlimit`, `autostr`.  
- Cleaner, abstracted code-reuse: Extracted utilities from multiple apps into a shared library.  
- Easy to adopt: Import only the packages you need, no monolithic “all-in-one” binary.

## Installation

```bash
go get github.com/azargarov/go-utils@latest
```
## Usage Examples

### Worker pool (wpool)

``` go
import (
    "context"
    "fmt"
    "time"

    wp "github.com/azargarov/go-utils/workerpool"
)

type JobPayload struct {
    ID int
}

func main() {
    ctx := context.Background()
    pool := wp.NewPool[JobPayload](3, *wp.GetDefaultRP())
    defer pool.Stop()

    n := 5
    done := make(chan struct{}, n)

    for i := 0; i < n; i++ {
        job := wp.Job[JobPayload]{
            Payload: JobPayload{ID: i},
            Ctx:     ctx,
            Fn: func(p JobPayload) error {
                fmt.Printf("Running job #%d\n", p.ID)
                time.Sleep(200 * time.Millisecond)
                if p.ID == 2 {
                    return fmt.Errorf("timeout simulated") // triggers retry
                }
                return nil
            },
            CleanupFunc: func() { done <- struct{}{} },
            Retry: &wp.RetryPolicy{Attempts: 3, Initial: 100 * time.Millisecond, Max: 1 * time.Second},
        }

        if err := pool.Submit(job); err != nil {
            fmt.Println("Submit failed:", err)
            done <- struct{}{}
        }
    }

    // Wait for all jobs to finish
    for i := 0; i < n; i++ {
        <-done
    }

    fmt.Println("All jobs completed")
}


```
### Exponential backoff (backoff)

``` go
import "github.com/azargarov/go-utils/backoff"

err := backoff.Retry(func() error {
    return doNetworkCall()
}, backoff.Config{
    MaxAttempts: 5,
    InitialDelay: 100 * time.Millisecond,
    Factor: 2.0,
})
if err != nil {
    // handle failure
}


```

## Architecture Overview

The project follows a modular layout, enabling clear boundaries and re-use:
``` bash
go-utils/
├── backoff/       # retry/back-off utilities
├── wpool/         # worker-pool abstraction
├── zlog/          # structured logger
├── netfetch/      # network fetch utilities (HTTP, TLS, etc.)
├── grlimit/       # group rate limiting
├── autostr/       # auto / string utilities
└── httpsrv/       # HTTP server helpers
```