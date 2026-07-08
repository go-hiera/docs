# Merge behaviours

When a key appears in more than one level of the hierarchy, the **merge
behaviour** decides how the per-level values combine. The engine gathers the
key's [interpolated](interpolation.md) values in **priority order** (highest
first) and merges them.

## The four strategies

| Strategy | Behaviour |
| --- | --- |
| `first` | Return the **highest-priority** value found. The default; gathering stops at the first match. |
| `unique` | **Deep-flatten** every found array (and scalar) into a single **deduplicated** array. |
| `hash` | **Shallow-merge** every found hash; a key set by a higher-priority level wins. |
| `deep` | **Recursively merge** every found hash; higher priority wins conflicts, with array-union and knockout options. |

```yaml
# common.yaml
sysctl::values:
  net.ipv4.ip_forward: 0
  vm.swappiness: 60

# nodes/web01.yaml   (higher priority)
sysctl::values:
  vm.swappiness: 10
```

A `hash` (or `deep`) merge of `sysctl::values` yields `ip_forward: 0` **and**
`swappiness: 10` — the higher-priority `swappiness` winning, `ip_forward`
inherited from common.

### `first`

Returns `values[0]`, the highest-priority match. No type constraint — any value
type is fine.

### `unique`

Flattens arrays recursively and deduplicates scalars (type-qualified, so integer
`1` and string `"1"` stay distinct), returning one array. It is **array-only**:
a hash value in the set is an error (*"unique merge cannot merge hash values"*).
With `sort_merged_arrays` the result is sorted.

### `hash`

Shallow-merges the hashes: for each key, the **highest-priority** level that
defines it wins; other keys are unioned in. Every value must be a hash, or the
merge errors (*"hash merge requires hash values …"*).

### `deep`

Recursively merges hashes key-by-key, folding from the lowest-priority value up
so **higher priority wins** conflicts. Nested hashes merge recursively; scalars
and type mismatches take the higher-priority value. Arrays merge per the options
below.

## Deep-merge options

Set on the merge spec (in `lookup_options`) or on a `MergeStrategy` passed per
call.

### `merge_hash_arrays`

Controls how two arrays combine during a deep merge.

- **Default (off):** arrays are **unioned** — the higher-priority elements first,
  then the lower-priority ones, with duplicates removed (and knockout applied).
- **On:** arrays merge **element-wise by index** — position 0 with position 0,
  and so on; where both sides have an element it is merged recursively, otherwise
  the present element is kept.

### `knockout_prefix`

In a deep merge, a prefixed element **removes** the plain one it names:

- In an array, an element `"<prefix>x"` (e.g. `"--ntp1"` with prefix `--`) drops
  both the marker itself **and** any plain `"x"` (`"ntp1"`) from the union.
- In a hash, a value **equal to** the prefix deletes its key from the result.

```yaml
lookup_options:
  ntp::servers:
    merge:
      strategy: deep
      knockout_prefix: "--"

# a higher-priority level removes the inherited "0.pool.ntp.org":
# ntp::servers: ["--0.pool.ntp.org", "time.example.com"]
```

### `sort_merged_arrays`

Sorts the merged array (by rendered form, deterministically) for `unique` and
`deep` merges.

## `lookup_options`

`lookup_options` is a reserved key in your data that selects the **per-key merge
strategy** so callers don't have to. It is itself **hash-merged** across the
hierarchy (higher-priority levels winning per key), then consulted for the key
being looked up.

```yaml
# common.yaml
lookup_options:
  # exact key
  ntp::servers:
    merge: unique

  # a regex (must start with ^) matched against the lookup key
  "^profile::.*::settings$":
    merge:
      strategy: deep
      knockout_prefix: "--"
      merge_hash_arrays: true
      sort_merged_arrays: true
```

A merge spec may be:

- a **bare strategy string** — `unique`, `hash`, `deep`, `first`;
- a `{ "merge": <spec> }` wrapper (the `<spec>` is any of these forms); or
- a `{ "strategy": <name>, ... }` mapping carrying `knockout_prefix`,
  `merge_hash_arrays` and `sort_merged_arrays`.

### Selection order

For a given key the strategy is chosen as:

1. an **explicit** `Options.Merge` passed to the call (overrides everything);
2. an **exact** `lookup_options` entry for the key;
3. a **`^regex`** `lookup_options` entry whose pattern matches the key;
4. otherwise **`first`**.

An exact match always beats a regex match.

## Defaults for a missing key

When a key is found in **no** level, `Lookup` can still return a value via
[`Options`](api.md):

- **`Default`** (enabled by `HasDefault`) — a `default_value`, which is itself
  **interpolated** against the scope before being returned. `HasDefault`
  distinguishes an intentional `nil` default from "no default".
- **`DefaultValuesHash`** — a per-key defaults map, consulted by the **root key**
  when `HasDefault` is false; returned as-is (not interpolated).

`Default` takes precedence over `DefaultValuesHash`. If neither applies, the
lookup reports *not found* (`found == false`).
