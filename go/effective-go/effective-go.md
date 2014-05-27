# [Effective Go](http://golang.org/doc/effective_go.html)

## Names

### Package names

* By convention, packages are given lower case, single-word names; there should be no need for underscores on mixedCaps.

* Another convention is that the package name is the base name of its source directory; the package in `src/pkg/encoding/base64` is imported as "encoding/base64" but has name `base64`, not `encoding_base64` and not `encodingBase64`.

* Because imported entities are always addressed with their package name, `bufio.Reader` does not conflict with `io.Reader`. Similarly, the function to make new instances of `ring.Ring`—which is the definition of a constructor in Go—would normally be called `NewRing`, but since `Ring` is the only type exported by the package, and since the package is called `ring`, it's called just `New`, which clients of the package see as `ring.New`.

### Getter/Setter

* If you have a field called `owner` (lower case, unexported), the`getter` method should be called `Owner` (upper case, exported), not `GetOwner`.

* The `setter` function, if needed, will likely be called `SetOwner`.

``` go
owner := obj.Owner()
if owner != user {
  obj.SetOwner(user)
}
```

### Interface names

* By convention, one-method interfaces are named by the method name plus an `-er` suffix or similar modification to construct an agent noun: `Reader`, `Writer`, `Formatter`, `CloseNotifier` etc.

* Call your string-converter method `String` not `ToString`.

### MixedCaps

* The convention in Go is to use `MixedCaps` or `mixedCaps` rather than underscore to write multiword names.

## Control structures

### If

* Since `if` and `switch` accept an initialization statement, it's common to see one used to set up a local variable.
  ``` go
  if err := file.Chmod(0644); err != nil {
    log.Print(err)
    return err
  }
  ```

### For

* There are three forms of `for`:
  ``` go
  // Like a C for
  for init; condition; post { }

  // Like a C While
  for condition { }

  // Like a C for(;;)
  for { }
  ```

### Switch

* There is no automatic fall through, but cases can be presented in comma-separated lists.

  ``` go
  func shouldEscape(c byte) bool {
    switch c {
    case ' ', '?', '&', '=', '#', '+', '%':
      return true
    }
    return false
  }
  ```

* Compare byte slices.

  ``` go
  // Compare returns an integer comparing the two byte slices,
  // lexicographically.
  // The result will be 0 if a == b, -1 if a < b, and +1 if a > b
  func Compare(a, b []byte) int {
      for i := 0; i < len(a) && i < len(b); i++ {
          switch {
          case a[i] > b[i]:
              return 1
          case a[i] < b[i]:
              return -1
          }
      }
      switch {
      case len(a) > len(b):
          return 1
      case len(a) < len(b):
          return -1
      }
      return 0
  }
  ```

### Type switch

* A switch can also be used to discover the dynamic type of an interface variable. Such a `type switch` uses the syntax of a type assertion with the keyword `type` inside the parentheses. If the switch declares a variable in the expression, the variable will have the corresponding type in each clause. It's also idiomatic to reuse the name in such cases, in effect declaring a new variable with the same name but a different type in each case.

  ``` go
  var t interface{}
  t = functionOfSomeType()
  switch t := t.(type) {
  default:
    fmt.Printf("unexpected type %T", t)  // %T prints whatever type t has
  case bool:
    fmt.Printf("boolean %t\n", t)
  case int:
    fmt.Printf("int %d\n", t)
  case *bool:
    fmt.Printf("pointer to boolean %t\n", *t)
  case *int:
    fmt.Printf("pointer to int %d\n", *t)
  }
  ```

## Functions

### Defer

* Go's `defer` statement schedules a function call (the deferred function) to be run immediately before the function executing the `defer` returns. It's an unusual but effective way to deal with situations such as resources that must be released regardless of which path a function takes to return. The canonical examples are unlocking a mutex or closing a file.
  ``` go
  // Contents returns the file's contents as a string.
  func Contents(filename string) (string, error) {
    f, err := os.Open(filename)
    if err != nil {
      return "", err
    }
    defer f.Close()  // f.Close will run when we're finished.

    var result []byte
    buf := make([]byte, 100)
    for {
      n, err := f.Read(buf[0:])
      result = append(result, buf[0:n]...)
      if err != nil {
        if err == io.EOF {
          break
        }
        return "", err  // f will be closed if we return here.
      }
    }
    return string(result), nil  // f will be closed if we return here.
  }
  ```

* We can do better by exploiting the fact that arguments to deferred functions are evaluated when the defer executes. The tracing routine can set up the argument to the untracing routine. This example:
  ``` go
  func trace(s string) string {
      fmt.Println("entering:", s)
      return s
  }

  func un(s string) {
      fmt.Println("leaving:", s)
  }

  func a() {
      defer un(trace("a"))
      fmt.Println("in a")
  }

  func b() {
      defer un(trace("b"))
      fmt.Println("in b")
      a()
  }

  func main() {
      b()
  }
  ```

  prints

  ```
  entering: b
  in b
  entering: a
  in a
  leaving: a
  leaving: b
  ```

## Data

### Allocation with `new`

* `new` is a built-in function that allocates memory, but unlike its namesakes in some other languages it does not _initialize_ the memory, it only _zeros_ it.

* That is, `new(T)` allocates zeroed storage for a new item of type T and returns its address, a value of type `*T`. In Go terminology, it returns a pointer to a newly allocated zero value of type `T`.

* Since the memory returned by `new` is zeroed, it's helpful to arrange when designing your data structures that the zero value of each type can be used without further initialization. This means a user of the data structure can create one with `new` and get right to work.

