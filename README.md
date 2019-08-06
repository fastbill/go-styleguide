# Go Styleguide <!-- omit in toc --> 

This serves as a supplement to
[Effective Go](https://golang.org/doc/effective_go.html), based on years of
experience and inspiration/ideas from conference talks.

## Table of contents <!-- omit in toc --> 

- [Add context to errors](#add-context-to-errors)
- [Consistent error and log messages](#consistent-error-and-log-messages)
- [Dependency management](#dependency-management)
	- [Use dep](#use-dep)
	- [Use Semantic Versioning](#use-semantic-versioning)
	- [Avoid unnessary version lockdown](#avoid-unnessary-version-lockdown)
- [Structured logging](#structured-logging)
- [Avoid global variables](#avoid-global-variables)
- [Testing](#testing)
	- [Use testify as assertion libary](#use-testify-as-assertion-libary)
	- [Use sub-tests to structure functional tests](#use-sub-tests-to-structure-functional-tests)
	- [Use table driven tests](#use-table-driven-tests)
	- [Avoid unnecessary mocks](#avoid-unnecessary-mocks)
	- [Avoid DeepEqual](#avoid-deepequal)
	- [Avoid testing unexported funcs](#avoid-testing-unexported-funcs)
	- [Add examples to your test files to demonstrate usage](#add-examples-to-your-test-files-to-demonstrate-usage)
- [Use linters](#use-linters)
	- [Properly refactor when fixing complexity warnings](#properly-refactor-when-fixing-complexity-warnings)
- [Use meaningful variable names](#use-meaningful-variable-names)
- [Favour pure functions](#favour-pure-functions)
- [Keep interfaces small](#keep-interfaces-small)
- [Don't under-package](#dont-under-package)
- [Handle signals](#handle-signals)
- [Divide imports](#divide-imports)
- [Avoid naked return](#avoid-naked-return)
- [Avoid empty interface](#avoid-empty-interface)
- [Order functions public-private and top-down](#order-functions-public-private-and-top-down)
- [Avoid helper/util](#avoid-helperutil)
- [Structs](#structs)
	- [Use named structs](#use-named-structs)
	- [Avoid new keyword](#avoid-new-keyword)
	- [Anonymous structs are ok for JSON parsing](#anonymous-structs-are-ok-for-json-parsing)
- [Packages/SDKs](#packagessdks)
	- [Return the error](#return-the-error)
	- [Return a struct on setup](#return-a-struct-on-setup)
	- [Provide an interface and a mock for optional use](#provide-an-interface-and-a-mock-for-optional-use)
	- [Extend interface when embedding structs](#extend-interface-when-embedding-structs)
- [Consistent header naming](#consistent-header-naming)

## Add context to errors

**Don't:**
```go
file, err := os.Open("foo.txt")
if err != nil {
	return err
}
```

Using the approach above can lead to unclear error messages because of missing
context.

**Do:**
```go
import "github.com/pkg/errors" // for example

// ...

file, err := os.Open("foo.txt")
if err != nil {
	return errors.Wrap(err, "open foo.txt failed")
}
```

Wrapping errors with a custom message provides context as it gets propagated up the stack.
This is not always necessary, use your judgement at which level in the code it makes most sense to add more context. 
If you're unsure if the context of a returned error is at all times sufficient, wrap it.
Make sure the root error is still accessible somehow for type checking.

## Consistent error and log messages
Error messages should start with a lowercase letter and should not end with a `.`. See [Wiki page on Errors](https://github.com/golang/go/wiki/Errors) for reference. For consistency the same logic should be applied for log messages.

**Don't:**
```go
logger.Print("Something went wrong.")
ReadFailError := errors.New("Could not read file")
```

**Do:**
```go
logger.Print("something went wrong")
ErrReadFailed := errors.New("could not read file")
```


## Dependency management

### Use dep
Use [dep](https://github.com/golang/dep), since it's production ready and will
soon become part of the toolchain.
– [Sam Boyer at GopherCon 2017](https://youtu.be/5LtMb090AZI?t=27m57s)

### Use Semantic Versioning
Since `dep` can handle versions, tag your packages using
[Semantic Versioning](http://semver.org).  
The git tag for your go package should have the format `v<major>.<minor>.<patch>`, e.g., `v1.0.1`.

### Avoid unnessary version lockdown
If possible only lock the major version in the `Gopkg.toml` file.  

**Don't:**
```
[[constraint]]
  name = "github.com/stretchr/testify"
  version = "1.1.4"
```

**Do:**
```
[[constraint]]
  name = "github.com/stretchr/testify"
  version = "^1.1.4"
```

## Structured logging

**Don't:**
```go
log.Printf("Listening on :%d", port)
http.ListenAndServe(fmt.Sprintf(":%d", port), nil)
// 2017/07/29 13:05:50 Listening on :80
```

**Do:**
```go
import "github.com/sirupsen/logrus"
// ...

logger.WithField("port", port).Info("Server is listening")
http.ListenAndServe(fmt.Sprintf(":%d", port), nil)
// {"level":"info","msg":"Server is listening","port":"7000","time":"2017-12-24T13:25:31+01:00"}
```

This is a harmless example, but using structured logging makes debugging and log
parsing easier.

## Avoid global variables

**Don't:**
```go
var db *sql.DB

func main() {
	db = // ...
	http.HandleFunc("/drop", DropHandler)
	// ...
}

func DropHandler(w http.ResponseWriter, r *http.Request) {
	db.Exec("DROP DATABASE prod")
}
```

Global variables make testing and readability hard and every method has access
to them (even those, that don't need it).

**Do:**
```go
func main() {
	db := // ...
	handlers := Handlers{DB: db}
	http.HandleFunc("/drop", handlers.DropHandler)
	// ...
}

type Handlers struct {
	DB *sql.DB
}

func (h *Handlers) DropHandler(w http.ResponseWriter, r *http.Request) {
	h.DB.Exec("DROP DATABASE prod")
}
```
Use structs to encapsulate the variables and make them available only to those functions that actually need them by making them methods implemented for that struct.

If you really need global variables or constants, e.g., for defining errors or string constants, put them at the top of your file.

**Don't:**
```go
import "xyz"

func someFunc() {
	//...
}

const route = "/some-route"

func someOtherFunc() {
	// usage of route
}

var NotFoundErr = errors.New("not found")

func yetAnotherFunc() {
	// usage of NotFoundErr
}
```

**Do:**
```go
import "xyz"

const route = "/some-route"

var NotFoundErr = errors.New("not found")

func someFunc() {
	//...
}

func someOtherFunc() {
	// usage of route
}

func yetAnotherFunc() {
	// usage of NotFoundErr
}
```


## Testing

### Use testify as assertion libary

**Don't:**
```go
func TestAdd(t *testing.T) {
	actual := 2 + 2
	expected := 4
	if (actual != expected) {
		t.Errorf("Expected %d, but got %d", expected, actual)
	}
}
```

**Do:**
```go
import "github.com/stretchr/testify/assert"

func TestAdd(t *testing.T) {
	actual := 2 + 2
	expected := 4
	assert.Equal(t, expected, actual)
	// OR
	require.Equal(t, expected, actual)
}
```

Using assert libraries makes your tests more readable, requires less code and provides consistent error output.

### Use sub-tests to structure functional tests
**Don't:**
```go
func TestSomeFunctionSuccess(t *testing.T) {
	// ...
}

func TestSomeFunctionWrongInput(t *testing.T) {
	// ...
}
```

**Do:**
```go
func TestSomeFunction(t *testing.T) {
	t.Run("success", func(t *testing.T){
		//...
	})
	
	t.Run("wrong input", func(t *testing.T){
		//...
	})
}
```

### Use table driven tests

**Don't:**
```go
func TestAdd(t *testing.T) {
	assert.Equal(t, 1+1, 2)
	assert.Equal(t, 1+-1, 0)
	assert.Equal(t, 1, 0, 1)
	assert.Equal(t, 0, 0, 0)
}
```

The above approach looks simpler, but it's much harder to find a failing case,
especially when having hundreds of cases.

**Do:**
```go
func TestAdd(t *testing.T) {
	cases := []struct {
		A, B, Expected int
	}{
		{1, 1, 2},
		{1, -1, 0},
		{1, 0, 1},
		{0, 0, 0},
	}

	for _, tc := range cases {
		t.Run(fmt.Sprintf("%d + %d", tc.A, tc.B), func(t *testing.T) {
			assert.Equal(t, t.Expected, tc.A+tc.B)
		})
	}
}
```

Using table driven tests in combination with subtests gives you direct insight
about which case is failing and which cases are tested.
– [Mitchell Hashimoto at GopherCon 2017](https://youtu.be/8hQG7QlcLBk?t=7m34s)

### Avoid unnecessary mocks

**Don't:**
```go
func TestRun(t *testing.T) {
	mockConn := new(MockConn)
	run(mockConn)
}
```

**Do:**
```go
func TestRun(t *testing.T) {
	ln, err := net.Listen("tcp", "127.0.0.1:0")
	t.AssertNil(t, err)

	var server net.Conn
	go func() {
		defer ln.Close()
		server, err := ln.Accept()
		t.AssertNil(t, err)
	}()

	client, err := net.Dial("tcp", ln.Addr().String())
	t.AssertNil(err)

	run(client)
}
```

Only use mocks if not otherwise possible, favor real implementations.
– [Mitchell Hashimoto at GopherCon 2017](https://youtu.be/8hQG7QlcLBk?t=26m51s)

### Avoid DeepEqual

**Don't:**
```go
type myType struct {
	id         int
	name       string
	irrelevant []byte
}

func TestSomething(t *testing.T) {
	actual := &myType{/* ... */}
	expected := &myType{/* ... */}
	assert.True(t, reflect.DeepEqual(expected, actual))
}
```

Instead use the [cmp module](https://github.com/google/go-cmp) or serialize the structs before comparision.
– [Mitchell Hashimoto at GopherCon 2017](https://youtu.be/8hQG7QlcLBk?t=30m45s)

### Avoid testing unexported funcs
Only exported functions should be tested so that packages can be more easily refactored without touching the tests as long as the public interface stays the same.
Only test unexported functions if you can't access the execution path via the exported functions.

### Add examples to your test files to demonstrate usage
```go
func ExamleSomeInterface_SomeMethod(){
	instance := New()
	result, err := instance.SomeMethod()
	fmt.Println(result, err)
	// Output: someResult, <nil>
}
```
This is expecially relevant for modules that are supposed to be used by multiple projects.

## Use linters

Of course, `gofmt` is a must. Only commit formatted files.

Use all the linters included in [gometalinter](https://github.com/alecthomas/gometalinter) to lint your projects before committing.
```bash
# Installation
go get -u gopkg.in/alecthomas/gometalinter.v2
gometalinter.v2 --install

# Usage in the project workspace
gometalinter.v2 --vendor ./...
```

Also tools like [goimports](https://godoc.org/golang.org/x/tools/cmd/goimports) can be used in most IDEs to format/update the import statements as you go.

### Properly refactor when fixing complexity warnings 
Sometimes the linter shows a warning about cyclomatic complexity, i.e. for the example below you will see something like this:
```
warning: cyclomatic complexity 13 of function DoLotsOfThings() is high (> 10) (gocyclo)
```
When fixing this it is important to still stick to CLEAN Code principles like keeping functions on the same level and not just refactor just as much that the warning disappears.

<details>
 <summary>Example</summary>

**Example**
```go
func DoLotsOfThings(input1 []someStruct, input2 *someStruct) error {
	for _, elem := range input1 {
		if elem.Method() == "" || elem.Route() == "" {
			return errors.New("some error")
		}

		if !strings.HasPrefix(elem.Route(), input2.Name) {
			return errors.New("some error")
		}

		if elem.Handler() == nil {
			return errors.New("some error")
		}

		if input2.someFunc == nil && !elem.IsPublic() {
			return errors.New("some error")
		}

		if input2.someOtherFunc == nil {
			return errors.New("some error")
		}

		switch c := elem.(type) {
		case *SomeType1:
			c.route = strings.TrimPrefix(c.route, input2.Name)
			c.group = input2
		case *SomeType2:
			c.route = strings.TrimPrefix(c.route, input2.Name)
		default:
			return errors.New("unknown type")
		}
	}

	return nil
}
```

**Don't:**
```go
func DoLotsOfThings(input1 []someStruct, input2 *someStruct) error {
	for _, elem := range input1 {
		if elem.Method() == "" || elem.Route() == "" {
			return errors.New("some error")
		}

		if !strings.HasPrefix(elem.Route(), input2.Name) {
			return errors.New("some error")
		}

		if elem.Handler() == nil {
			return errors.New("some error")
		}

		if input2.someFunc == nil && !elem.IsPublic() {
			return errors.New("some error")
		}

		if input2.someOtherFunc == nil {
			return errors.New("some error")
		}

		err := applySettings(elem)
		if err != nil {
			return err
		}
	}

	return nil
}
```

**Do:**
```go
func DoLotsOfThings(input1 []someStruct, input2 *someStruct) error {
	for _, elem := range input1 {
		if err := validateRouteAndMethod(elem, input2); err != nil {
			return err
		}

		if err := validateFunc(elem, input2); err != nil {
			return err
		}

		if err := applySettings(elem); err != nil {
			return err
		}
	}

	return nil
}
```

</details>

## Use meaningful variable names
Single-letter variable names should be used with care. They may seem more readable to you at the moment of writing but they make the code hard to understand for your colleagues and your future self.  

**Don't:**
```go
func findMax(l []int) int {
	m := l[0]
	for _, n := range l {
		if n > m {
			m = n
		}
	}
	return m
}
```

**Do:**
```go
func findMax(inputs []int) int {
	max := inputs[0]
	for _, value := range inputs {
		if value > max {
			max = value
		}
	}
	return max
}
```
Single-letter variable names are fine in the following cases.
* They are absolut standard like ...
	* `t` in tests
	* `r` and `w` in http request handlers
	* `i` for the index in a loop
* They name the receiver of a method, e.g., `func (s *someStruct) myFunction(){}`

Of course also too long variables names like `createInstanceOfMyStructFromString` should be avoided as well.

For more guidance read [this section of Dave Chaneys blog](https://dave.cheney.net/practical-go/presentations/qcon-china.html#_identifiers).

## Favour pure functions

> In computer programming, a function may be considered a pure function if both of the following statements about the function hold:
> 1. The function always evaluates the same result value given the same argument value(s). The function result value cannot depend on any hidden information or state that may change while program execution proceeds or between different executions of the program, nor can it depend on any external input from I/O devices.
> 2. Evaluation of the result does not cause any semantically observable side effect or output, such as mutation of mutable objects or output to I/O devices.

– [Wikipedia](https://en.wikipedia.org/wiki/Pure_function)

**Don't:**
```go
func MarshalAndWrite(some *Thing) error {
	b, err := json.Marshal(some)
	if err != nil {
		return err
	}

	return ioutil.WriteFile("some.thing", b, 0644)
}
```

**Do:**
```go
// Marshal is a pure func (even though useless)
func Marshal(some *Thing) ([]bytes, error) {
	return json.Marshal(some)
}

// ...
```

This is obviously not possible at all times, but trying to make functions pure if possible makes the code easier to follow and to test.

## Keep interfaces small
If possible, rely on existing interfaces when specifying expected input arguments. Also do not make functions expect larger interfaces then they actually need to perform their task.

When you create interfaces yourself, then remember: The smaller the interfaces, the more powerful they are. If you need bigger interfaces try to use composition to build them up from smaller ones.

**Don't:**
```go
type Server interface {
	Serve() error
	String() string
	SomeOtherMethod() float64
	YetAnotherMethod() int
}

func debug(srv Server) {
	fmt.Println(srv.String())
}

func run(srv Server) {
	srv.Serve()
}
```

**Do:**
```go
type Server interface {
	Serve() error
}

func (s *Server) String() {
	// ...
}

func debug(v fmt.Stringer) {
	fmt.Println(v.String())
}

func run(srv Server) {
	srv.Serve()
}
```

## Don't under-package

Deleting or merging packages is far easier than splitting big ones up. When unsure if a package can be split, do it.

## Handle signals

**Don't:**
```go
func main() {
	for {
		time.Sleep(1 * time.Second)
		ioutil.WriteFile("foo", []byte("bar"), 0644)
	}
}
```

**Do:**
```go
func main() {
	logger := // ...
	sc := make(chan os.Signal, 1)
	done := make(chan bool)

	go func() {
		for {
			select {
			case s := <-sc:
				logger.WithField("signal", s.String()).Info("Received signal, stopping application")
				done <- true
				return
			default:
				time.Sleep(1 * time.Second)
				ioutil.WriteFile("foo", []byte("bar"), 0644)
			}
		}
	}()

	signal.Notify(sc, os.Interrupt, os.Kill)
	<-done // Wait for go-routine
}
```

Handling signals allows us to gracefully stop our server, close open files and connections and therefore prevent file corruption among other things.

## Divide imports

**Don't:**
```go
import (
	"encoding/json"
	"github.com/some/external/pkg"
	"fmt"
	"github.com/this-project/pkg/some-lib"
	"os"
)
```

**Do:**
```go
import (
	"encoding/json"
	"fmt"
	"os"

	"git.fastbill.com/this-project/pkg/some-lib"

	"git.fastbill.com/another-project/pkg/some-lib"
	"git.fastbill.com/yet-another-project/pkg/some-lib"

	"github.com/some/external/pkg"
	"github.com/some-other/external/pkg"
)
```

Divide imports into four groups sorted from internal to external for readability:
1. Standard library
2. Project internal packages
3. Company internal packages
4. External packages

## Avoid naked return

**Don't:**
```go
func run() (n int, err error) {
	// ...
	return
}
```

**Do:**
```go
func run() (n int, err error) {
	// ...
	return n, err
}
```

Named returns are good for documentation, naked returns are bad for readability and error-prone.

## Avoid empty interface

**Don't:**
```go
func run(foo interface{}) {
	// ...
}
```

Empty interfaces make code more complex and unclear, avoid them where you can. With `interface{}` you do not really get the benefits of a typed language.

## Order functions public-private and top-down
When arranging your functions inside a package go by these rules in the given order.
1. *From public to private*: All exported function/methods come first in whatever order makes sense. Only after all of those put the unexported functions and methods.
2. *From top to bottom*: Helper functions are below the functions they are used in so the high level code can be read first and then the reader can dive into the details. E.g., in the main package `main(){...}` should be the first function

**Don't:**
```go
package example

func (s *someStruct) someHelper() int {
	someOtherHelper()
	// ...
}

func (s *someStruct) ExportedFunction1 {
	s.someHelper()
	// ...
}

func someOtherHelper() string {
	// ...
}

func (s *someStruct) ExportedFunction2 {
	// ...
}
```

**Do:**
```go
package example

func (s *someStruct) ExportedFunction1 {
	s.someHelper()
	// ...
}

func (s *someStruct) ExportedFunction2 {
	// ...
}

func (s *someStruct) someHelper() int {
	someOtherHelper()
	// ...
}

func someOtherHelper() string {
	// ...
}
```

Putting `main()` first makes reading the file a lot more easier. Only the
`init()` function should be above it.

## Avoid helper/util

Use clear names and try to avoid creating a `helper.go`, `utils.go` or even package.

## Structs
### Use named structs
Always include field names when instantiating it. This prevents your code from breaking in case an additional field is added to the struct.

**Don't:**
```go
params := request.Params{
	"http://example.com",
	"POST",
	map[string]string{"Authentication": "someToken"}
	someStruct,
}
```

**Do:**
```go
params := request.Params{
	URL: "http://example.com",
	Method: "POST",
	Headers: map[string]string{"Authentication": "someToken"}
	Body: someStruct,
}
```

### Avoid new keyword
Using the normal syntax instead of the `new` keyword makes it more clear what is happening: a new instance of the struct is created `MyStruct{}` and we get the pointer for it with `&`.

**Don't:**
```go
s := new(MyStruct)
```

**Do:**
```go
s := &MyStruct{}
```

### Anonymous structs are ok for JSON parsing
If you need parse a JSON request and the struct you would need to create for that purpose is not used anywhere else in the application, use an anonymous struct.
This has a small performance penalty but it improves the readability because the available fields in the struct are directly visible.

```go
input := struct{
	Name string `json:"name"`
}{}

err := json.NewDecoder(req.Body).Decode(&input)
fmt.Println(input.Name)
```

## Packages/SDKs
### Return the error
If possible the package should not require passing a logger, instead the functions/methods should just return the error so it can be handled by the consumer.  
**Don't:**
```go
type SDK struct {
	logger *log.Logger
}

func New(logger *log.Logger) *SDK {
	return &SDK{logger}
}

func(s *SDK) IsValid(s string) bool {
	// ...
	s.logger.Error("could not parse input")
	return false
}
```

**Do:**
```go
type SDK struct {}

func New() *SDK {
	return &SDK{}
}

func(s *SDK) Validate(s string) error {
	// ...
	return errors.New("could not parse input")
}
```

If you want to pass on the http error code that was returned from some (external or internal) service, use a specific error type like [httperrors](https://github.com/fastbill/httperrors).

**Do:**
```go
import "github.com/fastbill/httperrors"

func CallOtherService() error {
	// ...
	req.Header.Add("Authorization", "Bearer sometoken")
	resp, err := client.Do(req)
	if resp.StatusCode == http.StatusUnauthorized {
		return httperrors.New(http.StatusUnauthorized, nil)
	}
	// ...
}
```

### Return a struct on setup
Interfaces should be a matter of the consumer of your package. When setting up your SDK you should return a struct instead of an interface. That way the consumer can easily extend it by embedding it, adding methods etc. and then writing an interface that fits the consumers use case.  
See also [Go Wiki](https://github.com/golang/go/wiki/CodeReviewComments#interfaces).

**Don't:**
```go
type Thinger interface {
	// Foo does something cool, ...
	Foo() bool
}

type thing struct {}

func New() Thinger {
	return thing{}
}

// Foo implements Thinger#Foo
func (t thing) Foo() bool {
	// ...
}
```

**Do:**
```go
type Thinger interface { 
	Foo() bool
}

type Thing struct {}

func New() Thing {
	return Thing{}
}

// Foo does something cool, ...
func (t Thing) Foo() bool {
	// ...
}
```
As you see in the examples when returning the struct it needs to be public. But since you can always make all fields private no harm is done by this. Also note that the proper documentation should be on the methods, not on the interface definition so it shows up correctly in IDEs and the GoDocs.

### Provide an interface and a mock for optional use
To avoid code duplication in the consumer code, the package can contain an interface as shown in the example above. It is not used by the package itself, it just serves as a default interface in case the consumer does not need any customization.

Additionally if your package cannot be used "as-is" in unit tests (e.g., because it makes calls to external services) you can provide a mock implementation for the default interface in a `mock` subpackage. If you are working with [Testify Mocks](https://github.com/stretchr/testify#mock-package) it would look like this.

```go
package mock // import "github.com/name/my-sdk/mock"

import "github.com/stretchr/testify/mock"

type Thing struct {
	mock.Mock
}

func (t *Thing) Foo() bool {
	args := m.Called()
	return args.Bool(0)
}
```

### Extend interface when embedding structs
If you embed an existing struct definition in a new struct you define in your package take this in consideration when providing an interface in your package. In the example below `Service` was embedded and it fulfills the `Servicer` interface. Now the interface that abstracts `CustomService` should extend the `Servicer` interface so that if the consumer of the package that user the interface has access to both sets of methods. This of course only applies if it is known that the consumer will need both method sets. Otherwise interfaces should be kept small.

```go
type CustomService struct {
	service.Service
	field1 string
	field2 string
}

func (c *CustomService) SomeMethod() {
	// ...
}

type CustomerServicer interface {
	service.Servicer
	SomeMethod()
}
```

## Consistent header naming
**Don't:**
```go
r.Header.Get("authorization")
w.Header.Set("Content-type")
w.Header.Set("content-type")
w.Header.Set("content-Type")
```

**Do:**
```go
r.Header.Get("Authorization")
w.Header.Set("Content-Type")
```
