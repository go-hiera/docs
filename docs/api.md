# Go API

The public API lives at the module root, `github.com/go-hiera/hiera`. It is
**Hiera-shaped but Go-idiomatic**: explicit `error` returns, value types, no
global state, and a caller-injected [`Scope`](scope.md).

```sh
go get github.com/go-hiera/hiera
```

## Constructing an engine

### `Load`

```go
func Load(configPath string, scope Scope) (*Hiera, error)
```

Reads and parses the `hiera.yaml` at `configPath` and returns an engine
resolving interpolation against `scope`. The configuration's **directory** is the
base for relative datadirs. Returns the read or parse error if either fails.

### `New`

```go
func New(cfg *Config, scope Scope) *Hiera
```

Returns an engine for an already-parsed `cfg`. The built-in `yaml_data` and
`json_data` backends are registered; add more with
[`RegisterDataHash`](#registerdatahash). **Panics** if `cfg` or `scope` is `nil`.

### `ParseConfig`

```go
func ParseConfig(data []byte, dir string) (*Config, error)
```

Parses `hiera.yaml` bytes; `dir` is the directory containing it (the base for
relative datadirs). Use it with `New` when you already hold the config bytes.

## Looking up a key

### `(*Hiera) Lookup`

```go
func (h *Hiera) Lookup(key string, opts *Options) (Value, bool, error)
```

Resolves `key` and returns its value, whether it was found, and any error.
`key` may be [dotted](dig.md) (e.g. `"profile.ntp.servers.0"`) to dig into
structured data after the root key is looked up and merged. A `nil` `opts` is
equivalent to the zero `Options`.

```go
v, found, err := h.Lookup("ntp::servers", nil)
```

### `Options`

```go
type Options struct {
	Merge             *MergeStrategy  // override the merge behaviour for this call
	Default           Value           // default_value returned when the key is absent
	HasDefault        bool            // enable Default (so a nil default is distinguishable)
	DefaultValuesHash map[string]any  // per-root-key defaults, used when HasDefault is false
}
```

- **`Merge`** — when non-nil, overrides the merge behaviour, beating any
  `lookup_options` entry (see [Merge behaviours](merge-behaviors.md)).
- **`Default`** / **`HasDefault`** — a `default_value` returned when the key is
  found nowhere; it is **interpolated** against the scope. `HasDefault` lets a
  `nil` default be intentional.
- **`DefaultValuesHash`** — consulted by the **root key** when the key is absent
  and `HasDefault` is false; returned as-is.

```go
strat := &hiera.MergeStrategy{Kind: hiera.MergeDeep, KnockoutPrefix: "--"}
v, found, err := h.Lookup("profile::settings", &hiera.Options{
	Merge:      strat,
	Default:    map[string]any{},
	HasDefault: true,
})
```

## Merge types

### `MergeKind`

```go
type MergeKind int

const (
	MergeFirst  MergeKind = iota // highest-priority value (default)
	MergeUnique                  // deep-flatten arrays/scalars, deduplicated
	MergeHash                    // shallow hash merge, higher priority wins
	MergeDeep                    // recursive hash merge, higher priority wins
)
```

### `MergeStrategy`

```go
type MergeStrategy struct {
	Kind             MergeKind
	KnockoutPrefix   string // deep: a prefixed array element removes the plain one
	MergeHashArrays  bool   // deep: merge arrays element-wise instead of unioning
	SortMergedArrays bool   // unique/deep: sort merged arrays
}
```

See [Merge behaviours](merge-behaviors.md) for what each field does.

## Configuration types

### `Config`

```go
type Config struct {
	Version   int
	Defaults  Defaults
	Hierarchy []HierarchyEntry
	// unexported: the directory the config was loaded from
}

func (c *Config) Dir() string          // directory the config was loaded from
func (h *Hiera) Config() *Config        // the engine's parsed configuration
```

### `Defaults`

```go
type Defaults struct {
	DataDir     string
	BackendKind string          // "", "data_hash", "data_dig", "lookup_key"
	BackendName string
	Options     map[string]any
}
```

### `HierarchyEntry`

```go
type HierarchyEntry struct {
	Name        string
	Paths       []string // from path / paths, relative to the datadir
	Globs       []string // from glob / globs, relative to the datadir
	MappedPaths []string // exactly 3 elements: [scope_key, loop_var, template]
	DataDir     string
	BackendKind string   // "", "data_hash", "data_dig", "lookup_key"
	BackendName string
	Options     map[string]any
}
```

See [Configuration](configuration.md) for the YAML that populates these.

## Backends

### `RegisterDataHash`

```go
func (h *Hiera) RegisterDataHash(
	name string,
	fn func(data []byte, path string) (map[string]any, error),
)
```

Registers (or replaces) a `data_hash` backend under `name`, so a hierarchy level
may select it via `data_hash: name`. See [Backends](backends.md).

## The scope

### `Scope` and `MapScope`

```go
type Scope interface {
	Lookup(ref string) (Value, bool)
}

type MapScope map[string]Value
```

`Scope` is the pluggable variable provider `%{...}` resolves against; `MapScope`
is a bundled implementation over a nested map. See [The Scope seam](scope.md).

### `Value`

```go
type Value = any
```

A documentary alias — the public API uses `any`. Decoded data uses the fixed
[value model](index.md#value-model): `map[string]any`, `[]any`, `string`,
`int64`, `float64`, `bool`, `nil`.
