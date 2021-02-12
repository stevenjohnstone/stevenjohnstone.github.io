---
title: "Gofuzz (Without the Hyphen)"
date: 2021-02-11T16:55:20Z
draft: false
categories:
    - fuzzing
---

## Hyphens

There are two different go fuzzing "things": [go-fuzz](https://github.com/dvyukov/go-fuzz) and [gofuzz](https://github.com/google/gofuzz). Both are authored by Google employees. The naming is unfortunate but once you know the difference it
clears up a lot of confusion.

gofuzz (note the lack of hyphen) creates Go objects from random
sources. go-fuzz does coverage guided mutation fuzzing with
some [extra tricks](https://github.com/stevenjohnstone/toughfuzzer). gofuzz provides functionality to use go-fuzz
as its source of "randomness" (are you keeping up with the hyphens?) e.g. from the gofuzz [README](https://github.com/google/gofuzz/blob/master/README.md)

```go
// +build gofuzz
package mypackage

import fuzz "github.com/google/gofuzz"

func Fuzz(data []byte) int {
    var i int
    fuzz.NewFromGoFuzz(data).Fuzz(&i)
    MyFunc(i)
    return 0
}
```

In theory, this allows us to fuzz functions which take objects
other than byte slices (or objects like strings which are
trivially derived from byte slices) as arguments.

The claim by the gofuzz project is that you can do "easier go-fuzzing" in this way. I don't dispute the easy part but is it effective? That's what this post and some follow ups will test.

## Why Though?

Fuzzing functions which take objects as input is an important
problem to solve. A good solution would, I believe, reduce the barrier to
entry for fuzzing significantly. A good solution must work well
with go-fuzz and not break its mutational guided fuzzing, sonar or versifier. That is, it shouldn't break the magic of go-fuzz.

## Does the README example work?

Let's fill out the example with  "MyFunc":

```golang
// +build gofuzz
package mypackage

import fuzz "github.com/google/gofuzz"

func MyFunc(i int) {
    if i == 1337 {
        panic("found correct i")
    }
}

func Fuzz(data []byte) int {
    var i int
    fuzz.NewFromGoFuzz(data).Fuzz(&i)
    MyFunc(i)
    return 0
}
```

Here, *MyFunc* will crash if the input is 1337. It's very artificial but represents a key feature of software: some paths
are taken based on constants in the program and those paths
may be buggy.

What are the chances of finding this by feeding *MyFunc* with random *int* values? I'd expect to have to try 2^63 values before finding the magic 1337. Luckily, go-fuzz isn't feeding
just random values. It knows about program constants and it gets
updated when comparisons are made by its *sonar*. So go-fuzz has
knowledge of 1337 and will use it as input with various
encodings:

* big-endian
* little-endian
* decimal encoded ASCII
* hex encoded ASCII

That said, will go-fuzz be able to discover the path to a panic
with *NewFromGoFuzz* doing its thing? I gave it a try, running it for about an hour seeing no crashes (so the value 1337 wasn't found). This doesn't sound like a long time until you consider that this fuzzer without NewFromGoFuzz in the way finds the value nearly instantly:

```golang
// +build gofuzz

package mypackage

import "encoding/binary"

func MyFunc(i int) {
    if i == 1337 {
        panic("found correct i")
    }
}

func Fuzz(data []byte) int {
    i := binary.BigEndian.Uint64(data)
    MyFunc(int(i))
    return 0
}
```

## What Goes Wrong?

Let's breakdown how an *int* is constructed from go-fuzz (remember the hyphen) by gofuzz.

In *NewFromGoFuzz*, [eight bytes](https://github.com/google/gofuzz/blob/master/bytesource/bytesource.go#L43) are
taken from *data* (with zeroes if *data* is too short).
These eight bytes are used to seed a random number generator.
The random number generator is used once the data provided by
go-fuzz is used up.
Any remaining bytes are used to construct the *int* but once
those run out, the random number generator is used.

As go-fuzz tries the various encodings of 1337, it's likely that
the inputs it passes to *Fuzz* will be less then eight bytes and lost as
inputs to a random number generator. Eventually, go-fuzz will
move away from trying these encodings and their mutations.

## Can We Fix It? Yes We Can

Kinda. There's going to me more broken stuff later.

Applying this patch:

```diff
diff --git a/bytesource/bytesource.go b/bytesource/bytesource.go
index 5bb3659..efd6a65 100644
--- a/bytesource/bytesource.go
+++ b/bytesource/bytesource.go
@@ -40,7 +40,9 @@ func New(input []byte) *ByteSource {
                fallback: rand.NewSource(0),
        }
        if len(input) > 0 {
-               s.fallback = rand.NewSource(int64(s.consumeUint64()))
+               seed := make([]byte, 8)
+               copy(seed, input)
+               s.fallback = rand.NewSource(int64(binary.BigEndian.Uint64(seed)))
        }
        return s
 }
 ```

to gofuzz allows go-fuzz to find a crasher (1337) for *MyFunc*
almost instantaneously.

The improvement here is to keep the bytes used to seed the
random number generator for use as material to build objects.

## Conclusions

Gofuzz's (no hyphen) *NewFromGoFuzz* can break some of the clever
tricks go-fuzz has to increase code coverage. In particular, it
appears to break literal and sonar feedback.

A simple patch can be applied to work around this but, as we'll
see in later posts, there are more issues to contend with.
