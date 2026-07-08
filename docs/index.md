# go-hiera documentation

**A pure-Go (CGO-free) reimplementation of Puppet's [Hiera 5](https://www.puppet.com/docs/puppet/latest/hiera.html)
hierarchical data-lookup engine.**

`go-hiera/hiera` loads a Hiera 5 configuration (`hiera.yaml`), walks the
configured hierarchy of data sources, and resolves a key to a typed value —
honouring Hiera's merge behaviours (`first` / `unique` / `hash` / `deep`),
per-key `lookup_options`, dotted-key digging into structured data, and the full
`%{...}` interpolation grammar, with interpolation-loop detection. The module
path is `github.com/go-hiera/hiera`.

The engine is **standalone and reusable**: it has **no hard dependency on any
particular fact source**. The caller injects a [`Scope`](scope.md) (a variable
provider), so a facts engine such as [go-facter](https://github.com/go-facter),
or the Ruby binding [go-ruby-hiera](https://github.com/go-ruby-hiera) that maps
Ruby's `Hiera` API and Puppet's `lookup()` onto it, plugs its own facts in.

!!! success "Status: v0.1 shipped — faithful to Hiera 5"
    `hiera.yaml` v5 loader; `yaml_data` (via the pure-Go
    [go-ruby-yaml](https://github.com/go-ruby-yaml/yaml) Psych backend) and
    `json_data` backends; the four merge behaviours with `knockout_prefix` /
    `merge_hash_arrays` / `sort_merged_arrays`; per-key and `^regex`
    `lookup_options`; dotted-key dig; and the complete `%{...}` grammar
    (`scope` / `lookup` / `hiera` / `alias` / `literal`) with loop detection.
    **100% test coverage** including every error branch, `gofmt` + `go vet`
    clean, green across the six 64-bit Go targets (amd64, arm64, riscv64,
    loong64, ppc64le, s390x).

## What it is — and isn't

Resolving a hierarchy of YAML/JSON data with merge and interpolation is fully
**deterministic** and needs **no Ruby interpreter**, so it lives here as pure Go.
The Ruby-facing `Hiera` / `lookup()` surface and Puppet's automatic-data-binding
wiring stay in the consumer ([go-ruby-hiera](https://github.com/go-ruby-hiera) /
rbgo).

> **This library resolves data; the host presents the Ruby API.**

The engine also does not resolve fact/variable references itself. `%{facts.os.name}`,
`%{trusted.certname}` and `%{scope('x')}` are resolved against the injected
[`Scope`](scope.md) — the seam a facts source plugs into.

## Install

```sh
go get github.com/go-hiera/hiera
```

## Quick taste

```go
package main

import (
	"fmt"
	"log"

	"github.com/go-hiera/hiera"
)

func main() {
	// The caller supplies the variable/fact scope %{...} resolves against.
	scope := hiera.MapScope{
		"facts":   map[string]any{"os": map[string]any{"family": "Debian"}},
		"trusted": map[string]any{"certname": "web01.example.com"},
	}

	h, err := hiera.Load("hiera.yaml", scope)
	if err != nil {
		log.Fatal(err)
	}

	// First-found lookup.
	v, found, err := h.Lookup("ntp::servers", nil)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(v, found)

	// Force a merge strategy for this call.
	deep := &hiera.MergeStrategy{Kind: hiera.MergeDeep, KnockoutPrefix: "--"}
	merged, _, _ := h.Lookup("profile::settings", &hiera.Options{Merge: deep})
	fmt.Println(merged)

	// Dotted key: dig into structured data after the merge.
	port, _, _ := h.Lookup("profile.ntp.port", nil)
	fmt.Println(port)
}
```

## Value model

Data files decode to a small, fixed set of Go types, so a host can map results
onto its own object graph:

| Hiera / YAML | Go |
| --- | --- |
| Hash | `map[string]any` |
| Array | `[]any` |
| String | `string` |
| Integer | `int64` |
| Float | `float64` |
| Boolean | `bool` |
| Null | `nil` |

YAML is decoded through [go-ruby-yaml](https://github.com/go-ruby-yaml/yaml)
(the ecosystem's Ruby-faithful, pure-Go Psych backend, matching the YAML
semantics Puppet relies on); JSON through the standard library.

## Repositories

| Repo | What it is |
| --- | --- |
| [`hiera`](https://github.com/go-hiera/hiera) | the engine — `hiera.yaml` loader, backends, merge, interpolation, the `Load` / `New` / `Lookup` API |
| [`docs`](https://github.com/go-hiera/docs) | this documentation site (MkDocs Material, versioned with mike) |
| [`go-hiera.github.io`](https://github.com/go-hiera/go-hiera.github.io) | the organization landing page (Hugo) |
| [`brand`](https://github.com/go-hiera/brand) | logo and brand assets |

## Principles

- **Pure Go, `CGO_ENABLED=0`** — trivial cross-compilation, a single static
  binary, no C toolchain.
- **Faithful to Hiera 5.** Merge, `lookup_options`, interpolation and dig match
  Hiera's documented semantics so the engine can back Puppet's `lookup()`.
- **Standalone & reusable.** No hard dependency on any fact source; the caller
  injects a `Scope`.
- **100% test coverage** is the target, enforced as a CI gate.

## Where to go next

- [Configuration](configuration.md) — the `hiera.yaml` v5 schema: `version`,
  `defaults`, and the `hierarchy` levels.
- [Backends](backends.md) — `yaml_data`, `json_data`, and `RegisterDataHash`.
- [Merge behaviours](merge-behaviors.md) — `first` / `unique` / `hash` / `deep`
  and `lookup_options`.
- [Interpolation](interpolation.md) — the `%{...}` grammar and its functions.
- [Dotted-key dig](dig.md) — digging into structured data after merge.
- [Go API](api.md) — `Load`, `New`, `Lookup`, `Options`, `MergeStrategy`.
- [The Scope seam](scope.md) — how go-facter and go-ruby-hiera feed facts in.
- [Roadmap](roadmap.md) — what is done and what is next.

Source lives at [github.com/go-hiera/hiera](https://github.com/go-hiera/hiera).