* For example, the documentation for `bytes.Buffer` states that "the zero value for Buffer is an empty buffer ready to use." Similarly, `snc.Mutex` does not have an explicit constructor or `Init` method. Instead, the zero value for a `sync.Mutex` is defined to be an unlocked mutex. The zero-value-is-usefull property works transitively:
  ``` go
  type SyncedBuffer struct {
    lock sync.Mutex
    buffer bytes.Buffer
  }
  ```

  Values of type `SyncedBuffer` are also ready to use immediately upon allocation or just declaration.
  ``` go
  p := new(SyncedBuffer) // type *SyncedBuffer
  var v SyncedBuffer     // type SyncedBuffer
  ```

### Constructors and composite literals

* Sometimes the zero value isn't good enough and an initializing constructor is necessary, as in this example derived from package `os`.
  ``` go
  func NewFile(fd int, name string) *File {
    if fd < 0 {
      return nil
    }
    f := new(File)
    f.fd = fd
    f.name = name
    f.dirinfo = nil
    f.nepipe = 0
    return f
  }
  ```

  We can simplify it using a `composite literals`, which is an expression that creates a new instance each time it is evaluated.
  ``` go
  func NewFile(fd int, name string) *File {
    if fd < 0 {
      return nil
    }
    f := File{fd, name, nil, 0}
    return &f
  }
  ```

  Note that, unlike in C, it's perfectly OK to return the address of a local variable; the storage associated with the variable survives after the function returns. In fact, take the address of a composite literal allocates a fresh instance each time it is evaluated, so we can combine these last two lines:
  ```go
  return &File{fd, name, nil, 0}
  ```

  The fields of a composite literal are laid out in order and must be present. However, by labeling the elements as `field: value` pairs, the initializers can appear in any order, with the missing ones left as their respective zero values. Thus we could say:
  ``` go
  return &File{fd: fd, name: name}
  ```

  As a limiting case, if a composite literal contains no fields at all, it creates a zero value for the type. The expressions `new(File)` and `&File{}` are equivalent.

### Allocation with `make`

* The built-in function `make(T, args)` servers a purpose different from `new(T)`. It creates `slices`, `maps` and `channels` only, and it returns an initalized (not zeroed) value of type `T` (not `*T`).

* The reason for the distinction is that these three types represent, under the covers, references to data structures that must be initialized before use. A slice, for example, is a three-item descriptor containing a pointer to the data (inside the array), the length, and the capacity, and until those items are initlaized, the slice is `nil`.

* For `slices`, `maps`, and `channels`, `make` initializes the internal data structure and prepares the value for use.

* These examples illustrate the difference between `new` and `make`.
  ``` go
  var p *[]int = new([]int)       // allocates slice structure; *p == nil; rarely useful
  var v  []int = make([]int, 100) //  the slice v now refers to a new array of 100 ints

  // Unnecessarily complex:
  var p *[]int = new([]int)
  *p = make([]int, 100, 100)

  // Idiomatic:
  v := make([]int, 100)
  ```

* Remember that `make` applies only to `maps`, `slices` and `channels` and does not return a pointer. To obtain an explicit pointer allocate with `new` or take the address of a variable explicitly.

### Arrays

* There are major differences between the ways arrays work in Go and C. In Go,
  - Arrays are values. Assigning one array to another copies all the elements.
  - In particular, if you pass an array to a function, it will receive a copy of the array, not a pointer to it.
  - The size of an array is part of its type. Type types `[10]int` and `[20]int` are distinct.

### Slices

* Slices hold references to an underlying array. If a function takes a slice argument, changes it makes to the elements of the slice will be visible to the caller, analogous to passing a pointer to the underlying array.

* The length of a slice may be changed as long as it still fits within the limits of the underlying array; just assign it to a slice of itself. The capacity of a slice, accessible by the built-in function `cap`, reports the maximum length the slice may assume. Here is a function to append data to a slice. If the data exceeds the capacity, the slice is reallocated. The resulting slice is returned.
  ``` go
  func Append(slice, data []byte) []byte {
    l := len(slice)
    if l + len(data) > cap(slice) { // reallocate
      // Allocate double what's needed, for future growth.
      newSlice := make([]byte, (l + len(data)) * 2)
      // The copy function is predeclared and works for any slice type.
      copy(newSlice, slice)
      slice = newSlice
    }
    slice = slice[0:l+len(data)]
    for i, c := range data {
      slice[l + i] = c
    }
    return slice
  }
  ```

* The idea of appending to a slice is so useful it's captured by the `append` built-in function.

### Maps

* The key can be of any type for which the equality operator is defined. If you pass a map to a function that changes the contents of the map, the changes will be visible in the caller.

* An attempt to fetch a map value with a key that is not present in the map will return the zero value for the type of the entries in the map.

* Sometimes you need to distinguish a missing entry from a zero values.
  ``` go
  var seconds int
  var ok bool
  seconds, ok = timeZone[tz]
  ```

  This is called the "comma OK" idiom.
  ``` go
  func offset(tz string) int {
    if seconds, ok := timeZone[tz]; ok {
      return seconds
    }
    log.Println("unknown time zone:", tz)
    return 0
  }
  ```

* To delete a map entry, use the `delete` built-in function, whose arguments are the map and the key to be deleted. It's safe to do this even if the key is already absent from the map.

### [Printing](http://golang.org/doc/effective_go.html#printing)

### Append

* What `append` does is append the elements to the end of the slice and return the result.

* But how to append a slice to a slice? Use `...` at the call site.
  ``` go
  x := []int{1, 2, 3}
  y := []int{4, 5, 6}
  x = append(x, y...)
  fmt.Println(x)
  ```
