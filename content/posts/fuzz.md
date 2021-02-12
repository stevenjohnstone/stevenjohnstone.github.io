---
title: "Go-Fuzz"
date: 2021-01-10T20:55:53Z
categories:
    - fuzzing
---

I've found plenty of "crashers" using [go-fuzz](https://github.com/dvyukov/go-fuzz). If you browse the [trophy cabinet](https://github.com/dvyukov/go-fuzz#trophies), you'll notice that I added a few. With Golang being a memory-safe language, the impact
of these bugs is reduced to denial of service vectors. I'd like to point out here that fuzzing can be used for so much more than shaking out off-by-one
errors: if we know invariants or properties that our function outputs must satisfy, then we can build fuzz tests to check for violations.

## [Captain Hindsight](https://southpark.fandom.com/wiki/Captain_Hindsight)

There are examples of how [fuzzing](https://google.github.io/clusterfuzz/setting-up-fuzzing/heartbleed-example/), [static analysis](https://blog.trailofbits.com/2014/04/27/using-static-analysis-and-clang-to-find-heartbleed/) and even [symbolic execution](https://blog.trailofbits.com/2018/04/04/vulnerability-modeling-with-binary-ninja/) _could have been_ used to find the famous [heartbleed](https://en.wikipedia.org/wiki/Heartbleed) bug.
It's all lovely stuff but it's easy to take a known bug and show how your favourite toys can perform. Hindsight is a wonderful thing.

I'm going to do something similar here and show how, with the benefit of hindsight, fuzzing could
have found a security issue. Where I hope I depart from other examples of this kind of potentially annoying retrospect is in showing that there are plenty of places where
fuzzing can be applied in golang without doing major code surgery, learning a fancy new tool or taking a deep dive into the science of SAT solvers etc.
At the end of this ramble, I give my hot-take on why fuzzing isn't more pervasive in golang code.

In this [issue with Sourcegraph](https://github.com/sourcegraph/sourcegraph/security/advisories/GHSA-mx43-r985-5h4m),
the function *SafeRedirectURL* was found to have a corner case which could lead to an [open-redirect](https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html).

I think this is a great example of where fuzzing is straight-forward to apply

* *SafeRedirectURL* is a pure function (not necessary but makes life easier)
* it takes only one unstructured argument
* the security requirements for the function can be expressed concisely in code

Let's consider the last point. *SafeRedirectURL* should only return relative URLs. A necessary condition for this is that the string returned by SafeRedirectURL, when parsed into a URL, has no host component. In golang,

```golang
u, err := url.Parse(SafeRedirectURL(urlString))
if err != nil {
    panic(err) // should this ever happen?
}
if u.Host != "" {
    panic("oops, something has went wrong")
}
```

Concretely, here's how you could fuzz SafeRedirectURL (from before the fix):

```golang
package sourcegraph

import (
    "net/url"
    "strings"
)

func SafeRedirectURL(urlStr string) string {
    u, err := url.Parse(urlStr)
    if err != nil || !strings.HasPrefix(u.Path, "/") {
        return "/"
    }

    // Only take certain known-safe fields.
    u = &url.URL{Path: u.Path, RawQuery: u.RawQuery}
    return u.String()
}

func Fuzz(input []byte) int {
    s := SafeRedirectURL(string(input))
    u, err := url.Parse(s)
    if err != nil {
        return 0
    }

    // if u.Host is non-empty then this is definitely not a relative URL
    if u.Host != "" {
        panic(u.Host)
    }
    return 1
}
```

The fuzzer will input byte slices which are chosen to increase code coverage. When inputs are found which violate our condition on *u.Host*, *Fuzz* will panic
and the fuzzer will  register this as a _crasher_.

I tried this to see how long it'd take to find a bug and it was pretty much
(from a human point of view) instantaneous. I have a mid-range Ryzen laptop
which is nothing special by modern standards.

```shell
$ go-fuzz
2021/01/10 19:13:21 workers: 8, corpus: 155 (0s ago), crashers: 65, restarts: 1/0, execs: 0 (0/sec), cover: 0, uptime: 3s
```

That is, 65 inputs were found in 3 seconds which violate our expectation of *SafeRedirectURL*. What did these look like? Here's a representative example input

```shell
//0//k0
```

with output from *SafeRedirectURL* of

```shell
//k0
```

Sadly, that's exactly what we don't want: an _absolute URL_. The URL may look a bit odd without a scheme but browsers will accept it and work out the scheme from the
context. There's clearly a problem with inputs whose path starts have a double forward slash. That's reassuring as that's exactly what the [fix](https://github.com/sourcegraph/sourcegraph/pull/10167) addresses.
Even better, when the fix is applied to the fuzzing code, a few minutes of fuzzing results in no issues being found.

What should stand out here is that you don't need to know that much about URL parsing to find issues with the fuzzer. You need to know the basic standard library functions but the fuzzer can find the corner cases by trying to cover as much of that library code as possible. You can explore the coverage [here](/pages/coverage.html).

## A Few Minutes of Fuzzing

That's a bit of a throwaway line. How long should we fuzz for exactly? That's not really known. This puts this kind of testing outside of what usually runs
in Continuous Integration as tests which are quick and deterministic are required.

## Barriers to Adoption of Fuzzing

* Friction: It's not yet a [first class language feature of Go](https://go.googlesource.com/proposal/+/master/design/draft-fuzzing.md#:~:text=The%20goal%20is%20to%20make%20fuzzing%20a%20first-class,the%20basis%20for%20experiments%20with%20different%20mutation%20engines.)
* CI: it can take an unknown time to get good coverage
* Misconceptions: it appears to be about finding illegal array accesses etc
