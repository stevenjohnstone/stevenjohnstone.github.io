---
title: "Golang Types and Secrets: Part 1"
date: 2021-01-01T19:36:58Z
---

## A Toy Authentication System

Suspend disbelief and consider the following toy authentication system:

```golang
package main

import (
	"fmt"
)

func allowAccess(secret string) bool {
	return secret == "foo"
}

func main() {
	secret := "foo"
	fmt.Printf("Access: %v, Secret: %s", allowAccess(secret), secret)
}
```

Try this at the Golang [playground](https://play.golang.org/p/zYFWdgh-5vF).

## What's wrong with this code?

In my work auditing Golang codebases, I often see secrets being passed around as strings or byte slices. While these are reasonable representations of the
actual data, it'd be better to use the type system to represent the *concept of a secret* and leverage type *operations* to make code safer.

Consider the output of the above program:

```shell
Access: true, Secret: foo
```

The secret "foo" is leaked. Generally speaking, secrets should not end up in logs but if this code were running in a container the secret would, most likely,  end up in the docker logs.

Let's use a type definition to make the situation a little better:

```golang
package main

import (
	"fmt"
)

type Secret string

func (s Secret) String() string {
	return "REDACTED"
}

func allowAccess(secret Secret) bool {
	return secret == "foo"
}

func main() {
	var secret Secret = "foo"
	fmt.Printf("Access: %v, Secret: %s", allowAccess(secret), secret)
}
```

The type *Secret* has the underlying type *string* with all of string's useful operations. Crucially, when a Secret is passed to fmt.Printf the method *String()* is called to get a string representation and this is simply the string "REDACTED". Running [this code at the playground](https://play.golang.org/p/Z92vfBVOTiB) gives

```shell
Access: true, Secret: REDACTED
```

What's been gained here?

1. The code is now clearer about what is a secret and what is just a plain old string
2. It's hard to accidentally send the secret to program output, logs etc.




