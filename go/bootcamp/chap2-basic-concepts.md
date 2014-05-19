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

``` go
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

## 2.11 Structs

* A `struct` is a collection of fields/properties, you can consider a struct to be a light class that __doesn't support inheritance__.

* No need to define `getters` and `setters` on struct fields, they can be accessed automatically. __NOTE__: only exported fields can be accessed from outside of the package.

* A struct literal denotes a newly allocated struct value by listing the values of its fields.

* The special prefix `&` constructs a pointer to a newly allocated struct.

``` go
package main

import (
  "fmt"
  "time"
)

type Bootcamp struct {
  // Latitude of the event
  Lat float64
  // Longitude of the event
  Lon float64
  // Date of the event
  Date time.Time
}

func main() {
  fmt.Println(Bootcamp{Lat: 34.012836,
            Lon: -118.495338,
            Date: time.Now()})
}
```

Declaration of struct literals:

``` go
package main

import "fmt"

type Vertex struct {
  X, Y int
}

var {
  p = Vertex{1, 2}  // has type Vertex
  q = &Vertex{1, 2} // has type *Vertex
  r = Vertex{X: 1}  // Y:0 is implicit
  s = Vertex{}      // X:0 and Y:0
}

func main() {
  fmt.Println(p, q, r, s)
}
```

## 2.12 Pointers

* Go has pointers, but no pointer arithmetic. Struct fileds can be accessed through a struct pointer. The indirection through the pointer is transparent.

``` go
package main

import "fmt"

type Vertex struct {
  X int
  Y int
}

func main() {
  p := Vertex{1, 2}
  q := &p
  q.X = 1e9
  fmt.Println(p)
}
```

* __NOTE__: by default Go passes arguments by values(copying the arguments), if you want to pass the arguments by reference, you need to pass pointers.

* Just like `C`: To get the pointer of a value, use the `&` symbol in front of the value, to dereference a pointer, use the `*` symbol.

* Methods are often defined on pointers and not values(although they can be defined on both), so you will often store a pointer in a variable as in the example below:

``` go
client := &http.Client()
resp, err := client.Get("http://gobootcamp.com")
```

## 2.13 Initializing

* Use `new` expression to allocate a zeroed value of the requested type, and will get a pointer to it.

``` go
x := new(int)
```

``` go
package main

import "fmt"

type Bootcamp struct {
  Lat float64
  Lon float64
}

func main() {
  x := new(Bootcamp)
  y := &Bootcamp{}
    fmt.Println(*x == *y)
}
```

### 2.13.1 Resources

* [Allocation with `new` - effective Go](http://golang.org/doc/effective_go.html#allocation_new)
* [Composite Literals - effective Go](http://golang.org/doc/effective_go.html#composite_literals)
* [Allocation with `make` - effective Go](http://golang.org/doc/effective_go.html#allocation_make)

## 2.14 Arrays

* The type `[n] T` is an array of n values of type T. `var a [10]int` declares a variable `a` as an array of ten integers.

* An array's length is part of its type, so arrays cannot be resized. This seems limiting, but Go provides a convenient way of working with arrays.

``` go
package main

import "fmt"

func main() {
  var a [2]string
  a[0] = "Hello"
  a[1] = "World"
  fmt.Println(a[0], a[1])
  fmt.Println(a)
}
```

* You can also set the array entries as you declare the array:

``` go
package main

import "fmt"

func main() {
  a := [2]string{"hello", "world!"}
  fmt.Printf("%q", a)
}
```

* Finally, you can use an __ellipsis__ `...` to use an implicit length when you pass the values:

``` go
package main

import "fmt"

func main() {
  a := [...]string{"hello", "world!"}
  fmt.Printf("%q", a)
}
```

### 2.14.1 Printing arrays

* Note how we used the `fmt` package using `Printf` and use the `%q` "verb" to print each element quoted. If we had used `Println` or the `%s` "verb", we would have a different result.

``` go
package main

import "fmt"

func main() {
  a := [2]string{"hello", "world!"}
  fmt.Println(a)
  // [hello world!]
  fmt.Printf("%s\n", a)
  // [hello world!]
  fmt.Printf("%q\n", a)
  // ["hello" "world!"]
}
```

### 2.14.2 Multi-dimensional arrays

* You can also create multi-dimensional arrays:

``` go
package main

import "fmt"

