# Chapter 3. Methods and interfaces

## 3.1 Methods

* _Note_: A method is a function that has a defined receiver in OOP terms, a method is a function on an instance of an object.

* Go does not have classes. However, you can define methods on struct types.

* The __method receiver__ appears in its own argument list between the `func` keyword and the method name. Here is an example with a `User` struct containing two fields: `FirstName` and `LastName` of string type.

  ```go
  package main

  import (
    "fmt"
  )

  type User struct {
    FirstName, LastName string
  }

  func (u *User) Greeting() string {
    return fmt.Sprintf("Dear %s %s", u.FirstName, u.lastName)
  }

  func main() {
    u := &User{"Matt", "Aimonetti"}
    fmt.Println(u.Greeting())
  }

  ```

### 3.1.1 Code organization

* Methods can be defined on any file in the package, but my recommendation is to organize the code as shown below:

  ``` go
  package models

  // list of packages to import

  import (
    "fmt"
  )

  // list of constants
  const (
    ConstExample = "const before vars"
  )

  // list of variables

  var (
    ExportedVar    = 42
    nonExportedVar = "so say we all"
  )

  // Main type(s) for the file,
  // try to keep the lowest amount of structs per file when possible.
  type User struct {
    FirstName, LastName string
    Location            *UserLocation
  }

  type UserLocation struct {
    City    string
    Country string
  }

  // List of functions
  func NewUser(firstName, lastName string) *User {
    return &User{
             FirstName: firstName,
             LastName: lastName,
             Location: &UserLocation{
               City:    "Santa Monica",
               Country: "USA"
             },
           }
  }

  // List of methods
  func (u *User) Greeting() string {
    return fmt.Sprintf("Dear %s %s", u.Firstname, u.LastName)
  }

  ```

* In fact, you can define a method on __any__ type you define in your package, not just structs. You cannot define a method on a type from another package, or on a basic type.

### 3.1.2 Type aliasing

* To define methods on a type you don't "own", you need to define an alias for the type you want to extend:

  ``` go
  package main

  import (
    "fmt"
    "strings"
  )

  type MyStr string

  func (s MyStr) Uppercase() string {
    return strings.ToUpper(string(s))
  }

  func main() {
    fmt.Println(MyStr("test").Uppercase())
  }
  ```

  ``` go
  package main

  import (
    "fmt"
    "math"
  )

  type MyFloat float64

  func (f MyFloat) Abs() float64 {
    if f < 0 {
      return float64(-f)
    }
    return float64(f)
  }

  func main() {
    f := MyFloat(-math.Sqrt(2))
    fmt.Println(f.Abs())
  }
  ```

### 3.1.3 Method receivers

* Methods can be associated with a named type or a pointer to a named type. There are two reasons to use a pointer receiver. First, to avoid copying the value on each method call (more efficient if the value type is a large struct). Second, so that the method can modify the value that its receiver points to.

  ``` go
  package main

  import (
    "fmt"
    "math"
  )

  type Vertex struct {
    X, Y float64
  }

  func (v *Vertex) Scale(f float64) {
    v.X = v.X * f
    v.Y = v.Y * f
  }

  func (v *Vertex) Abs() float64 {
    return math.Sqrt(v.X * v.X + v.Y * v.Y)
  }

  func main() {
    v := &Vertex{3, 4}
    v.Scale(5)
    fmt.Println(v, v.Abs())
  }
  ```

## 3.2 Interfaces

* An interface type is defined by a set of methods.

* A value of interface type can hold any value that implements those methods.

  ``` go
  package main

  import (
    "fmt"
    "math"
  )

  type Abser interface {
    Abs() float64
  }

  func main() {
    var a Abser
    f := MyFloat(-math.Sqrt(2))
    v := Vertex{3, 4}

    a = f // a MyFloat implements Abser
    a = &v // a *Vertex implements Abser

    // In the following line, v is a Vertex (not *Vertex)
    // and does NOT implement Abser.
    a = v

    fmt.Println(a.Abs())
  }

  type MyFloat float64

  func (f MyFloat) Abs() float64 {
    if f < 0 {
      return float64(-f)
    }
    return float64(f)
  }

  type Vertex Struct {
    X, Y float64
  }

  func (v *Vertex) Abs() float64 {
    return math.Sqrt(v.X * v.X + v.Y * v.Y)
  }
  ```

### 3.2.1 Interfaces are satisfied implicitly

* A type implements an interface by implementing the methods.

* There is not explicit declaration of intent.

* Implicit interfaces decouple implementation packages from the packages that define the interfaces: neither depends on the other.

* It also encourages the definition of precise interfaces, because you don't have to find every implementation and tag it with the new interface name.

  ``` go
  package main

  import (
    "fmt"
    "os"
  )

  type Reader interface {
    Read(b []byte) (n int, err error)
  }

  type Writer interface {
    Write(b []byte) (n int, err error)
  }

  type ReadWriter interface {
    Reader
    Writer
  }

  func main() {
    var w Writer

    // os.Stdout implements Writer
    w = os.Stdout
    fmt.Fprintf(w, "hello, writer\n")
  }
  ```

## 3.3 Errors

* An error is anything that can describe itself as an error string. The idea is captured by the predefined, built-in interface type, __error__, with its single method, `Error`, returning a string:
  ```go
  type error interface {
    Error() string
  }
  ```
* The `fmt` package's various print routines automatically know to call the method when asked to print an error.
  ``` go
  package main

  import (
    "fmt"
    "time"
  )

  type MyError struct {
    When time.Time
    What string
  }

  func (e *MyError) Error() string {
    return fmt.Sprintf("at %v, %s", e.When, e.What)
  }

  func run() error {
    return &MyError{time.Now(), "it didn't work"}
  }

  func main() {
    if err := run(); err != nil {
      fmt.Println(err)
    }
  }
  ```

## 3.4 Exercise: Errors

``` go
package main

import (
  "fmt"
)

type ErrNegativeSqrt float64

func (e ErrNegativeSqrt) Error() string {
  return fmt.Sprintf("cannot Sqrt negative number: %g", float64(e))
}

func Sqrt(x float64) (float64, error) {
  if x < 0 {
    return 0, ErrNegativeSqrt(x)
  }

  // nested function to generate an approximation
  approximation := func(z, x float64) float64 {
    return z - ((z * z) - x) / (2 * z)
  }

  z := 1.0
  for i := 0; i< 10; i++ {
    z = approximation(z, x)
  }

  return z, nil
}

func main() {
  fmt.Println(Sqrt(2))
  fmt.Println(Sqrt(-2))
}
```
