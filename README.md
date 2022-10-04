# fluentassert/verify

> Extensible, type-safe, fluent assertion Go library.

[![Go Reference](https://pkg.go.dev/badge/github.com/fluentassert/verify.svg)](https://pkg.go.dev/github.com/fluentassert/verify)
[![Keep a Changelog](https://img.shields.io/badge/changelog-Keep%20a%20Changelog-%23E05735)](CHANGELOG.md)
[![GitHub Release](https://img.shields.io/github/v/release/fluentassert/verify)](https://github.com/fluentassert/verify/releases)
[![go.mod](https://img.shields.io/github/go-mod/go-version/fluentassert/verify)](go.mod)
[![LICENSE](https://img.shields.io/github/license/fluentassert/verify)](LICENSE)

[![Build Status](https://img.shields.io/github/workflow/status/fluentassert/verify/build)](https://github.com/fluentassert/verify/actions?query=workflow%3Abuild+branch%3Amain)
[![Go Report Card](https://goreportcard.com/badge/github.com/fluentassert/verify)](https://goreportcard.com/report/github.com/fluentassert/verify)
[![Codecov](https://codecov.io/gh/fluentassert/verify/branch/main/graph/badge.svg)](https://codecov.io/gh/fluentassert/verify)

Please ⭐ `Star` this repository if you find it valuable and worth maintaining.

## Description

The fluent API makes the assertion code easier
to read and write ([more](https://dave.cheney.net/2019/09/24/be-wary-of-functions-which-take-several-parameters-of-the-same-type)).

The generics (type parameters) make the usage type-safe.

The library is [extensible](#extensibility) by design.

### Quick start

```go
package test

import (
	"testing"

	"github.com/fluentassert/verify"
)

func Foo() (string, error) {
	return "wrong", nil
}

func TestFoo(t *testing.T) {
	got, err := Foo()

	verify.NoError(err).Require(t)           // Require(f) uses t.Fatal(f), stops execution if fails
	verify.String(got).Equal("ok").Assert(t) // Assert(f) uses t.Error(f), continues execution if fails
}
```

```sh
$ go test
--- FAIL: TestFoo (0.00s)
    basic_test.go:17:
        the objects are not equal
        got: "wrong"
        want: "ok"
```

### Deep equality

```go
package test

import (
	"testing"

	"github.com/fluentassert/verify"
)

type A struct {
	Str   string
	Bool  bool
	Slice []int
}

func TestDeepEqual(t *testing.T) {
	got := A{Str: "wrong", Slice: []int{1, 4}}

	verify.Obj(got).DeepEqual(
		A{Str: "string", Bool: true, Slice: []int{1, 2}},
	).Assert(t)
}
```

```sh
$ go test
--- FAIL: TestDeepEqual (0.00s)
    deepeq_test.go:20:
        mismatch (-want +got):
          test.A{
        -       Str:  "string",
        +       Str:  "wrong",
        -       Bool: true,
        +       Bool: false,
                Slice: []int{
                        1,
        -               2,
        +               4,
                },
          }
```

### Collection unordered equality

```go
package test

import (
	"testing"

	"github.com/fluentassert/verify"
)

func TestSlice(t *testing.T) {
	got := []int { 3, 1, 2 }

	verify.Slice(got).Equivalent([]int { 2, 3, 4 }).Assert(t)
}
```

```sh
$ go test
TODO
```

### Asynchronous (periodic polling)

```go
package test

import (
	"net/http"
	"testing"
	"time"

	"github.com/fluentassert/verify"
)

func TestAsync(t *testing.T) {
	verify.Periodic(10*time.Second, time.Second, func() verify.FailureMessage {
		client := http.Client{Timeout: time.Second}
		resp, err := client.Get("http://not-existing:1234")
		if err != nil {
			return verify.NoError(err)
		}
		return verify.Number(resp.StatusCode).Lesser(300)
	}).Eventually().Assert(t)
}
```

```sh
$ go test
--- FAIL: TestAsync (10.00s)
    async_test.go:19:
        timeout
        function always failed
        last failure message:
        non-nil error:
        Get "http://not-existing:1234": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
```

## Extensibility

### Custom fluent assertions

You can take advantage of the `verify.FailureMessage` and `verify.Fluent*` types
to create your own fluent assertions for a given type.

For reference, take a look at the implementation
of existing fluent assertions in this repository
(for example [comparable.go](verify/comparable.go)).

### Custom assertion function

For simple cases, you can simply prepare a function that returns `FailureMessage`.

```go
package test

import (
	"testing"

	"github.com/fluentassert/verify"
)

type A struct {
	Str string
	Ok  bool
}

func TestCustom(t *testing.T) {
	got := A{Str: "something was wrong"}

	verifyA(got).Assert(t)
}

func verifyA(got A) verify.FailureMessage {
	var msg verify.FailureMessage
	msg.Merge("got.String assertion:",
		verify.String(got.Str).Contain("ok"),
	)
	msg.Merge("got.Ok assertion:",
		verify.True(got.Ok),
	)
	return msg
}
```

```sh
$ go test
--- FAIL: TestCustom (0.00s)
    custom_test.go:17:
        got.String assertion:
        the value does not contain the substring
        got: "something was wrong"
        substr: "ok"

        got.Ok assertion:
        the value is false
```

### Custom predicates

For the most basic scenarios, you can also use one of the
`Check`, `Should`, `ShouldNot` assertions.

```go
package test

import (
	"strings"
	"testing"

	"github.com/fluentassert/verify"
)

func TestShould(t *testing.T) {
	got := "wrong"

	chars := "abc"
	verify.Obj(got).Should(func(got string) bool {
		return strings.ContainsAny(got, chars)
	}).Assertf(t, "does not contain any of: %s", chars)
}
```

```sh
$ go test
--- FAIL: TestShould (0.00s)
    should_test.go:16: does not contain any of: abc
        object does not meet the predicate criteria
        got: "wrong"
```

## Supported Go versions

Minimal supported Go version is 1.18.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) if you want to help.

## License

**fluentassert/verify** is licensed under the terms of [the MIT license](LICENSE).

[`github.com/google/go-cmp`](https://github.com/google/go-cmp)
(license: [BSD-3-Clause](https://pkg.go.dev/github.com/google/go-cmp/cmp?tab=licenses))
is the only [third-party dependency](go.mod).
