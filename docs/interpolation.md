# Interpolation

Every string in the data — values **and** the `path` / `glob` / `mapped_paths`
templates in `hiera.yaml` — may contain `%{...}` tokens. The engine resolves
them recursively: nested strings inside arrays and hashes are interpolated too,
so a whole structured value is expanded before it is merged.

```yaml
# data resolved against scope {facts:{os:{family:"Debian"}}, trusted:{certname:"web01"}}
motd: "Welcome to %{trusted.certname} (%{facts.os.family})"
# -> "Welcome to web01 (Debian)"
```

## The `%{...}` grammar

Between `%{` and the first `}` is an **expression**, with surrounding whitespace
trimmed. It is one of:

| Form | Resolves to |
| --- | --- |
| `%{name}` | A **bare variable** — `name` looked up in the [`Scope`](scope.md). |
| `%{facts.os.name}` | A **dotted path** — the same lookup; the `Scope` digs through nested maps/arrays. `facts.` / `trusted.` are just conventional roots. |
| `%{scope('x')}` | The scope variable `x` (equivalent to `%{x}`). |
| `%{lookup('key')}` | A **recursive Hiera lookup** of `key`. |
| `%{hiera('key')}` | Alias of `lookup` — the legacy Hiera-3 spelling. |
| `%{alias('key')}` | A **whole-value, type-preserving** lookup — see below. |
| `%{literal('%')}` | The literal argument, unchanged — the way to emit a bare `%`. |

An unterminated `%{` (no closing `}`) is an error. A function call must be
`name('arg')` or `name("arg")` (single or double quotes); any other expression
containing `(` is *"malformed"*. An **unknown** function name is an error.

### Bare variables and dotted paths

A bare expression is passed straight to `Scope.Lookup`. With the bundled
[`MapScope`](scope.md), a dotted reference digs through nested `map[string]any`
and `[]any` (numeric index) values, and a leading `::` (Puppet top-scope) is
ignored. A reference that resolves to nothing substitutes the **empty string**
(missing values render empty, matching Hiera).

### `scope()`, `lookup()` / `hiera()`

`scope('x')` is the explicit form of a variable reference. `lookup('key')` and
its alias `hiera('key')` perform a full recursive lookup of another Hiera key and
splice its (stringified) value into the surrounding string.

### `alias()`

`alias('key')` replaces the **entire** value with the looked-up value, preserving
its **type** — so a string value can alias an array or a hash, not just a
scalar's text:

```yaml
# common.yaml
base_packages: ["nginx", "curl"]
web_packages:  "%{alias('base_packages')}"   # -> the ARRAY ["nginx","curl"], not a string
```

Because it returns a whole value, `alias` must be the **complete** string:
`"%{alias('x')} tail"` (any surrounding text, or a second token) is an error
(*"alias interpolation … must be the entire value"*).

### `literal()`

`literal('x')` emits its argument verbatim, without further interpolation — the
standard way to produce a literal percent sign: `%{literal('%')}`.

## Rendering of substituted values

When a token is spliced into a surrounding string, its value is rendered:

- `nil` (missing) → empty string;
- `true` / `false` → `"true"` / `"false"`;
- a string → itself;
- anything else → its default Go rendering (e.g. `42`).

`alias` is the exception: it does not render, it **returns the value itself**.

## Interpolation in paths

The `path` / `glob` / `mapped_paths` templates are interpolated in a restricted
**path mode**: only **scope** references (bare vars and `scope()`) are allowed.
The recursive `lookup()` / `hiera()` / `alias()` functions are **refused** in a
data-source path (*"… is not allowed in a data-source path"*), since a file
location must not depend on data still being resolved.

## Loop detection

Recursive interpolation (`lookup` / `hiera` / `alias`, and the `mapped_paths`
expansion) tracks the stack of keys being resolved. If a key is re-entered while
already on the stack, the engine stops with *"interpolation/lookup loop detected
for key …"* rather than recursing forever.

```yaml
a: "%{lookup('b')}"
b: "%{lookup('a')}"     # looking up 'a' errors: loop a -> b -> a
```
