# Dotted-key dig

A lookup key may be **dotted** to reach *inside* a structured value. The engine
splits the key on `.`: the **first** segment is the **root key** looked up and
merged across the hierarchy as usual; the **remaining** segments then **dig**
into that merged result.

```yaml
# common.yaml
profile::ntp:
  port: 123
  servers:
    - 0.pool.ntp.org
    - 1.pool.ntp.org
```

```go
h.Lookup("profile::ntp", nil)            // the whole hash
h.Lookup("profile::ntp.port", nil)       // 123        (map key)
h.Lookup("profile::ntp.servers", nil)    // the array
h.Lookup("profile::ntp.servers.0", nil)  // "0.pool.ntp.org"  (array index)
```

## How digging works

After the root key is resolved and **merged**, each remaining segment steps one
level down:

- into a **hash** (`map[string]any`) by **key**;
- into an **array** (`[]any`) by **decimal index** (`0`-based; a negative or
  out-of-range index does not resolve).

Digging happens **after** the merge, so it sees the fully-merged structure — a
deep-merged hash assembled from several levels digs exactly as a single-file hash
would.

## When a path does not resolve

If the root key is found but a later segment misses — a key that isn't in the
hash, a non-numeric or out-of-range array index, or a segment that tries to
descend into a scalar — the lookup reports **not found** (`found == false`), the
same as a missing root key. It is not an error.

```go
v, found, _ := h.Lookup("profile::ntp.servers.9", nil)   // v=nil, found=false
v, found, _ := h.Lookup("profile::ntp.port.deeper", nil) // digging into a scalar: found=false
```

## Root keys and separators

The split is purely on `.`, so a Puppet-style key such as `ntp::servers` (no
dots) is a **single root key** — the `::` is part of the name, not a separator.
Only `.` introduces a dig segment.

!!! note "Literal dots in a root key"
    Because the whole key is split on `.`, a root key that itself contains a
    literal dot cannot yet be addressed — the first dot always starts the dig
    path. **Quoted dotted keys** (to escape a dot inside a key name) are on the
    [roadmap](roadmap.md).

The same dotted-dig mechanism powers [`Scope`](scope.md) references
(`%{facts.os.name}`) and `mapped_paths` loop variables — one `digPath` walk over
maps and arrays, used everywhere structured data is addressed by path.
