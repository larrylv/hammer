# Chapter 2. Basic Concepts

## 2.1 Packages and imports

* Programs start running in package __main__.

``` go
package main

func main() {
  print("Hello, World\n");
}
```

* A `main()` function in a `main` package will be the entry point to your software.

* The package name is the same as the last element of the import path.

## 2.2 Exported names

* After importing a package, your can refer to the names it exports, which begins with a capital letter.

## 2.3 Functions, signature, return values, named results

* A function can take zero or more typed arguments. The type comes after the variable name. Functions can be defined to return any number of values. Return values are always typed.

``` go
package main

import "fmt"

func add(x int, y int) int {
  return x + y
}

func main() {
  fmt.Println(add(42, 13))
}
```

* We can also only declare one type that applies to both arguments.

``` go
package main

import "fmt"

func add(x, y int) int {
  return x + y
}

func main() {
  fmt.Println(add(42, 13))
}
```

* Example for function with multiple results

``` go
package main

import "fmt"

func swap(x, y string) (string, string) {
  return y, x
}

func main() {
  a, b := swap("hello", "world")
  fmt.Println(a, b)
}
```

* Example for function with named results

``` go
package main

import "fmt"

func split(sum int) (x, y int) {
  x = sum * 4 / 9
  y = sum - x
  return
}

func main() {
  fmt.Println(split(17))
}
```

I personally recommend against using named return parameters because they often cause more confusion than they save time or help clarify your code.

### Resources

[Go's declaration syntax](http://blog.golang.org/gos-declaration-syntax)

## 2.4 Variables / inferred typing, short assignment

* The var statement declares a list of variables, the type is last.

``` go
package main

import "fmt"

var i int
var c, python, java bool

func main() {
  fmt.Println(i, c, python, java)
}
```

* A var declaration can include initializers, one per variable. If an initializer is present, the type can be omitted; the variable take the type of the initializer.

``` go
package main

import "fmt"

var i, j int = 1, 2
var c, python, java = true, false, "no!"

func main() {
  fmt.Println(i, c, python, java)
}
```

* `:=`

  - Inside a function, the `:=` short assignment can be used in place of a var declaration with implicit type.
  - Outside a function, every construct begins with a keyword(`var`, `func`, and so on) and the := construct is not available.

``` go
package main

import "fmt"

func main() {
  var i, j int = 1, 2
  k := 3
  c, python, java := true, false, "no!"

  fmt.Println(i, j, k, c, python, java)
}
```

## 2.5 Basic types

```
bool

string

int  int8  int16  int32  int64
uint uint8 uint16 uint32 uint64 uintptr

byte // alias for uint8

rune // alias for int32
     // represents a Unicode code point

float32 float64

complex64 complex128
```

``` go
package main

import (
  "fmt"
  "math/cmplx"
)

var (
  ToBe   bool       = false
  MaxInt uint64     = 1<<64 - 1
  z      complex128 = cmplx.Sqrt(-5 + 12i)
)

func main() {
  const f = "%T(%v)\n"
  fmt.Printf(f, ToBe, ToBe)
  fmt.Printf(f, MaxInt, MaxInt)
  fmt.Printf(f, z, z)
}
```

## 2.6 Type conversion

* The expression T(v) converts the value v to the type T. Some numeric conversions:

``` go
var i int = 42
var f float64 = float64(i)
var u uint = uint(f)
```

Or, simply:

``` go
i := 42
f := float64(i)
u := uint(f)
```

## 2.7 Constants

* Constants are declared like variables, but with the `const` keyword. Constants can be character, string, boolean, or numeric values. Constants cannot be declared using the `:=` syntax. An untyped constant takes the type needed by its context.

``` go
const Pi = 3.14
const (
  StatusOK      = 200
  StatusCreated = 201
)
```

## 2.8 For Loop

* Go has only one looping construct, the `for` loop. The basic for `loop` looks as it does in C or Java, except that `()` are gone (they are not even optional) and the `{}` are required.

* For loop example

``` go
sum := 0
for i := 0; i < 10; i++ {
  sum += i
}
```

* For loop without pre/post statements

``` go
sum := 1
for ; sum < 1000; {
  sum += sum
}
```

* For loop as a `while` loop

``` go
sum := 1
for sum < 1000 {
  sum += sum
}
```

* Infinite loop

``` go
for {
  // do something in a loop forever
}
```

## 2.9 If statement

* The `if` statement looks as it does in C or Java, except that `()` are gone and the `{}` are required. Like `for`, the `if` statement can start with a short statement to execute before the condition.

``` go
if err := foo(); err != nil {
  panic(err)
}
```

* Variables declared by the statement are only in scope until the end of the `if`. Variables declared inside an `if` short statement are also available inside any of the `else` blocks.

``` go
package main

import (
  "fmt"
  "math"
)

func pow(x, n, lim float64) float64 {
  if v := math.Pow(x, n); v < lim {
    return v
  } else {
    fmt.Println("%g >= %g\n", v, lim)
  }

  // can't use v here though
  return lim
}

func main() {
  fmt.Println(
    pow(3, 2, 10),
    pow(3, 3, 20)
  )
}
```

## 2.10 Exercise: Loops and Functions

* Implement the square root function using Newton's method.

``` go
package main

import (
  "fmt"
  "math"
)

func Sqrt(x float64) {
  // nested function to generate an approximation
  approximation := func(z, x float64) float64 {
    return z - ((z * z) - x) / (2 * z)
  }

  z := 1.0
  for i := 0; i< 10; i++ {
    z = approximation(z, x)
  }

  return z
}

func main() {
  for i := 1.0; i< 11.0; i++ {
    fmt.Println("%d: %f vs %f\n", int(i), Sqrt(i), math.Sqrt(i))
  }
}
```
