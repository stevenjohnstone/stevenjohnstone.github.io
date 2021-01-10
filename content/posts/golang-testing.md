---
title: "Golang Testing"
date: 2021-01-06T14:03:52Z
---

My (very opinionated) checklist/cheatsheet for Golang testing. Nothing particularly original here.

## Tactics
### Tactic: Testable Example

Start with a [testable example](https://blog.golang.org/examples), if you can. This is invaluable documentation and will tell you if the
code has a usable interface, has too-many moving parts etc.

### Tactic: Table-Driven Tests

[Dave Chaney](https://dave.cheney.net/2019/05/07/prefer-table-driven-tests) makes the case for table-driven tests.

### Tactic: Network Testing

Avoid hard coding listener port numbers. Overcome the problems of port and address allocations by letting the operating system choose the port.
Every (Go) programmer should know the following trick:

```golang
// allocate a loopback listener with a port chosen by the operating system
func newLoopbackListener() (net.Listener, error) {
    return net.Listen("127.0.0.1:0")
}
```

Here, a net.Listener is created without hard coding the port. This allows us to run many tests in parallel without port conflicts.
If you run out of ports, then you could always move to the other loopback addresses: 127.0.0.n for n=2,3,...

Say we write a server implementing the following minimal interface:
```golang
type Server interface {
    // Start the server processing requests
    Start() error
    // Stop the server and return
    Stop()
}
```

Then we can imagine testing its client as follows:

```golang
func TestServer(t *testing.T) {
    lis, err := newLoopbackListener()
    if err != nil {
        t.Fatal(err)
    }
    server, err := NewServer(l)
    if err != nil {
        t.Fatal(err)
    }

    go server.Start()
    defer server.Stop()

    client, err := NewClient(lis.Addr())
    if err != nil {
        t.Fatal(err)
    }

    // do testing stuff
}
```

This removes the need to mock listeners, client connections etc.

### Tactic: Golden Files
Hashicorp uses this [trick](https://github.com/hashicorp/consul/blob/87f6617eecd23a64add1e79eb3cd8dc3da9e649e/agent/xds/golden_test.go#L36):

```golang
// golden reads and optionally writes the expected data to the golden file,
// returning the contents as a string.
func golden(t *testing.T, name, subname, got string) string {
	t.Helper()

	suffix := ".golden"
	if subname != "" {
		suffix = fmt.Sprintf(".%s.golden", subname)
	}

	golden := filepath.Join("testdata", name+suffix)
	if *update && got != "" {
		err := ioutil.WriteFile(golden, []byte(got), 0644)
		require.NoError(t, err)
	}

	expected, err := ioutil.ReadFile(golden)
	require.NoError(t, err)

	return string(expected)
}
```

Golden test data can be used for outputs that you don't know in precise detail up front but can inspect for correctness once you have them. You run your tests
with the "update" flag first, then inspect the golden file to see if it is correct and then finally commit the golden file to your repository.
Subsequent tests will use this data as the *gold standard*.

### Tactic: Metamorphic Testing

Wayne Hillel give an excellent overview of metamorphic testing [here](https://www.hillelwayne.com/post/metamorphic-testing/). If *f* is the function
under test, *x* is some golden test data and *T* is some transformation of the data, then we should test that the relationship between *f(x)* and *f(Tx)*
is as we expect. The skill here is thinking up interesting transforms *T*. A good place to start is with those for which *f* is *invariant*. That is, *T* such that *f(Tx) = f(x)*. Other important cases are idempotent functions and round-trip guarantees e.g. *Decode(Encode(x)) == x*

### Tactic: Fuzz

Fuzzing is really a subject unto itself. However, it's not hard to add the green shoots of fuzz testing. A good start is to simply fuzz a function and see if it crashes e.g. https://github.com/trustelem/zxcvbn/commit/c2422dd63cdfab8ebc99174ce967424ab9088624.

Fuzzing can be combined with metamorphic testing to cover more code.

## Use Abstraction but Not Too Much

Try to decouple code from OS resources (sockets, file handles etc) by judicious use of abstraction. For example, this

```golang
func Foo(filename string) (*Result, error) {
    f, err := os.Open(filename)
    if err != nil {
        return nil, err
    }
    defer f.Close()
    // Do amazing customer pleasing stuff here
}
```

can be refactored to

```golang
func FooReader(r io.Reader) (*Result, error) {
    // Do amazing customer pleasing stuff here
}
func Foo(filename string) (*Result, error) {
    f, err := os.Open(filename)
    if err != nil {
        return nil, err
    }
    defer f.Close()
    return FooReader(f)
}
```

Now we can test the function _FooReader_ without creating files:

```golang
func TestFooReader(t *testing.T) {
    testReader := strings.NewReader("some test data")
    res, err := FooReader(testReader)
    // do some checking here based on expectations
}
```

Mocking resources such as databases can be very useful but care should be taken not to try to mock too much. There's a point where you're better off
doing integration tests or end-to-end tests.

Where mocking shines is testing error handling paths. There's some [evidence](https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-yuan.pdf) to suggest that many of the critical failures observed in distributed systems occur in (presumably) untested error handling paths. In our example, we could [mock
the filesystem](https://talks.golang.org/2012/10things.slide#8):

```golang
var fs fileSystem = osFS{}

type fileSystem interface {
    Open(name string) (file, error)
    Stat(name string) (os.FileInfo, error)
}

type file interface {
    io.Closer
    io.Reader
    io.ReaderAt
    io.Seeker
    Stat() (os.FileInfo, error)
}

// osFS implements fileSystem using the local disk.
type osFS struct{}

func (osFS) Open(name string) (file, error)        { return os.Open(name) }
func (osFS) Stat(name string) (os.FileInfo, error) { return os.Stat(name) }

func foo(filename string, fs filesystem) (*Result, error) {
    f, err := fs.Open(filename)
    if err != nil {
        return nil, err
    }
    // Do amazing customer pleasing stuff here
}

func Foo(filename string) (*Result, error) {
    return foo(filename, fs)
}
```

When testing, we can create a *filesystem* implementation which injects errors. Something along the lines of this:

```golang
type faultFS struct{}
type faultFile struct{}

func (faultFile) Close() error {
    return errors.New("close failed")
}

// Exercise for the interested reader to provide Read, ReadAt, Write, Close, Seeker and Stat for faultFile


func (faultFS) Open(string) (file, error) { return faultFile{} }
func (faultFS) Stat(string) (os.FileInfo, error) {return nil, errors.New("not supported")}

func TestFoo(t *testing.T) {
    res, err := foo("foo", faultFS{})
    // check the results
}
```

IMHO, mocking doesn't work well when imitating complex systems like databases. I prefer to use container technology to spin up an actual instance of the database and test against that. If you need quite sophisticated fault injection, then it'd pay to write a library for this. How much effort you put
into this is a value judgement.

## Integration Tests vs Unit Tests

Instead of mocking a complex sub-system, consider making an integration test which uses an instance of the real service in a container. For example, starting
a docker container from golang is pretty [straight-forward](https://golang.testcontainers.org/). Such tests can be resource intensive so can be hidden behind [build tags](https://golang.org/pkg/go/build/#hdr-Build_Constraints) or [skipped](https://golang.org/pkg/testing/#hdr-Skipping) when the short flag is used.

The challenge here is in avoiding flaky tests caused by having to wait for the service to come online (port open, accepting requests etc). That said, if errors returned by the code don't allow for graceful retries, the client fails to timeout in a reasonable way etc, then that's a worthwhile finding in itself.
