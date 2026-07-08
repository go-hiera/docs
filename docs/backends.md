# Backends

A **backend** parses the raw bytes of one data file into a hash
(`map[string]any`). A hierarchy level selects a `data_hash` backend by name; the
engine ships two and lets you register more.

## Built-in `data_hash` backends

| Name | Parses | Decoder |
| --- | --- | --- |
| `yaml_data` | YAML data files | [go-ruby-yaml](https://github.com/go-ruby-yaml/yaml) — the Ruby-faithful, pure-Go Psych backend |
| `json_data` | JSON data files | the Go standard library (`encoding/json`) |

Both require the document's **top level to be a mapping** (Hiera data files are
always hashes). An **empty document** decodes to an empty hash; a non-mapping top
level (e.g. a bare list or scalar) is an error. Decoded values are normalised to
the engine's [value model](index.md#value-model) — `map[string]any`, `[]any`,
strings, `int64`, `float64`, `bool`, `nil`.

```yaml
version: 5
defaults:
  datadir: data
hierarchy:
  - name: "YAML common"
    path: "common.yaml"          # yaml_data (the default backend)

  - name: "JSON secrets"
    path: "secrets.json"
    data_hash: json_data         # per-level backend override
```

### Default backend

When no backend is declared at the level **or** in `defaults`, the engine falls
back to `data_hash: yaml_data`. So a minimal `hiera.yaml` with only `path`
entries reads YAML.

## Registering a custom backend

`RegisterDataHash` adds (or replaces) a `data_hash` backend on an engine, so a
level can select it by name. This is the seam future **eyaml** / **hocon**
backends slot into.

```go
h := hiera.New(cfg, scope)

// A backend gets one file's bytes and its path (for error context) and
// returns the parsed top-level hash.
h.RegisterDataHash("toml_data", func(data []byte, path string) (map[string]any, error) {
	var m map[string]any
	if err := toml.Unmarshal(data, &m); err != nil {
		return nil, fmt.Errorf("%s: %w", path, err)
	}
	return m, nil
})
```

```yaml
hierarchy:
  - name: "TOML nodes"
    path: "nodes/%{trusted.certname}.toml"
    data_hash: toml_data
```

The signature is:

```go
func (h *Hiera) RegisterDataHash(
	name string,
	fn func(data []byte, path string) (map[string]any, error),
)
```

- `data` — the raw bytes of one candidate file (only existing files are passed;
  the engine skips missing ones before calling the backend).
- `path` — the file path, for wrapping into error messages.
- The returned hash is the level's contribution, interpolated and merged like any
  other level's data.

Backends registered on an engine created with `New` are added on top of the two
built-ins; registering under an existing name replaces it.

!!! note "Only the `data_hash` kind is executable in v0.1"
    A level whose backend **kind** is `data_dig` or `lookup_key` parses but
    fails at lookup time (*"v0.1 implements data_hash only"*), and an unknown
    `data_hash` name fails with *"unknown data_hash backend …"*. The
    `data_dig` / `lookup_key` execution kinds are on the [roadmap](roadmap.md).
