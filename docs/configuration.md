# Configuration — `hiera.yaml`

The engine is configured by a **Hiera 5** `hiera.yaml`. `Load` reads it,
`ParseConfig` parses it, and the file's **directory becomes the base** for
resolving relative datadirs. Only **version 5** is accepted.

```yaml
version: 5

defaults:
  datadir: data
  data_hash: yaml_data

hierarchy:
  - name: "Per-node data"
    path: "nodes/%{trusted.certname}.yaml"

  - name: "Per-OS defaults"
    path: "os/%{facts.os.family}.yaml"

  - name: "Common"
    path: "common.yaml"
```

## `version`

```yaml
version: 5
```

**Required**, and **must be `5`**. A missing `version`, a non-integer value, or
any version other than 5 is a parse error (`only version 5 is supported`).

## `defaults`

Config-level defaults inherited by every hierarchy level. Every key is optional;
an unknown key is a parse error.

| Key | Meaning |
| --- | --- |
| `datadir` | Base directory for data files (relative to the `hiera.yaml` directory, or absolute). Defaults to `data` when unset at every level. |
| `data_hash` | The default `data_hash` backend for levels that don't set one (e.g. `yaml_data`). |
| `data_dig` | Reserved: a `data_dig` backend name. |
| `lookup_key` | Reserved: a `lookup_key` backend name. |
| `options` | A mapping of backend options. |

`data_hash`, `data_dig` and `lookup_key` select a **backend function**, and are
mutually exclusive — declaring more than one in `defaults` is an error
(`declares multiple backend functions`).

!!! note "v0.1 implements `data_hash` only"
    `data_dig` and `lookup_key` **parse** but are not yet executable: a lookup
    against a level whose backend kind is `data_dig` or `lookup_key` fails with
    *"backend kind … is not supported (v0.1 implements data_hash only)"*. When no
    backend is set anywhere, the engine falls back to `data_hash: yaml_data`.
    See [Backends](backends.md).

## `hierarchy`

A **list** of levels, consulted **top to bottom** (highest priority first). Each
entry is a mapping; an unknown key, or an entry without a `name`, is a parse
error. Every entry must select data files with **at least one** of `path`,
`paths`, `glob`, `globs` or `mapped_paths`.

### Per-level keys

| Key | Meaning |
| --- | --- |
| `name` | **Required.** A human label for the level. |
| `path` | A single data-file path, relative to the level's datadir. |
| `paths` | A list of paths (a string or list of strings). |
| `glob` | A single shell glob, expanded against the datadir. |
| `globs` | A list of globs. |
| `mapped_paths` | `[scope_key, loop_var, template]` — see below. |
| `datadir` | Overrides the datadir for this level. |
| `data_hash` / `data_dig` / `lookup_key` | Overrides the backend for this level (mutually exclusive). |
| `options` | Per-level backend options mapping. |

`path` and `paths` accumulate, as do `glob` and `globs`; a single entry may mix
them. Each `path` / `glob` / `mapped_paths` template is
[interpolated](interpolation.md) with the scope before use — but only scope
variables are allowed in a path; `lookup()` / `hiera()` / `alias()` are refused
there.

### `datadir` resolution

The datadir for a level is chosen in order: the **entry's** `datadir`, else the
**config default** `datadir`, else the literal `data`. A relative datadir is
resolved against the `hiera.yaml` directory; an absolute datadir is used as-is.

### Paths and globs

```yaml
hierarchy:
  - name: "Nodes"
    paths:
      - "nodes/%{trusted.certname}.yaml"
      - "nodes/%{facts.networking.fqdn}.yaml"

  - name: "Roles"
    glob: "roles/*.yaml"

  - name: "Common"
    path: "common.yaml"
    datadir: shared        # override datadir for this level only
```

A `path` names exactly one candidate file. A `glob` / `globs` entry is expanded
with the OS globber (`filepath.Glob`) against the datadir; every match becomes a
candidate. **Missing files are skipped** (not an error); a candidate that exists
but fails to read or parse *is* an error.

### `mapped_paths`

`mapped_paths` derives one path per element of an array-valued scope variable. It
takes **exactly three** string elements — anything else is a parse error:

```yaml
hierarchy:
  - name: "Per-datacenter"
    mapped_paths:
      - datacenters          # scope_key: an array in the scope
      - dc                   # loop_var: bound to each element
      - "dc/%{dc}.yaml"      # template: interpolated per element
```

The `scope_key` is looked up in the [`Scope`](scope.md); if it is missing or is
not an array, the level contributes **no** paths. Otherwise each element is bound
to `loop_var` (as both `%{dc}` and, for structured elements, `%{dc.sub.key}`)
and the `template` is interpolated to a path under the datadir.

## `default_hierarchy`

Not supported in v0.1: a `hiera.yaml` containing a top-level `default_hierarchy`
key is rejected at parse time. It is on the [roadmap](roadmap.md).