func main() {
  var a [2][3]string
  for i := 0; i < 2; i++ {
    for j := 0; j < 3; j++ {
      a[i][j] = fmt.Sprintf("row %d - column %d", i+ 1, j + 1)
    }
  }
  fmt.Printf("%q", a)
  // [["row 1 - column 1" "row 1 - column 2" "row 1 - column 3"]
  //  ["row 2 - column 1" "row 2 - column 2" "row 2 - column 3"]]
}
```

## 2.15 Slices

* Slices wrap arrays to give a more general, powerful, and convenient interface to sequences of data. Most array programming in Go is done with slices rather than simple arrays.

* Slices hold references to an underlying array.

* A slice points to an array of values and also includes a length. Slices can be resized since they are just a wrapper on top of another data structure.

* `[]T` is a slice with elements of type `T`.

``` go
package main

import "fmt"

func main() {
  p := []int{2, 3, 5, 7, 11, 13}
  fmt.Println(p)
  // [2 3 5 7 11 13]
}
```

### 2.15.1 Slicing a slice

* Slices can be re-sliced, creating a new slice value that points to the same array.

* The expression `s[lo:ho]` evaluates to a slice of the elements from lo through hi-1, inclusive.

* __NOTE__: lo and hi would be integers representing indexes.

``` go
package main

import "fmt"

func main() {
  mySlice := []int{2, 3, 5, 7, 11, 13}
  fmt.Println(mySlice)
  // [2 3 5 7 11 13]
  fmt.Println(mySlice[1:4])
  // [3 5 7]

  // missing low index implies 0
  fmt.Println(mySlice[:3])
  // [2, 3, 5]

  // missing high index implies len(s)
  fmt.Println(mySlice[4:])
  // [11, 13]
}
```

### 2.15.2 Making slices

* Besides creating slices by passing the values right away(array literal), you can also use `make`. You create an empty slice of a specific length and then populate each entry:

``` go
package main

import "fmt"

func main() {
  cities := make([]string, 3)
  cities[0] = "Santa Monica"
  cities[1] = "Venice"
  cities[2] = "Los Angeles"
  fmt.Printf("%q", cities)
  // ["Santan Monica" "Venice" "Los Angeles"]
}
```

* It works by allocating a zeroed array an returning a slice that refers to that array.

## 2.15.3 Appending to a slice

* Note however, that you would get a runtime error if you were to do that:

``` go
cities := []string{}
cities[0] = "Santa Monica"
```

A slice is seating on top of an array, in this case, the array is empty and the slice can't set a value in the referred array.

* You can use `append` function to do that:

``` go
package main

import "fmt"

func main() {
  cities := []string{}
  cities = append(cities, "San Diego")
  fmt.Println(cities)
  // [San Diego]
}
```

* You can append more than one entry to a slice:

``` go
package main

import "fmt"

func main() {
  cities := []string{}
  cities = append(cities, "San Diego", "Mountain View")
  fmt.Printf("%q", cities)
  // ["San Diego" "Mountain View"]
}
```

* Also you can append a slice to another using an ellipsis `...`:

``` go
package main

import "fmt"

func main() {
  cities := []string{"San Diego", "Mountain View"}
  otherCities := []stirng{"San Diego", "Venice"}
  cities = append(cities, otherCities...)
  fmt.Printf("%q", cities)
  ["San Diego" "Mountain View" "San Diego" "Venice"]
}
```

* The ellipsis is a built-in feature of the language that means that the element is a collection. Using the ellipsis (`...`) after our slice, we indicate that we want to append each element of our slice.

### 2.15.4 Length

* You can check the length of a slice by using `len`:

``` go
package main

import "fmt"

func main() {
  cities := []string{"Santa Monica", "San Diego", "San Francisco"}
  fmt.Println(len(cities))
  // 3
  countries := make([]string, 42)
  fmt.Println(len(countries))
  // 42
}
```

### 2.15.5 Nil slices

* The zero value of a slice is nil. A nil slice has a length and capacity of 0.

``` go
package main

import "fmt"

func main() {
  var z []int
  fmt.Println(z, len(z), cap(z))
  // [] 0 0
  if z == nil {
    fmt.Println("nil!")
  }
  // nil!
}
```

### 2.15.6 Resources

* [Go slices: usage and internals](http://blog.golang.org/go-slices-usage-and-internals)
* [Effective Go - slices](http://golang.org/doc/effective_go.html#slices)
* [Append function documentation](http://golang.org/pkg/builtin/#append)
* [Slice tricks](https://code.google.com/p/go-wiki/wiki/SliceTricks)
* [Effective Go - two-demensional slices](http://golang.org/doc/effective_go.html#two_dimensional_slices)
* [Go by example - slices](https://gobyexample.com/slices)

## 2.16 Range

* `range` is like `each_with_index` in Ruby. With `range`, you can iterate over all the elements of a `slice` or a `map`. __NOTE__: idx comes first!

``` go
package main

