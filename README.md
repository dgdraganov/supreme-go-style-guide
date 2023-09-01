
# Go Style Guide

- [Introduction](#introduction)
- [Style](#style)
    - [Naming](#naming)
        - [Interfaces](#interfaces)
        - [Files](#files)
        - [Globals](#globals)
        - [Constants](#constants)
        - [Modules](#modules)
        - [Errors](#errors)
        - [Tests](#tests)
    - [The any keyword](#the-any-keyword)
    - [Initializing Structs](#initializing-structs)
- [Efficiency](#efficiency)
    - [Initialize with capacity](#initialize-with-capacity)
    - [Struct fields ordering](#struct-fields-ordering)
    - [Naked returns](#naked-returns)
    - [HTTP connection reuse](#http-connection-reuse)
    - [Parsing data](#parsing-data)
- [Best practice](#best-practice)
    - [Error handling](#error-handling)
    - [Error wrapping](#error-wrapping)
    - [Mocking](#mocking)


## Introduction

Every code base needs standardization and good practices! The idea behind this guide is to set the conventions and standards that are and will be used company wide in order achieve consistency and high quality of the code base. 

Here are some great resources that can provide guidance and best practices for projects using the Go language. 

- [Effective Go](https://golang.org/doc/effective_go.html)
- [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md#introduction)
- [Go standard project layout](https://github.com/golang-standards/project-layout)


## Style

### Naming

### Interfaces

***Pascal/Camel case***

By convention, one-method interfaces are named by the method name plus an -er suffix or similar modification to construct an agent noun: 

 ```go
type Reader  interface {...}
type Writer interface {...}

type tokenIssuer interface {...}
```

For more complex interfaces many languages prefer the 'I' convention (or 'i' for non exported types in Go):

```go
type IZendeskAPI interface {...}
type IDatabaseClient  interface {...}

type iRobotClient interface {...}
```

### Packages

***Kebab case***

Here are some package naming rules:

- no plurals
- single word
- all lower case
- no `common`, `util`, `shared`, or `lib` - these are bad, uninformative names

It's considered best practice for package aliasing to be avoided. Always use the full name of the package to reference types and globals. Aliasing should be used in case two or more packages have conflicting names or there is a version at the end of the package name:

```go
import(
    kole1 "gitlab.com/me/app/kole"
    kole2 "gitlab.com/me/app/pkg/kole"

    trace "example.com/trace/v2"
)
```

If multiple words need to be used in a package name/directory `-` should be used as a separator:

```go
import redis "github.com/redis/go-redis/v9"
```

### Files

***Snake case***

File names should consist of a single word. In case of two or more words `_` should be used as a separator:

```go
client.go
dp.go
payment_api.go
```

### Globals
***Pascal/Camel case***

Naming unexported globals with prefix `_` . This makes it clear for the reader where this variable is coming from.

```go
var (
    _serviceType = "whole-glory"
    _defaultPort = "9205"
)
```

### Constants

***All upper kebab case***

```go
var(
    GRAMS = 4.20
    TELEPORT_TO = "SecretCowLevel"
)
```

### Modules

Modules should be named with full repository path. This allows easy import and install of the package.


<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
module gin

go 1.20
```

</td><td>

```go
module github.com/gin-tonic/gin

go 1.20
```
</td></tr>
</tbody>
</table>

### Errors

For error values stored as global variables, use the prefix `Err` or `err` depending on whether they're exported:

```go
var (
    ErrReceivingMsg = errors.New("Cole, did you receive?")

    errNotFound = errors.New("not found")
)
```

For custom error types, use the suffix `Error` instead:

```go
type NotFoundError struct {
    File string
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("file %q not found", e.File)
}
```

### Tests

When naming tests longer and more descriptive names are actually encouraged. The name of the test needs to provide us with enough information about
- which method/function is being tested
- which case is being tested
- what is the expected result
A great example of naming tests (when not applying Table Driven Testing) is the **Roy Osherove’s naming strategy**. It devides the test name into three parts: 

`Test_<function_name>_<test_case>_<expected_result>`

Let's say we have the below function that needs to be tested: 

```go
var ErrEmptyUsername = errors.New("username is empyt")

func GetPlayerData(username string) (PlayerData, error) {
	if username == "" {
		return PlayerData{}, ErrEmptyUsername
	}
	...
}
```


### The `any` keyword

Use `any` instead of `interface{}` where possible

```go
m := map[string]any{}
```
```go
var tr any
```

but

```go
m := map[string]interface{
    Get(string)int
}{}
```
```go 
var getter interface {
        Get(string) int
    }
```

### Initializing Structs

We should almost always specify field names when initializing structs. This makes reading und understanding the code much easier.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
k := User{"Koll", "Et'o", true}
```

</td><td>

```go
user := User{
    FirstName: "Koll",
    LastName: "Et'o",
    HasReceived: true,
}
```

</td></tr>
</tbody></table>

## Efficiency

### Initialize with capacity

When creating slices and maps we have the option to provide initial length and capacity:

```go
sl := make([]int, 3, 7)
// len(sl) == 3
// cap(sl) == 7
```

The above example will create a slice of type `int` with `len(sl) == 3` and `cap(sl) == 7`. 
In case we know what slice length we will be using we always need to initialize the capacity in advance:

```go
numOfDigis = 10
digits := make([]int, 0, numOfDigis)
for i:=0; i < numOfDigis; i++{
    digits = append(digits, i)
}
```

Initializing slices with capacity can greatly improve the performance of an application as it removes the need for dynamic resizing (which involves a lot of allocations and data copying).

### Struct fields ordering

The order of the fields used in the struct directly affects the memory usage.

Resource: https://en.wikipedia.org/wiki/Data_structure_alignment

```go
type Poo struct {
    a  bool    // 1 byte
    b int64    // 8 bytes
    c  bool    // 1 byte
    d int64    // 8 bytes
    e  bool    // 1 byte
    f int64    // 8 bytes
    g  bool    // 1 byte
}
```

One might expect the size of an object of the type `Poo` to be `28` bytes. 

```go
func main() {
    u := Poo{}
    fmt.Println(unsafe.Sizeof(u)) // 56 bytes
}
```

Due to how memory alignment works internally in a 64-bit architecture for all `bool` type fields there will be `8` bytes reserved instead of `1`.

A great memory improvement can be made by reordering the fields:

```go
// 32 bytes in total
type Poo struct {
    a  bool    // 1 byte
    g  bool    // 1 byte
    e  bool    // 1 byte
    c  bool    // 1 byte
    b int64    // 8 bytes
    d int64    // 8 bytes
    f int64    // 8 bytes
}
```

If we now check the size of a `Poo` object it will be significantly smaller - `32` bytes. 


### Naked returns

Naked returns (or named returns) can harm readability but also can increase performance.

Resource: https://blog.min.io/golang-internals-part-2-nice-benefits-of-named-return-values-2/

<table>
<thead><tr><th>Readability</th><th>Performance</th></tr></thead>
<tbody>
<tr><td>

```go
func IllBeBack(phrase string)(int, error){
    if phrase {
        return 0, errors.New("it went kaput")
    } else if{
        return 23, nil
    }
    ...
    return 42, nil
}
```

</td><td>

```go
func IllBeBack(phrase string)(num int, err error){
    if phrase {
        num, err = 0, errors.New("it went kaput")
        return 
    } else if{
        num, err = 23, nil
        return 
    }
    num, err = 42, nil
    return
}
```
</td></tr>
</tbody></table>


Using naked returns reduces the code generated by the compiler which in terms leads to performance benefits. 

Luckily Go offers a compromise between readability and performance for this specific case - we can explicitly return the variables but still use naked returns:

```go
func IllBeBack(phrase string)(num int, err error){
    if phrase {
        ...)
        return num, err
    } else if{
        ...
        return num, err
    }
    ...
    return num, err
}
```

</td></tr>
</tbody></table>

### HTTP connection reuse


`http` package will always try to reuse connections for performance reasons. However we need to be careful not to block it from doing it.

Here are the most common reasons that prevent connection reuse:

- **not closing the response body** 

Always close the response body - `res.Body.Close()`. That's all. Let's move on!

- **not reading the response body** 

Resource: https://husni.dev/how-to-reuse-http-connection-in-go/

In some cases we don't need the response body and decide to close it withoout reading it.
```go
defer res.Body.Close()
```
This will block the connection from being reused! We always need to read the response body. The below example shows how the body can be read and discarded: 

```go
defer res.Body.Close()
defer io.Copy(ioutil.Discard, res.Body)
```

- **creating new `http.Client` before each request**

Creating new `http.Client` object before each request prevents connection reuse! To avoid this problem, we can create a single `http.Client` and use it for all future requests (in the scope of the current object):

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type DEAHttpClient struct{}

func (c *DEAHttpClient) GetMeth(req *http.Request) {
    client := http.Client{}
    resp, _ := client.Do(req)
    ...
}

```

</td><td>

```go
type DEAHttpClient struct{
    client http.Client
}

func (c *DEAHttpClient) GetMeth(req *http.Request) {
    resp, _ := c.client.Do(req)
    ...
}

```
</td></tr>
</tbody>
</table>

### Parsing data

Go provides two ways of serialization and deserialization of `xml` and `json` data - `Marshal`/`Unmarshal` and `Encode`/`Decode`

If the source of data we need to deserialize implements the `io.Reader` interface the right way to deserialize is by using `json.Decoder` (or `xml.Decoder`) and not `json.Unmarshal` (or `xml.Unmarshal`):


<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
bytes, _ := io.ReadAll(reader)
var pl Player
_ = json.Unmarshal(bytes, &pl)

```

</td><td>

```go
var pl Player
err := json.NewDecoder(reader).Decode(&pl)
```
</td></tr>
</tbody>
</table>

Similarly if the data we serialize will be written to an object implementing the `io.Writer` interface the better way would be to use `json.Encoder` (or `xml.Encoder`) and not `json.Marshal` (or `xml.Marshal`):


<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
var pl Player
bytes, _ := json.Marshal(pl)
nb, _ := writer.Write(bytes)
```

</td><td>
```go
var pl Player
_ = json.NewEncoder(writer).Encode(pl)
```
</td></tr>
</tbody>
</table>

For the cases where we do not have an `io.Writer` or `io.Reader` at our dispose we should simply use `Marshal` and `Unmarshal`.

## Best practice

### Error handling

The basic rule is to "**handle errors only once**". The idea is to propagate the error upwards in the code hierarchy until it is logged or handled gracefully. 

**Do NOT log AND return an error!** Doing so is causing a lot of noise in the application logs for little value. Either wrap an error with additional context information and return it or handle and/or log it.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func HoldTheDoor() error{
    err := GetUserData()
    log.Errorf("Hodor: %s", err)
    return err
}
```

</td><td>

```go
func HoldTheDoor() error{
    err := GetUserData()
    return fmt.Errorf("getting user data: %w", err)
}
```
</td></tr>
</tbody>
</table>

### Error wrapping

Errors wrapping provides the benefit of being able to check whether a specific error happened in the chain of errors. 

Vanilla Go wrapping is performed with `%w` in string formatting:


<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
fmt.Errorf("getting PT data: %s", err)
```

</td><td>

```go
fmt.Errorf("getting PT data: %w", err)
```

</td></tr>
</tbody>
</table>

Here is how to check if an error chain contains a specific error:

```go
ErrFailedSuccess := errors.New("failed successfully")

err := fmt.Errorf("check: %w", ErrFailedSuccess)
err = fmt.Errorf("sure: %w", err)
err = fmt.Errorf("test: %w", err)

fmt.Println(err)
// `test: sure: check: failed successfully`

fmt.Println(errors.Is(err, ErrFailedSuccess))
// `true`
```

Do not add words like `failed`, `error`, `unsuccessful` , etc. They provide no additional information since we already know an error has occured. Usually the inner most error will include information regarding what happened and why. 


<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
pl, err := pt.GetPlayer()
if err != nil {
    return fmt.Errorf(
        "getting player failed: %w", err)
}
```

</td><td>

```go
pl, err := pt.GetPlayerData()
if err != nil {
    return fmt.Errorf(
        "getting player: %w", err)
}
```

</td></tr><tr><td>

```plain
failed this: error that: getting player failed: error loading pt certs
```

</td><td>

```plain
this: that: getting player: error loading pt certs
```

</td></tr>
</tbody></table>


### Mocking

A great practice for mocking Go structs is to use fields of type `func` that each represent a method we would like to mock:

Let's consider the below example:

```go
type SupplierAPI struct{}

func (api *SupplierAPI) GetInfo() string{
    // ...
}

func (api *SupplierAPI) IsUpToDate() bool{
    // ...
}
```

In order to avoid creating multiple mock objects (one for each test case) we can use the below technique: 

```go
type SupplierAPIMock struct {
    getInfoFunc    func() string
    isUpToDateFunc func(string) bool
}

func (api *SupplierAPIMock) GetInfo() string {
    return api.getInfoFunc()
}

func (api *SupplierAPIMock) IsUpToDate(data string) bool {
    return api.isUpToDateFunc(data)
}
```

Now for every test case we can plug mocked functionality depending on what we want to test and we no longer need to create separate mock objects:

```go
test_value := "test_value_1"

supp := SupplierAPIMock{
    getInfoFunc: func()string{ return test_value },
    isUpToDateFunc: func(data string)bool{ 
        if data == test_value{
        	return true
        }
        return false // or fail test
    },
}
```

### Table driven tests

When it comes to designing unit tests the best practice (if applicable) is to use **table driven testing (TDT)**. This technique improves readability and tests management by grouping test cases that refer to the same function. Consider the below example:

```go
func Sum(a, b int) int{
    return a + b
}
```

By using **TDT** we can unit test this function the following way:

```go
func Test_Sum(t *testing.T) {
    tests := map[string]struct {
        a        int
        b        int
        expected int
    }{
        "10+5": {
            a:        10,
            b:        5,
            expected: 15,
        },
        "0+0": {
            a:        0,
            b:        0,
            expected: 0,
        },
    }

    for name, test := range tests {
        t.Run(name, func(t *testing.T) {
            actual := sum(test.a, test.b)

            if actual != test.expected {
                t.Errorf("sum don't match: expected %d, got %d", test.expected, actual)
            }
        })
    }
}
```

**TDT** provides a great way of combining several tests to be executed together in the same unit test function. 



