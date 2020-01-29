---
title: "Golang: Panicking Safely"
tags:
    - panic
    - bugs
categories:
    - golang
date: 2020-01-28T19:16:22Z
---

Golang code can panic, causing availability issues, maybe even a denial of service vector.
Compared to C and C++, it's pretty innocuous. Still, denial of service is a serious issue: why not just use ```recover``` in all goroutines
and keep the show on the road?

# An Example
Consider the following code:

```golang
// buggy reads data from input 16 bytes at a time, munges it and then writes it to output
func buggy(input, output string) error {
    in, err := os.Open(input)
    if err != nil {
        return err
    }
    defer in.Close() // okay to ignore errors

    out, err := os.Create(output)
    if err != nil {
        return err
    }

    var buf [16]byte

    for {
        _, err := io.ReadFull(in, buf[:])
        if err == io.EOF {
            break
        }
        if err != nil {
            _ = out.Close()
            return err
        }

        bufferMunger(buf[:])

        if _, err := out.Write(buf[:]); err != nil {
            _ = out.Close()
            return err
        }
    }

    // Avoids a common bug where files which are written to
    // are closed without checking the error...with async IO
    // the Close() could result in data actually making it to
    // disk which can error
    return out.Close()
}
```

Now, suppose that this is called in a program which *recovers all panics and keeps rolling on*. If ```bufferMunger``` panics,
 then (at least temporarily), ```buggy``` will leak a file descriptor for
```out```, as explained in an [earlier post]({{<ref "file-descriptor-go.md">}}). With a little imagination, these scenarious can be extended to leaks of
* database connections
* database transactions
* customer reservations

etc.

It could be argued that it'd be better if the program failed hard rather than continuing on in some broken way. Nevertheless, a large chunk of Go code in the wild does keep rolling on
because it _runs inside an HTTP handler_. It may surprise some that **net/http will recover from panics in HTTP handlers**.

# HTTP Handlers

When writing HTTP handlers, we have a couple of strategies for avoiding strange _resource exhaustion_ bugs caused by
panics:

1. make sure all code is "panic safe" (can be recovered)
2. configure http middleware to turn a panic into an exit

## Panic Safety

Let's make ```buggy``` "panic safe". For the sake of argument, suppose that ```bufferMunger``` doesn't
leak resources when it panics. If it can, it needs modifications too.

### Using defer

```golang {hl_lines=["15-20", 44]}
// buggy reads data from input 16 bytes at a time, munges it and then writes it to output
func buggy(input, output string) error {
    in, err := os.Open(input)
    if err != nil {
        return err
    }
    defer in.Close() // okay to ignore errors


    out, err := os.Create(output)
    if err != nil {
        return err
    }

    outClosed := false
    defer func() {
        if !outClosed {
            _ = out.Close()
        }
    }()

    var buf [16]byte

    for {
        _, err := io.ReadFull(in, buf[:])
        if err == io.EOF {
            break
        }
        if err != nil {
            return err
        }

        bufferMunger(buf[:])

        if _, err := out.Write(buf[:]); err != nil {
            return err
        }
    }

    // Avoids a common bug where files which are written to
    // are closed without checking the error...with async IO
    // the Close() could result in data actually making it to
    // disk which can error
    outClosed = true
    return out.Close()
}
```

Here, ```out``` will be closed in a defered function. Note how I avoid closing ```out``` twice.

### Using recover

A simple mechanism would be to wrap bufferMunger so it returns an error rather than causing a panic. For example,

```golang
func bufferMungerWrapper(buf []byte) (err error) {
    defer func() {
        if r := recover(); r != nil {
            switch x := r.(type) {
            case string:
                err = errors.New(x)
            case error:
                err = x
            default:
                err = errors.New("unknown panic")
            }
        }
    }()

    bufferMunger(buf)
    return nil
}
```


## Middleware

If a quick and obvious failure after a panic is preferred to a potentially hard-to-debug resource exhaustion (hint: it is), then wrapping a handler which causes
the program to exit is a good strategy:

```golang
func NewPanicHandler(h http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if r := recover(); r != nil {
				// log.Fatalf causes the program to exit
				log.Fatalf("%s", debug.Stack())
			}
		}()
		h.ServeHTTP(w, r)
	})
}
```

# Conclusion

Using ```recover``` isn't a silver bullet. My 2¢: let panic cause a program exit. Fix logic bugs. Find them with fuzzing.