import "fmt"

var pow = []int{1, 2, 4, 8, 16, 32, 64, 128}

func main() {
  for i, v := range pow {
    fmt.Printf("2 ** %d = %d\n", i, v)
  }
}
```

* You can skip the index or value by assigning to `_`. If you only want the index, drop the ", value" entirely.

``` go
package main

import "fmt"

func main() {
  pow := make([]int, 10)
  for i := range pow {
    pow[i] = 1 << uint(i)
  }
  for _, value := range pow {
    fmt.Printf("%d\n", value)
  }
}
```

### 2.16.1 Break & continue

* You can stop the iteration anytime by using `break`:

``` go
package main

import "fmt"

func main() {
  pow := make([]int, 10)
  for i := range pow {
    pow[i] = 1 << uint(i)
    if pow[i] > 16 {
      break
    }
  }
  fmt.Println(pow)
  // [ 1 2 4 8 16 0 0 0 0 0]
}
```

* You can also skip an iteration by using `continue`:

``` go
package main

import "fmt"

func main() {
  pow := make([]int, 10)
  for i := range pow {
    if i % 2 == 0 {
      continue
    }
    pow[i] = 1 << uint(i)
  }
  fmt.Println(pow)
  // [0 2 0 8 0 32 0 128 0 512]
}
```

### 2.16.2 Range and maps

* Range can also be used on `maps`, in that case, the first parameter isn't an incremental integer but the map key:

``` go
package main

import "fmt"

func main() {
  cities := map[string]int{
    "New York": 8336697,
    "Los Angeles": 3857799,
    "Chicago": 2714856
  }
  for key, value := range cities {
    fmt.Println("%s has %d inhabitants\n", key, value)
  }
}
```

## 2.17 Exercise: Slice

``` go
package main

import "code.google.com/p/go-tour/pic"

func Pic(dx, dy int) [][]uint8 {
  img := make([][]uint8, dy)
  for y := range img {
    img[y] = make([]uint8, dx)
    for x := 0; x < dx; x ++ {
      img[y][x] = uint8(x * y)
    }
  }
  return img
}

func main() {
  pic.Show(Pic)
}
```

## 2.18 Maps

* Maps are somewhat similar to what other languages call "dictionaries" or "hashes".

``` go
package main

import "fmt"

func main() {
  celebs := map[string]int {
    "Nicolas Cage": 50,
    "Selena Gomez": 21,
    "Jude Law":     41,
    "Scarlett Johansson": 29,
  }
  fmt.Printf("%#v", celebs)
}
```

* When not using map literals like above, maps must be created with `make` (not new) before use. The nil map is empty and cannot be assigned to.

``` go
package main

import "fmt"

type Vertex struct {
  Lat, Lon float64
}

var m map[string]Vertex

func main() {
  m = make(map[string]Vetex)
  m["Bell Labs"] = Vertex{40.68433, -74.39967}
  fmt.Println(m["Bell Labs"])
}
```

* When using map literals, if the top-level type is just a type name, you can omit it from the elements of the literal.

``` go
package main

import "fmt"

type Vertex struct {
  Lat, Lon float64
}

var m = map[string]Vertex {
  "Bell Labs": {40.68433, -74.39967},
  // same as "Bell Labs": Vertex{40.68433, -74.39967}
  "Google": {37.42202, -122.08408},
}

func main() {
  fmt.Println(m)
}
```

### 2.18.1 Mutating Maps

* Insert or update an element in map m:
``` go
m[key] = elem
```

* Retrieve an element:
``` go
elem = m[key]
```

* Delete an element:
``` go
delete(m, key)
```

* Test that a key is present with a two-value assignment. If key is in m, ok is true. If not, ok is false and elem is the zero value for the map's element type.
``` go
elem, ok = m[key]
```

* Similarly, when reading from a map if the key is not present, the result is the zero value for the map's element type.

## 2.19 Exercise: Map

``` go
package main

import (
  "code.google.com/p/go-tour/wc"
  "strings"
)

func WordCount(s string) map[string]int {
  counter = make(map[string]int)

  for _, word := range strings.Fields(s) {
    counter[word] += 1
  }

  return counter
}

func main() {
  wc.Test(WordCount)
}
```
