# The `Scope` seam

The engine never resolves fact or variable references itself. Everything inside
`%{...}` that is a **variable** — `%{certname}`, `%{facts.os.name}`,
`%{trusted.certname}`, `%{scope('x')}` — and every scope key used in a
`mapped_paths` level, is resolved against a caller-supplied **`Scope`**. This is
the composition seam: the engine resolves *data*, the host supplies the *facts*.

```go
type Scope interface {
	Lookup(ref string) (Value, bool)
}
```

`Lookup` receives the **raw reference** used inside `%{...}` or passed to
`scope()` — a bare name (`certname`) or a dotted path (`facts.os.name`,
`trusted.certname`) — and returns the value and whether it is present. The engine
imposes no structure on it: how `facts.` / `trusted.` / `server_facts.` are
organised is entirely the `Scope`'s business.

## The bundled `MapScope`

For tests and simple hosts, `MapScope` implements `Scope` over a nested map:

```go
type MapScope map[string]Value

scope := hiera.MapScope{
	"facts": map[string]any{
		"os":         map[string]any{"family": "Debian", "name": "Ubuntu"},
		"networking": map[string]any{"fqdn": "web01.example.com"},
	},
	"trusted": map[string]any{"certname": "web01.example.com"},
	"datacenters": []any{"dc1", "dc2"},   // usable by a mapped_paths level
}
```

A dotted reference **digs** through nested `map[string]any` and `[]any`
(numeric index) values — so `%{facts.os.family}` reaches `"Debian"` above — using
the same [dig](dig.md) walk the data side uses. A leading `::` (Puppet
top-scope, e.g. `%{::certname}`) is ignored. A reference that doesn't resolve
returns *not present*, and the engine substitutes the empty string.

## Implementing your own

Any type with a `Lookup(ref string) (any, bool)` method is a `Scope`. This is how
the ecosystem composes:

### go-facter

[go-facter](https://github.com/go-facter) — a pure-Go facts engine — implements
`Scope` over **live, discovered facts**. Instead of a static map, its `Lookup`
computes `facts.os.family`, `facts.networking.*`, `facts.kernel`, and so on from
the running system, so the same `hiera.yaml` resolves against real machine facts.

### go-ruby-hiera

[go-ruby-hiera](https://github.com/go-ruby-hiera) — the Ruby binding — implements
`Scope` over the **Puppet variable scope**, mapping `$facts`, `$trusted`,
`$server_facts` and top-scope variables from the compiler's binding into
`Lookup`. It wraps this engine to present Ruby's `Hiera` API and Puppet's
`lookup()` function, wiring automatic data binding on top. The deterministic data
resolution stays here; the Ruby surface lives there.

```go
// A minimal custom Scope backed by whatever the host already has.
type hostScope struct{ vars map[string]any }

func (s hostScope) Lookup(ref string) (any, bool) {
	// resolve ref (bare or dotted) against the host's variable graph
	return hiera.MapScope(s.vars).Lookup(ref) // e.g. delegate to the dig helper
}

h, _ := hiera.Load("hiera.yaml", hostScope{vars: myVars})
```

Because the dependency runs **one way** — the host injects a `Scope`, the engine
never reaches back — the engine has no hard dependency on any fact source and
stays a standalone, reusable module.
