# Roadmap

`go-hiera/hiera` is grown **test-first**, each capability covered to 100%
(including every error branch) before the next is added. The deterministic core
of Hiera 5 — hierarchy walking, merge, interpolation and dig — is **shipped**;
the Ruby-facing surface is downstream by design.

## v0.1 — shipped

| Area | What | Status |
| --- | --- | --- |
| `hiera.yaml` v5 loader | `version: 5`, `defaults` (datadir + backend function + options), and a `hierarchy` of levels using `path` / `paths` / `glob` / `globs` / `mapped_paths`, with per-level datadir, backend and options. | **Done** |
| Backends | `yaml_data` (via the pure-Go [go-ruby-yaml](https://github.com/go-ruby-yaml/yaml) Psych backend) and `json_data` (stdlib), plus `RegisterDataHash` for custom `data_hash` backends. | **Done** |
| Merge behaviours | `first`, `unique`, `hash`, `deep` — with `knockout_prefix`, `merge_hash_arrays` and `sort_merged_arrays`. | **Done** |
| `lookup_options` | Per-key and `^regex` merge selection, hash-merged across the hierarchy; overridable per call; `default_value` and `default_values_hash`. | **Done** |
| Interpolation | The full `%{...}` grammar — bare / dotted variables, `scope`, `lookup` / `hiera`, `alias` (whole-value, type-preserving), `literal` — with **interpolation-loop detection**. | **Done** |
| Dotted-key dig | Dig into structured data (map keys + array indices) after the merge. | **Done** |
| Quality | 100% coverage incl. error branches, `gofmt` + `go vet` clean, green across the six 64-bit Go targets (amd64, arm64, riscv64, loong64, ppc64le, s390x). | **Done** |

## v0.2 — planned

| Area | What | Status |
| --- | --- | --- |
| `eyaml` backend | An encrypted-YAML `data_hash` backend registered through the existing `RegisterDataHash` seam. | Planned |
| `hocon` backend | A HOCON `data_hash` backend, same seam. | Planned |
| `default_hierarchy` | Support the top-level `default_hierarchy` key (currently rejected at parse time). | Planned |
| Quoted dotted keys | Escape a literal `.` inside a root key so it isn't treated as a dig separator. | Planned |
| `data_dig` / `lookup_key` | Execute the `data_dig` and `lookup_key` backend **kinds** (parsed today, but only `data_hash` runs in v0.1). | Planned |

## Downstream by design

These stay in the **consumer**, not this module — they need a Ruby interpreter or
the Puppet runtime, which the pure-Go engine deliberately does not embed:

- **The Ruby `Hiera` / `lookup()` surface.** [go-ruby-hiera](https://github.com/go-ruby-hiera)
  maps Ruby's `Hiera` API and Puppet's `lookup()` function onto this engine and
  wires **Puppet automatic data binding**. The engine resolves data; the host
  presents the Ruby API.
- **Fact discovery.** The engine consumes a [`Scope`](scope.md); computing live
  facts is [go-facter](https://github.com/go-facter)'s job.

See [Go API](api.md) for the current surface and [Configuration](configuration.md)
for the `hiera.yaml` schema these stages build on.
