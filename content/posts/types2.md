---
title: "Golang Types and Secrets: Part 2"
date: 2021-01-02T19:37:42Z
---

The [previous post](/posts/types), described how a simple toy authentication system could be improved by using a type definition to redact secrets when they are passed to fmt.Printf, logging functions etc. In this post, hiding the complexity of defending against timing attacks is outlined.

Let's frame the discussion with a simple scenario: you run an online store with user accounts, you store usernames and you want to avoid [user enumeration](https://blog.rapid7.com/2017/06/15/about-user-enumeration/). You may not have user enumeration in your threat-model but if you care about GDPR, you should probably add it. In the [previous post](/posts/types), I outlined how to stop a secret ending up in a log message. Stopping personally identifiable information ending up in a log unintentionally could help avoid a hefty fine. So it makes sense to define a type to carry around usernames so they don't accidentally leak
into log messages and the approach outlined previously is one way to go about it.

Be warned: the code in the [previous post](/posts/types) is vulnerable to timing attacks due to the comparison of the secret against a string using "==". You can google plenty of [references](https://www.cs.rice.edu/~dwallach/pub/crosby-timing2009.pdf) on timing attacks. The thing a developer needs to remember is:

__If you have a secret and need to compare it to
something, you need to use [timing-independent comparison functions](https://golang.org/pkg/crypto/subtle/)__. 

Here's some code which shows how to ensure that
comparisons are done safely for our hypothetical usernames with our previous "log redaction" folded in.

```golang
package main

import (
	"crypto/subtle"
	"fmt"
)

// Secret hides its data from the programmer so that unsafe comparisons are
// forbidden (or would require sneaky use of reflection)
type Secret struct {
	data []byte
}

// String returns something suitable for logs
func (s Secret) String() string {
	return "REDACTED"
}

// Equals is timing safe
func (s Secret) Equals(secret string) bool {
	return 1 == subtle.ConstantTimeCompare(s.data, []byte(secret))
}


// Username is an alias of Secret to avoid user enumeration problems
type Username = Secret

func NewUsername(name string) *Username {
	// would be wise to consider unicode normalization here
	return &Username{ data: []byte(name) }
}

func main() {
	username := NewUsername("stevie")
	fmt.Printf("username: %s, known: %v\n", username, username.Equals("stevie"))
}
```








