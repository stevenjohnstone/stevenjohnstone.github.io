---
title: "Leaking File Descriptors in Go"
categories:
    - golang
    - bugs
date: 2017-10-13T11:39:00+01:00
---

# Golang, File Descriptors and Finalizers

If you [Open()](https://golang.org/pkg/os/#Open) a file in Golang and it goes out of scope, then it'll be closed the next time the garbage collector runs as there's a  cleanup _finalizer_ set when Open() is called.

You can see this in action in the following program:

```golang
package main

import (
	"fmt"
	"os"
	"runtime"
)

func allocate() int {
	//use this file as our test subject
	_, sourceFile, _, ok := runtime.Caller(1)
	if !ok {
		panic("source file not found")
	}
	counter := 0
	var files []*os.File
	for {
		f, err := os.Open(sourceFile)
		if err != nil {
			fmt.Printf("successful opens= %d\n", counter)
			return counter
		}
		counter++
		files = append(files, f)
	}
}

func main() {
    allocate()
    runtime.GC()
    allocate()
}

```

On my system, the output is

```
successful opens = 1020
successful opens = 1020
```

The first time allocate() is called, it manages to open the file 1020 times before returning. The call to [runtime.GC()](https://golang.org/pkg/runtime/#GC) causes the garbage collector to run which closes the open files allowing the next allocate() to open the file 1020 times. What happens if we don't call the garbage collector?

We modify main:

```golang
func main() {
    allocate()
    allocate()
}
```

The result is now:

```
successful opens = 1020
successful opens = 0
```

The garbage collector hasn't run by the time the second allocate() call happens so the available file descriptors for the process are exhausted. When would the garbage collector run?

```golang
func main() {
	allocate()
	for i := 0; ; i++ {
		if allocate() > 0 {
			fmt.Printf("garbage collection ran after %d iterations", i)
			return
		}
	}
}

```

With main as above, it takes ~30k iterations of allocate for the garbage collector to run on my system. This isn't too surprising. Exhaustion of file descriptors will not result in enough memory being used to trigger a garbage collection. The garbage collector is oblivious to the lack of system resources other than memory.

# Don't Rely on the Garbage Collector

...for anything other than collecting memory resources. Tying other resource types to garbage collection is a bad idea and could lead to resource exhaustion. After all, you aren't guaranteed to ever have the garbage collector run for your program. Close files when you're finished with them; defer is your friend.



