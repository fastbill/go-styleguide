# Go Styleguide at FastBill <!-- omit in toc --> 

This serves as a supplement to [Effective Go](https://golang.org/doc/effective_go.html) and the [Go Wiki](https://github.com/golang/go/wiki/CodeReviewComments), based on [github.com/bahlo/go-styleguide](https://github.com/bahlo/go-styleguide) by Arne Bahlo and our experience building Go services and modules.

## Table of contents <!-- omit in toc --> 

- [Errors And Logs](#errors-and-logs)
	- [Consistent Message Format](#consistent-message-format)
	- [Add Context to Errors](#add-context-to-errors)
	- [Visually Group Error Handling for a Function](#visually-group-error-handling-for-a-function)
	- [Keep (Shared) Errors In a Separate File](#keep-shared-errors-in-a-separate-file)
	- [Use Structured Logging](#use-structured-logging)
- [Dependency Management](#dependency-management)
	- [Use Go Modules](#use-go-modules)
	- [Use Semantic Versioning](#use-semantic-versioning)
	- [Go Modules and Docker](#go-modules-and-docker)
	- [Go Modules and Private Git Repositories](#go-modules-and-private-git-repositories)
- [Avoid Global Variables](#avoid-global-variables)
- [Testing](#testing)
	- [Use Testify as Assertion Library](#use-testify-as-assertion-library)
	- [Use Sub-Tests to Structure Functional Tests](#use-sub-tests-to-structure-functional-tests)
	- [Use Table Driven Tests](#use-table-driven-tests)
	- [Avoid Unnecessary Mocks](#avoid-unnecessary-mocks)
	- [Avoid DeepEqual](#avoid-deepequal)
	- [Avoid Testing Unexported Functions](#avoid-testing-unexported-functions)
	- [Avoid "_test" Packages](#avoid-_test-packages)
	- [Add Examples to Your Test Files to Demonstrate Usage](#add-examples-to-your-test-files-to-demonstrate-usage)
- [Use Linters](#use-linters)
	- [Properly Refactor When Fixing Complexity Warnings](#properly-refactor-when-fixing-complexity-warnings)
- [Use Meaningful Variable Names](#use-meaningful-variable-names)
	- [Not Too Short](#not-too-short)
	- [Not Too Long](#not-too-long)
- [Favour Pure Functions](#favour-pure-functions)
- [Keep Interfaces Small](#keep-interfaces-small)
- [Don't Under-Package](#dont-under-package)
- [Handle Signals](#handle-signals)
- [Structure Import Statements](#structure-import-statements)
- [Avoid Naked Return](#avoid-naked-return)
- [Avoid Empty Interface](#avoid-empty-interface)
- [Order Functions Top-Down](#order-functions-top-down)
- [Avoid Helper/Util](#avoid-helperutil)
- [Structs](#structs)
	- [Use Named Structs](#use-named-structs)
	- [Avoid "new" Keyword](#avoid-new-keyword)
	- [Anonymous Structs Are Ok for JSON Parsing](#anonymous-structs-are-ok-for-json-parsing)
	- [Consider providing a "Constructor"](#consider-providing-a-constructor)
- [Packages/SDKs](#packagessdks)
	- [Return the Error](#return-the-error)
	- [Return a Struct on Setup](#return-a-struct-on-setup)
	- [Provide an Interface and a Mock for Optional Use](#provide-an-interface-and-a-mock-for-optional-use)
	- [Extend Interface When Embedding Structs](#extend-interface-when-embedding-structs)
- [Consistent Header Naming](#consistent-header-naming)


## Errors And Logs

### Consistent Message Format
Error messages should start with a lowercase letter and should not end with a `.`. See [Wiki page on Errors](https://github.com/golang/go/wiki/Errors) for reference. For consistency the same logic should be applied for log messages and for error messages in custom error structs.

**Don't:**
```go
logger.Print("Something went wrong.")
ReadFailError := errors.New("Failed to read file")
```

**Do:**
```go
logger.Print("something went wrong")
ErrReadFailed := errors.New("failed to read file")
```

### Add Context to Errors

**Don't:**
```go
file, err := os.Open("foo.txt")
if err != nil {
	return err
}
```
Using the approach above can lead to unclear error messages because of missing context, especially as the error might travel up the function chain before being logged for example.

**Do:**
```go
fileName := "foo.txt"
file, err := os.Open(fileName)
if err != nil {
	return fmt.Errorf("opening %v failed: %w", fileName, err)
}
```
See [the official blog post](https://blog.golang.org/go1.13-errors) for more details on the new error wrapping introduced in Go 1.13.

As a general rule you should always add context to your errors via `fmt.Errorf`. This way you build up a good "text trace" as the error travels up the stack, e.g. `failed to create new account: failed to save API credentials to DB: connection error`.

Adding additional properties like `fileName` in the example above is optional as it sometimes just leads to duplication. Many Go errors (like errors from parsing) already contain all the necessary information (besides the context).

### Visually Group Error Handling for a Function
**Don't:**
```go
err := myFunction()

if err == ErrNotFound {
	return httperrors.New(http.StatusNotFound, err)
}

if err != nil {
	return err
}
```

**Do:**
```go
err := myFunction()
if err == ErrNotFound {
	return httperrors.New(http.StatusNotFound, err)
} else if err != nil {
	return err
}
```

Although the `else` statement is not logically necessary it helps to keep all the error handling for the error returned by `myFunction` tightly grouped together so the function and its error handling constitute one visual unit.

### Keep (Shared) Errors In a Separate File
If your package consists of multiple files consider putting all custom errors and error variables for that package into a separate file called `error.go`. That way they are easy to find and can be reused by other developers or your future self.

This is particularly important if an error variable is used in multiple files of the package, e.g. this standard one from a repository package:
```go
var ErrNotFound = errors.New("entity not found")
```

### Use Structured Logging

**Don't:**
```go
log.Printf("listening on :%d", port)
http.ListenAndServe(fmt.Sprintf(":%d", port), nil)
// 2017/07/29 13:05:50 Listening on :80
```

**Do:**
```go
import "github.com/sirupsen/logrus"
// ...

logger.WithField("port", port).Info("server is listening")
http.ListenAndServe(fmt.Sprintf(":%d", port), nil)
// {"level":"info","msg":"Server is listening","port":"7000","time":"2017-12-24T13:25:31+01:00"}
```

This is a harmless example, but using structured logging makes debugging and log
parsing easier.

## Dependency Management

### Use Go Modules
Use the new dependency management that was introduced with Go 1.13. See (the official blog posts)[https://blog.golang.org/using-go-modules] for more information. Include the `go.mod` and `go.sum` file in your source code management (e.g. git).

### Use Semantic Versioning
Since Go Modules can handle versions, tag your packages using [Semantic Versioning](http://semver.org).  
The git tag for your go package should have the format `v<major>.<minor>.<patch>`, e.g., `v1.0.1`.

### Go Modules and Docker
If you are working with Docker you can save yourself some trouble by using the Go modules in vendored mode. By default Go will keep all your third party libraries in one place (e.g. `~/go/pkg/mod`). When building a Docker image, Docker is not allowed to access anything outside of the current scope (the application directory). That means it will not be able to access and include the external Go modules when building the image. You can force Go to make a copy of the dependencies in a folder called `vendor` in the application directory. This is done by running `go mod vendor`.  
Since Go 1.14 all other Go commands (go test, go build, ...) will automatically use the the `vendor` folder if it exists. No need to explicitly set `-mod=vendor` anymore.

### Go Modules and Private Git Repositories
Currently the Go dependency management does not support fetching packages via ssh instead of https. But for many private repositories (e.g. on-premise GitLab) ssh is the only option to fetch the code without manually entering the password or generating some special https URLs that include authentication tokens. A lot of the discussion around this is collected in [this GitHub Issue](https://github.com/golang/go/issues/29953).

There is no really nice way to deal with this issue at the moment. We recommend to add the following to your global `.gitconfig` file to change the URLs used when checking out the private repositories.
```
[url "ssh://git@git.yourcompany.com/"]
    insteadOf = https://git.yourcompany.com/
```
⚠️ The caveat here is that this has to be done by all developers and in all CI pipelines that fetch the modules etc. In a CI script the following can be used to make the addition to the git config file.
```
echo -e "[url \"ssh://git@git.yourcompany.com/\"]\n\tinsteadOf = https://git.yourcompany.com/" > ~/.gitconfig'
```

By default Go tries to fetch modules from the official Go Proxy server. In that process the names of your private packages will be sent to Google, see [proxy.golang.org/privacy](https://proxy.golang.org/privacy). To avoid that you can exclude domains from being fetched via the proxy using the `GOPRIVATE` variable. To apply it for all the Go commands you run locally, use this command:
```
go env -w GOPRIVATE=git.yourcompany.com
```
Remember to also apply this environment variable in CI pipelines.

## Avoid Global Variables

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

### Use Testify as Assertion Library
Using an assertion library makes your tests more readable, requires less code and provides consistent error output. [Testify](https://github.com/stretchr/testify) is a powerful assertion library that contains a lot of nice helpers.

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
import "github.com/stretchr/testify/require"

func TestAdd(t *testing.T) {
	actual := 2 + 2
	expected := 4
	require.Equal(t, expected, actual)
}
```
Prefer the use of `require` instead of `assert`. With `require` the test execution is stopped in case the assertion fails which is usually the better choice than to continue with the next test that might also fail due to a follow up error and cause a confusing error message.

**Don't:**
```go
import "github.com/stretchr/testify/require"

func TestMyHandler(t *testing.T) {
	err := MyHandler()
	assert.Error(t, err)
	assert.Equal(t, http.StatusNotFound, err.(*httperrors.HTTPError).StatusCode)
	// the second assertion would be executed and even if the first assert was not successful
	// and it would result in:
	// "panic: interface conversion: error is nil, not *httperrors.HTTPError"
}
```

**Do:**
```go
import (
	"github.com/stretchr/testify/require"
)

func TestMyHandler(t *testing.T) {
	err := MyHandler()
	require.Error(t, err) {
	require.Equal(t, http.StatusNotFound, err.(*httperrors.HTTPError).StatusCode)
	// the second assertion will not be executed if the first one fails
}
```

### Use Sub-Tests to Structure Functional Tests
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

### Use Table Driven Tests

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

Using table driven tests in combination with sub-tests gives you direct insight
about which case is failing and which cases are tested.
– [Mitchell Hashimoto at GopherCon 2017](https://youtu.be/8hQG7QlcLBk?t=7m34s)

### Avoid Unnecessary Mocks

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

Instead use the [cmp module](https://github.com/google/go-cmp) or serialize the structs before comparison.
– [Mitchell Hashimoto at GopherCon 2017](https://youtu.be/8hQG7QlcLBk?t=30m45s)

### Avoid Testing Unexported Functions
Only exported functions should be tested so that packages can be more easily refactored without touching the tests as long as the public interface stays the same. Only test unexported functions if you can't access the execution path via the exported functions.

### Avoid "_test" Packages
Putting the tests of a package xy into a separate package `xy_test` makes it impossible to test any unexported functions from the package. To avoid exporting functions just for the test or writing a lot of duplicated test code that could be avoided e.g. by a table-driven test on an unexported function, put your test into the same package as the normal code. This is especially helpful when the code includes unexported functions that are run in a separate Go routine.

### Add Examples to Your Test Files to Demonstrate Usage
```go
func ExamleSomeInterface_SomeMethod(){
	instance := New()
	result, err := instance.SomeMethod()
	fmt.Println(result, err)
	// Output: someResult, <nil>
}
```
This is especially relevant for modules that are supposed to be used by multiple projects.

## Use Linters

Of course, `gofmt` is a must. Only commit formatted files.

Use the linters included in [Golangci-Lint](https://github.com/golangci/golangci-lint) to lint your projects locally and via the CI pipeline. Unfortunately a couple of nice linters like `gocyclo` are disabled by default in Golangci-Lint. To enable additional linters consistently you need to add a `.golangci.yml` file to your project. Here a recommendation:

```yml
linters:
  enable:
    - gocyclo
    - revive
    - dupl
    - unconvert
    - goconst
    - gosec
    - bodyclose

run:
  deadline: 10m
  modules-download-mode: vendor

issues:
  exclude-use-default: false
  exclude-rules:
    - path: _test\.go
      linters:
        - dupl
        - goconst
        - gosec
    - linters:
        - govet
      text: 'shadow: declaration of "err" shadows declaration'
    - linters:
        - revive
      text: 'in another file for this package'

linters-settings:
  gocyclo:
    min-complexity: 10
  revive:
    min-confidence: 0
  govet:
    check-shadowing: true
```

Also tools like [goimports](https://godoc.org/golang.org/x/tools/cmd/goimports) can be used in most IDEs to format/update the import statements as you go.

### Properly Refactor When Fixing Complexity Warnings
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

## Use Meaningful Variable Names

### Not Too Short
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

For more guidance read [this section of Dave Chaney's blog](https://dave.cheney.net/practical-go/presentations/qcon-china.html#_identifiers).

### Not Too Long
Of course also too long variables names like `createInstanceOfMyStructFromString` should be avoided as well.

Especially unnecessary verbs in the function name should be avoided. For getters this is detailed [here](https://golang.org/doc/effective_go.html#Getters). The same should be applied for example when naming functions that retrieve data from the database.

**Don't:**
```go
repo.GetDocumentByID(123)
repo.FindDocumentByID(123)
repo.FetchDocumentByID(123)
repo.RetrieveDocumentByID(123)
```

**Do:**
```go
repo.DocumentByID(123)
```


## Favour Pure Functions

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

## Keep Interfaces Small
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

## Don't Under-Package

Deleting or merging packages is far easier than splitting big ones up. When unsure if a package can be split, do it.

## Handle Signals

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

## Structure Import Statements

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

	"git.fastbill.com/another-project/pkg/some-lib"
	"git.fastbill.com/yet-another-project/pkg/some-lib"
	"github.com/some/external/pkg"
	"github.com/some-other/external/pkg"

	"git.fastbill.com/this-project/pkg/some-lib"
)
```

Divide imports into three groups for readability:
1. Standard library
2. External packages (can be provided by third-party or in-house)
3. Project internal packages

## Avoid Naked Return

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

## Avoid Empty Interface

**Don't:**
```go
func run(foo interface{}) {
	// ...
}
```

Empty interfaces make code more complex and unclear, avoid them where you can. With `interface{}` you do not really get the benefits of a typed language.

## Order Functions Top-Down
Arrange your functions *from top to bottom*: Helper functions are below the functions they are used in so the high level code can be read first and then the reader can dive into the details. E.g., in the main package `main(){...}` should be the first function.

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
```

**Do:**
```go
package example

func (s *someStruct) ExportedFunction1 {
	s.someHelper()
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
If two high level functions share some helper function, the helper function can be below any of the two high level functions.

## Avoid Helper/Util

Use clear names and try to avoid creating a `helper.go`, `utils.go` or even package.

## Structs
### Use Named Structs
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

### Avoid "new" Keyword
Using the normal syntax instead of the `new` keyword makes it more clear what is happening: a new instance of the struct is created `MyStruct{}` and we get the pointer for it with `&`.

**Don't:**
```go
s := new(MyStruct)
```

**Do:**
```go
s := &MyStruct{}
```

### Anonymous Structs Are Ok for JSON Parsing
If you need parse a JSON request and the struct you would need to create for that purpose is not used anywhere else in the application, use an anonymous struct.
This has a small performance penalty but it improves the readability because the available fields in the struct are directly visible.

```go
input := struct{
	Name string `json:"name"`
}{}

err := json.NewDecoder(req.Body).Decode(&input)
fmt.Println(input.Name)
```

### Consider providing a "Constructor"
If your struct contains fields that need to be set to properly work with it or some struct fields are generated, it makes sense to provide a constructor for your struct. The naming convention is to call the constructor `New<StructName>`.

```go
type Context struct {
	userID uint64
	accountID uint64
	requestID string
}

func NewContext(userID, accountID uint64) (*Context, error) {
	if userID == 0 || accountID == 0 {
		return errors.New("invalid arguments")
	}

	return &Context{
		userID: userID,
		accountID: accountID,
		requestID: generateRequestID(),
	}
}
```

## Packages/SDKs
### Return the Error
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

### Return a Struct on Setup
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

### Provide an Interface and a Mock for Optional Use
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

To generate testify mocks for interfaces the [go-mock-gen tool](https://github.com/fastbill/go-mock-gen) can be used.

### Extend Interface When Embedding Structs
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

## Consistent Header Naming
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
