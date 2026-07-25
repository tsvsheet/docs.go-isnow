---
title: go-isnow
---

**The [isnow](https://github.com/tsvsheet/isnow) date/time pattern language as an importable Go library.** An isnow is one compact expression describing anything from a fixed instant to a complex recurrence, answering a single question: _is it now?_ `go-isnow` parses a pattern into an immutable `Pattern`, tests membership at any instant, and derives past and future occurrences — with no filesystem, network, or clock access of its own.

The library is the one engine behind the [`isnow`](https://github.com/tsvsheet/isnow.go) CLI and the `ISNOW(...)` spreadsheet function in [go-tsvsheet](https://github.com/tsvsheet/go-tsvsheet). Import it to evaluate isnow patterns from your own Go programs, services, or schedulers.

## Install

```sh
go get github.com/tsvsheet/go-isnow
```

API reference: **[pkg.go.dev/github.com/tsvsheet/go-isnow](https://pkg.go.dev/github.com/tsvsheet/go-isnow)**.

## Quickstart

```go
package main

import (
	"fmt"
	"time"

	isnow "github.com/tsvsheet/go-isnow"
)

func main() {
	p, err := isnow.Parse("M,W,F noon")
	if err != nil {
		panic(err) // one of the four sentinels, matchable with errors.Is
	}

	at := time.Date(2026, 7, 13, 12, 0, 0, 0, time.UTC) // a Monday
	fmt.Println(p.Holds(at))                            // true
	fmt.Println(p.Canonical())                          // */*/* Monday,Wednesday,Friday 12:00:00

	next, ok := p.Next(at)
	fmt.Println(ok, next) // true 2026-07-15 12:00:00 +0000 UTC (Wednesday)
}
```

`Parse` recognizes the pattern, resolves symbols (`M`, `noon`, `mwf`) and the shorthand ladder, validates every field domain, and returns an immutable `Pattern`. `Holds` is the defining operation — membership of an instant — and `Next`/`Prev` derive occurrences from it.

## The language

One expression constrains up to seven fields — year, month, day, weekday, hour, minute, second — with a shorthand ladder that infers which fields you meant: `6` is 6 AM daily, `M noon` is Mondays at noon, `12/25 mn` is every Christmas at midnight. Fields take lists (`M,W,F`), spans (`9-17`, `M-F`), steps (`+[15]`, `Th+[2]` for the 2nd Thursday), from-end selection (`-1w`), periodic intervals (`+[90mn]`), bounds (`>=2026`, `<=17:30`), and exclusions (`! 12/25`). The language itself — grammar, semantics, and the cross-implementation conformance corpus — is specified in the [isnow](https://github.com/tsvsheet/isnow) repo (`SPECIFICATION.md`).

## Core API

- **`Parse(src PatternText) (Pattern, error)`** — compile pattern source into an immutable `Pattern`. `PatternText` is the exported source-text type, so callers convert their string input at the call site.
- **`Pattern.Holds(at time.Time) bool`** — the membership test: does the isnow hold at the instant? Evaluated in the instant's own location, truncated to whole seconds.
- **`Pattern.Next(from) / Prev(from) (time.Time, bool)`** — the earliest occurrence strictly after (or latest strictly before) an instant, or false when none exists within the pattern's window or the 100-year search horizon.
- **`Pattern.NextContext(ctx, from) / PrevContext(ctx, from)`** — the same derivations with cancellation, for pathological patterns whose scan would otherwise pin a CPU.
- **`Pattern.Canonical() string`** — the fully-qualified seven-field form (also `String()`); parsing a canonical form is a fixed point.
- **`Pattern.Explain() string`** — a deterministic English description of the pattern.
- **`Is(src PatternText, at time.Time) (bool, error)`** — the one-shot form: Parse then Holds.
- **`Code(err error) string`** — the stable cross-implementation code (`syntax`, `symbol`, `range`, `context`) of a library error.

`Pattern` is immutable and every method is a value receiver: patterns are safe to copy, alias, and share across goroutines.

## Errors

`Parse` rejects a bad pattern with one of four `errs.Const` sentinels shared by every isnow implementation, matchable with `errors.Is`:

- **`ErrSyntax`** — a malformed pattern the grammar rejects.
- **`ErrSymbol`** — an unknown or ambiguous weekday/time/unit name (`xyzzy`, an ambiguous prefix like `T`).
- **`ErrRange`** — a value outside its field's domain (`25:00`).
- **`ErrContext`** — a grammatical but semantically invalid construct (`///`, a weekday bound).

## Design

The public surface is one package — `github.com/tsvsheet/go-isnow` — a thin, documented facade. The recognizer (the grammar repo's ANTLR-generated parser), the ladder, the field compiler, and the derivation engine are all internal; no parser or grammar type escapes into the public API. The library is clock-free by construction — every query takes the instant to evaluate against — so callers inject their own time source and results are deterministic and testable. Behavior is pinned by the cross-implementation conformance corpus in the grammar repo.

## Related

- **[isnow](https://github.com/tsvsheet/isnow)** — the language: grammar, specification, and conformance corpus.
- **[isnow.go](https://github.com/tsvsheet/isnow.go)** — the CLI built on this library.
- **[isnow.js](https://github.com/tsvsheet/isnow.js)** — the JavaScript implementation.
- **[go-tsvsheet](https://github.com/tsvsheet/go-tsvsheet)** — the tsvsheet spreadsheet engine, whose `ISNOW(...)` function this library powers.
