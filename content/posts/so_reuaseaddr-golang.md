---
title: "SO_REUASEADDR and Golang"
categories:
    - golang
date: 2017-10-13T13:53:00+01:00
---

# A Flaky Test


I was investigating a test which failed with a very low probability. It boiled down to this:

```golang
package main

import (
	"net"
	"time"
)

func main() {
	//port zero means let the OS choose a free port
	l, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		panic(err)
	}

	addr := l.Addr().String()

	go func() {
		for {
			c, err := l.Accept()
			if err != nil {
				return
			}
			c.Close()
		}
	}()

	time.Sleep(time.Second)
	if err := l.Close(); err != nil {
		panic(err)
	}

	//listen on the same port as before
	l, err = net.Listen("tcp", addr)
	if err != nil {
		panic(err)
	}
}
```

It's pretty standard to stop a goroutine which is blocked in Accept() by calling Close() on the listener. Passes casual inspection, linters don't complain etc. However, if you run the above enough times, you may see something like this:

```
panic: listen tcp 127.0.0.1:37327: bind: address already in use
```

Address already in use? This caused me some confusion as I know that [SO_RESUSEADDR](http://man7.org/linux/man-pages/man7/socket.7.html) is set set by Golang in net.Listen and I've been conditioned to think "SO_RESUSEADDR" when I see EALREADY in a situation like this.

# Root Cause

The problem here is that Close() of a listener doesn't immediately close the underlying file descriptor. What it [does](https://golang.org/src/internal/poll/fd_unix.go#L69) is decrement a reference count and if that reference count is non-zero will exit with a file descriptor still open. Subsequent calls to net.Listen may fail with "bind: address already in use" because the address really is in use. Not waiting on sockets to leave TIME_WAIT, just _open_.

The reference count on the underlying file descriptor may be non-zero because a goroutine is still in the guts of [Accept()](https://golang.org/src/internal/poll/fd_unix.go#L317) where it has locked the file-descriptor. This explains our intermittent failure: we call Close() and then Listen() but there's a goroutine still in Accept() which has prevented the underlying file-descriptor from being closed.

# The fix

...is to ensure that the goroutine is not in Accept() when we try to listen again on the same port. For example, we can close a channel to signal that the goroutine is done:

```golang
package main

import (
	"net"
	"time"
)

func main() {
	//port zero means let the OS choose a free port
	l, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		panic(err)
	}

	addr := l.Addr().String()
	done := make(chan bool)

	go func() {
		for {
			c, err := l.Accept()
			if err != nil {
				close(done)
				return
			}
			c.Close()
		}
	}()

	time.Sleep(time.Second)
	if err := l.Close(); err != nil {
		panic(err)
	}

	//wait for the goroutine to have left Accept()
	<-done

	//listen on the same port as before
	l, err = net.Listen("tcp", addr)
	if err != nil {
		panic(err)
	}
}
```

# Lessons Learned

* when Close() of a listener returns, the underlying file descriptor may still be open
* any method calls for the listener must complete to be sure that the underlying file descriptor is closed






