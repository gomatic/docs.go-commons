---
title: go-commons
---

`go-commons` is the gomatic ecosystem's grab-bag of **tiny, dependency-light Go helpers** — the small utilities that keep reappearing across gomatic projects, collected once so each one lives in a single, tested place. The root [`commons`](https://pkg.go.dev/github.com/gomatic/go-commons) package holds generic primitives (`IsNil`, `Must`, `Cast`); focused subpackages cover hashing ([`digest`](https://pkg.go.dev/github.com/gomatic/go-commons/digest)), value/pointer conversion ([`ptr`](https://pkg.go.dev/github.com/gomatic/go-commons/ptr)), prefix grouping ([`prefix`](https://pkg.go.dev/github.com/gomatic/go-commons/prefix)), and string casing/joining ([`str`](https://pkg.go.dev/github.com/gomatic/go-commons/str)). Everything depends on the standard library and a couple of well-known casing helpers — nothing heavyweight.

- **Source:** [`gomatic/go-commons`](https://github.com/gomatic/go-commons)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-commons](https://pkg.go.dev/github.com/gomatic/go-commons)

## Install

```sh
go get github.com/gomatic/go-commons
```

## Packages

| Package | Import path | What's in it |
|---------|-------------|--------------|
| [`commons`](https://pkg.go.dev/github.com/gomatic/go-commons) | `github.com/gomatic/go-commons` | `IsNil` (catches typed nils), `Must`, `Cast` |
| [`digest`](https://pkg.go.dev/github.com/gomatic/go-commons/digest) | `github.com/gomatic/go-commons/digest` | `SHA256`, `SHA512` — hex hashes of a byte slice |
| [`ptr`](https://pkg.go.dev/github.com/gomatic/go-commons/ptr) | `github.com/gomatic/go-commons/ptr` | `To` (value → pointer), `Val` (pointer → value, nil-safe) |
| [`prefix`](https://pkg.go.dev/github.com/gomatic/go-commons/prefix) | `github.com/gomatic/go-commons/prefix` | `CommonPrefixes` — group strings by shared leading prefix |
| [`str`](https://pkg.go.dev/github.com/gomatic/go-commons/str) | `github.com/gomatic/go-commons/str` | case conversion, slice-to-string joiners, JSON `Marshal` |

## Usage

### `commons` — nil checks and generic unwrapping

The root package is imported as `commons`. `IsNil` catches the typed nils that hide inside a non-nil interface (a `nil` slice, map, channel, func, or pointer), which a plain `i == nil` misses. `Must` unwraps a `(value, err)` pair and panics on error — for init-time code where failure genuinely shouldn't happen. `Cast` is a panic-free comma-ok type assertion.

```go
package main

import (
	"fmt"
	"net/url"

	"github.com/gomatic/go-commons"
)

func main() {
	var p *int
	fmt.Println(p == nil)            // true
	fmt.Println(commons.IsNil(p))    // true
	fmt.Println(commons.IsNil(any(p))) // true — typed nil inside an interface

	// Must: panic instead of threading an error through init.
	u := commons.Must(url.Parse("https://gomatic.github.io"))
	fmt.Println(u.Host) // gomatic.github.io

	// Cast: comma-ok type assertion, no panic on a miss.
	var v any = "hello"
	if s, ok := commons.Cast[string](v); ok {
		fmt.Println(s) // hello
	}
}
```

### `digest` — hex SHA-2 hashes

`SHA256` and `SHA512` return the lowercase hex digest of a byte slice — the same bytes you'd get from `sha256sum` / `sha512sum`.

```go
package main

import (
	"fmt"

	"github.com/gomatic/go-commons/digest"
)

func main() {
	fmt.Println(digest.SHA256([]byte("hello")))
	// 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
	fmt.Println(digest.SHA512([]byte("hello")))
	// 9b71d224bd62f3785d96d46ad3ea3d73319bfbc2890caadae2dff72519673ca72323c3d99ba5c11d7c7acc6e14b8c5da0c4663475c2e5c3adef46f73bcdec043
}
```

### `ptr` — value ⇄ pointer

`To` returns a pointer to its argument, which inlines a literal where an API wants a `*T`. `Val` dereferences a pointer, returning the zero value of `T` when the pointer is `nil`.

```go
package main

import (
	"fmt"

	"github.com/gomatic/go-commons/ptr"
)

func main() {
	p := ptr.To("x") // *string pointing at "x", from a literal
	fmt.Println(*p)  // x

	var nilp *int
	fmt.Println(ptr.Val(p))    // x
	fmt.Println(ptr.Val(nilp)) // 0 — nil-safe, yields the zero value
}
```

### `prefix` — group strings by shared prefix

`CommonPrefixes` splits each input string into its run of shared leading prefixes, peeling them off in a single pass. Inputs must not contain the NUL byte (`"\000"`), which the algorithm borrows internally as a separator.

```go
package main

import (
	"fmt"

	"github.com/gomatic/go-commons/prefix"
)

func main() {
	out := prefix.CommonPrefixes([]string{"ABC", "ABCD", "ABCDE"})
	fmt.Println(out)
	// [[ABC] [ABC D] [ABC D E]]
}
```

### `str` — casing, joining, and JSON

`str` collects string helpers. Case conversion (`LowerCamelCase`, `UpperCamelCase`, `Title`, `SnakeCase`, `SnakeCaseSplit`) keeps known acronyms (`CIDR`, `SAN`) whole instead of shattering them into single letters. The joiners turn slices of things into a single string: `Stringer` collects each element's `String()` (skipping nils), `Joiner` is `Stringer` + `strings.Join`, `Join` maps each element through a `MapFunc` and concatenates, and `Lister` retypes a `[]string` into a slice of a `~string` newtype. `Marshal` renders any value as indented JSON, falling back to a `%#v` dump if it won't marshal — so it's safe to drop straight into a log line.

```go
package main

import (
	"fmt"

	"github.com/gomatic/go-commons/str"
)

func main() {
	fmt.Println(str.UpperCamelCase("foo_bar")) // FooBar
	fmt.Println(str.LowerCamelCase("FooBar"))  // fooBar
	fmt.Println(str.SnakeCase("FooCIDRs"))     // foo_cidrs — acronym kept whole

	// Join: map each element through a function, then concatenate.
	nums := str.Join(", ", func(n int) string {
		return fmt.Sprintf("#%d", n)
	}, 1, 2, 3)
	fmt.Println(nums) // #1, #2, #3

	// Marshal: indented JSON, log-safe.
	fmt.Println(str.Marshal(map[string]int{"a": 1}))
	// {
	//   "a": 1
	// }
}
```

## Design

- **Tiny and dependency-light** — the root, `digest`, `ptr`, and `prefix` packages depend only on the standard library; `str` adds two well-known casing helpers ([`stoewer/go-strcase`](https://github.com/stoewer/go-strcase) and `golang.org/x/text`).
- **Generics where they pay off** — `Must`, `Cast`, `ptr.To`, `ptr.Val`, `str.Lister`, `str.Stringer`, `str.Joiner`, and `str.Join` are generic so callers keep their concrete types without assertions or wrappers.
- **One helper, one home** — each utility lives in exactly one tested place, so the same `IsNil` / casing / hashing behavior is shared (not re-implemented) across gomatic projects.
- **Library, not a CLI** — the repo carries a `library_marker` build-tag file that is never compiled; it exists only so gomatic tooling recognizes the repo as a library.

## Who uses it

`go-commons` is the shared utility floor for gomatic Go projects — pulled in wherever the same nil check, pointer helper, hash, or string casing is needed, instead of being copied around. See the other [`gomatic/go-*`](https://github.com/orgs/gomatic/repositories?q=go-) libraries.
