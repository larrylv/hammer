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

## Initialization

* Complex structures can be built during intialization and the ordering issues among intialized objects, even among different packages, are handled correctly.

### Constants

* Constants are created at compile time, even when defined as locals in functions, and can be only numbers, charactres (runes), strings or booleans.

* Because of the compile-time restriction, the expressions that define them must be constant expressions, evaluated by the compiler. For instance, `1<<3` is a constant expression, while `math.Sin(math.Pi/4)` is not because the function call to `math.Sin` needs to happen at run time.

* In Go, enumerated constants are created using the `iota` enumerator. Since `iota` can be part of an expression and expressions can be implicityly repeated, it is easy to build intricate sets.

  ``` go
  type ByteSize float64

  const (
    _           = iota  // ignore first value by assigning to blank identifier
    KB ByteSize = 1 << (10 * iota)
    MB
    GB
    TB
    PB
    EB
    ZB
    YB
  )
  ```

  The ability to attach a method such as `String` to any user-defined type makes it possible for arbitrary values to format themselves automatically for printing. Although you'll see it most often applied to structs, this technique is also usefull for scalar types such as floating-point types like `ByteSize`.

  ``` go
  func (b ByteSize) String() string {
    switch {
    case b >= YB:
      return fmt.Sprintf("%.2fYB", b/YB)
    case b >= ZB:
      return fmt.Sprintf("%.2fZB", b/ZB)
    case b >= EB:
      return fmt.Sprintf("%.2fEB", b/EB)
    case b >= PB:
      return fmt.Sprintf("%.2fPB", b/PB)
    case b >= TB:
      return fmt.Sprintf("%.2fTB", b/TB)
    case b >= GB:
      return fmt.Sprintf("%.2fGB", b/GB)
    case b >= MB:
      return fmt.Sprintf("%.2fMB", b/MB)
    case b >= KB:
      return fmt.Sprintf("%.2fKB", b/KB)
    }
    return fmt.Sprintf("%.2fB" b)
  }
  ```

  The expression YB prints as 1.00YB, while ByteSize(1e13) prints as 9.09TB.

  __NOTE:__ The use here of `Sprintf` to implement `ByteSize's String` method is safe (avoids recurring indefinitely) not because of a conversion but because it calls `Sprintf` with `%f`, which is not a string format: `Sprintf` will only call the `String` method when it wants a string, and `%f` wants a floating-point value.

### Variables

* Variables can be initialized just like constants but the initializer can be a general expression computerd at run time.

  ``` go
  var (
    home   = os.Getenv("HOME")
    user   = os.Getenv("USER")
    gopath = os.Getenv("GOPATH")
  )
  ```

### The init function

* Each source file can define its own `init` function to setup whatever state is required.

* `init` is called after all the variable declarations in the package have evaluated their initializers, and those are evaluated only after all the imported packages have been initlaized.

* Besides initializations that cannot be expressed as declarations, a common use of `init` functions is to verify or repair correctness of the program state before real execution begins.

  ``` go
  func init() {
    if user == "" {
      log.Fatal("$USER not set")
    }
    if home == "" {
      home = "/home/" + user
    }
    if gopath == "" {
      gopath = home + "/go"
    }
    // gopath may be overridden by --gopath flag on command line.
    flag.StringVar(&gopath, "gopath", gopath, "override default GOPATH")
  }
  ```

## Methods

### Pointers vs. Values

* Methods can be defined for any named type (except a pointer or an interface); the receiver does not have to be a struct.

* In the previous example of `Append` method for `[]byte`, we need to return the updated slice. We can eliminate the clumsiness by redefining the method to take a `pointer fo a ByteSlice` as its receiver, so the method can overwrite the caller's slice:

  ``` go
  type ByteSlice []byte

  func (p *ByteSlice) Append(data []byte) {
    slice := *p
    // Body as above, without the return.
    *p = slice
  }
  ```

  The rule about pointers vs. values for receivers is that value methods can be invoked on pointers and values, but pointer methods can only be invoked on pointers. This is because pointer methods can modify the receiver; invoke them on a copy of the value would cause those modifications to be discarded.

## Interfaces and other types

### Interfaces

* Interfaces in Go provide a way to specify the behavior of an object: if something can do this, then it can be used here.

* A type can implement multiple interfaces. For instance, a collection can be sorted by the routines in package `sort` if it implements `sort.Interface`, which contains `Len()`, `Less(i, j int) bool`, and `Swap(i, j int)`, and it could also have a custom formatter.

  ``` go
  type Sequence []int

  // Methods required by sort.Interface
  func (s Sequence) Len() int {
    return len(s)
  }
  func (s Sequence) Less(i, j int) bool {
    return s[i] < s[j]
  }
  func (s Sequence) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
  }

  // Method for printing - sorts the elements before printing.
  func (s Sequence) String() string {
    sort.Sort(s)
    str := "["
    for i, elem := range s {
      if i > 0 {
        str += " "
      }
      str += fmt.Sprint(elem)
    }
    return str + "]"
  }
  ```

### Conversions

* The `String` method of `Sequence` is recreating the work that `Sprint` already does for slices. We can share the effort if we convert the `Sequence` to a plain `[]int` before calling `Sprint`.

  ``` go
  func (s Sequence) String() string {
    sort.Sort(s)
    return fmt.Sprint([]int(s))
  }
  ```

* It's an idiom in Go pragrams to convert the type of an expression to access a different set of methods. As an example, we could use the existing type `sort.IntSlice` to reduce the entire example to this:

  ``` go
  type Sequence []int

  func (s Sequence) String() string {
    sort.IntSlice(s).Sort()
    return fmt.Sprint([]int(s))
  }
  ```

### Interface conversions and type assertions

* __Type Switches__ are a form of conversion: they take an interface and, for each case in the switch, in a sense convert it to the type of that case.

* Here's a simplified version of how the code under `fmt.Printf` turns a value into a string using a type switch. If it's already a string, we want the actual string value held by the interface, while if it has a `String` method we want the result of calling the method.

  ``` go
  type Stringer interface {
    String() string
  }

  var value interface{} // Value provided by caller.
  switch str := value.(type) {
  case string:
    return str
  case Stringer:
    return str.String()
  }
  ```

* __Type Assertion__: A type assertion takes an interface value and extracts from it a value of the specified explicit type.

  ``` go
  value.(typeName)
  ```

  And the result is a new value with the static type `typeName`. That type must either be concrete type held by the interface, or a second interface type that the value can be converted to.

  To extract the string we know is in the value, we could write:
  ``` go
  str := value.(string)
  ```

  But if it turns out that the value does not contain a string, the program will crash with a run-time error. Use the "comman, ok" idiom to test, safely, whether the value is a string:

  ``` go
  str, ok := value.(string)
  if ok {
    fmt.Printf("string value is %q\n", str)
  } else {
    fmt.Printf("value is not a string\n")
  }
  ```

  If the type assertion fails, `str` will still exist an be of type string, but it will have the zero value, an empty string.

* As an illustration of the capability, here's an `if-else` statement that's equivalent to the type switch that opened this section.

  ``` go
  if str, ok := value.(string); ok {
    return str
  } else if str, ok := value.(Stringer); ok {
    return str.String()
  }
  ```

### Interfaces and methods

* Since almost anything can have methods attached, almost anything can satisfy an interface. One illustrative example is in the `http` package, which defines the `Handler` interface. Any object that implements `Handler` can server HTTP requests.

  ``` go
  type Handler interface {
    serverHTTP(ResponseWriter, *Request)
  }
  ```

  `ResponseWriter` is itself an interface that provides access to the methods needed to return the response to the client. Those methods include the standard Write method, so an `http.ResponseWriter` can be used wherever an `io.Writer` can be used. `Request` is a struct containing a parsed representation of the request from the client.

  ``` go
  // Simple counter server.
  type Counter struct {
    n int
  }

  func (ctr *Counter) ServerHTTP(w http.ResponseWriter, req *http.Request) {
    ctr.n++
    fmt.Fprintf(w, "counter = %d\n", ctr.n)
  }
  ```

  Here's how to attach such a server to a node on the URL tree.
  ``` go
  import "net/http"

  cr := new(Counter)
  http.Handle("/counter", ctr)
  ```
* What if your program has some internal state that needs to be notified that a page has been visited? Tie a channel to the web page.

  ``` go
  // A channel that sends a notification on each visit.
  // (Probably want the channel to be buffered.)
  type Chan chan *http.Request

  func (ch Chan) ServerHTTP(w http.ResponseWriter, req *http.Request) {
    ch <- req
    fmt.Fprint(w, "notification sent")
  }
  ```
* Finally, let's say we wanted to present on `/args` the arguments used when invoking the server binary.

  ``` go
  func ArgServer() {
    fmt.Println(os.Args)
  }
  ```

  How do we turn that into an HTTP server? Since we can define a method for any type except pointers and interfaces, we can write a method for a function. The `http` package contains this code:

  ``` go
  // The HandlerFunc type is an adapter to allow the use of
  // ordinary functions as HTTP handlers. If f is a function
  // with the appropriate signature, HandlerFunc(f) is a
  // Handler object that calls f.
  type HandlerFunc func(ResponseWriter, *Request)

  // ServerHTTP calls f(c, req)
  func (f HandlerFunc) ServerHTTP(w ResponseWriter, req *Request) {
    f(w, req)
  }
  ```

  `HandlerFunc` is a type with a method, `ServerHTTP`, so values of that type can server HTTP requests. Look at the implementation of the method: the receiver is a function, `f`, and the method calls `f`. That may seem odd but it's not different from, say, the receiver being a channel and the method sending on the channel.

  To make `ArgServer` into an HTTP server, we first modify it to have the right signature.

  ``` go
  // Argument server.
  func ArgServer(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintln(w, os.Args)
  }
  ```

  `ArgServer` now has same signature as `HandlerFunc`, so it can be converted to that type to access its methods, just as we converted `Sequence` to `IntSlice` to access `IntSlice.Sort`. The code to set it up is concise:

  ``` go
  http.Handle("/args", http.HandleFunc(ArgServer))
  ```
* In this section, we have made an HTTP server from a struct, an integer, a channel, and a function, all because interfaces are just sets of methods, which can be defined for (almost) any type.

## The blank identifier

### The blank identifier in multiple assignment

``` go
if _, err := os.Stat(path); os.IsNotExist(err) {
  fmt.Printf("%s does not exist\n", path)
}
```

### Unused imports and variables

* It is an error to import a package or to declare a variable without using it. Unused imports bloat the program and slow compilation, while a variable that is initialized but not used is at least a wasted computation and perhaps indicative of a larger bug. When a program is under active development, however, unused imports and variables often arise and it can be annoying to delete them just to have the compilation proceed, only to have them be needed again later. The blank identifier provides a workaround.

  ``` go
  package main

  import (
      "fmt"
      "io"
      "log"
      "os"
  )

  var _ = fmt.Printf // For debugging; delete when done.
  var _ io.Reader    // For debugging; delete when done.

  func main() {
      fd, err := os.Open("test.go")
      if err != nil {
          log.Fatal(err)
      }
      // TODO: use fd.
      _ = fd
  }
  ```

### Import for side effect

* Sometimes it is useful to import a package only for its side effects, without any explicit use. For example, during its `init` function, the [net/http/pprof](http://golang.org/pkg/net/http/pprof/) package registers HTTP handlers that provide debugging information. It has an exported API, but most clients need only the handler registration and access the data through a web page.

* To import the package only for its side effects, rename the package to the blank identifier:

  ``` go
  import _ "net/http/pprof"
  ```

  This form of import makes clear that the package is being imported for its side effects.

### Interface checks

* A type need not declare explicitly that it implements an interface. Instead, a type implements the interface just by implementing the interface's methods. In practice, most interface conversions are static and therefor checked at compile time. Eg. passing an `*os.File` to a function expecting an `io.Reader` will not compile unless `*os.File` implements the `io.Reader` interface.

* Some interface checks do happen at run-time, though. One instance is in the `encoding/json` package, which defines a `Marshaler` interface. When the JSON encoder receives a value that implements that interface, the encoder invokes the value's marshaling method to convert it to JSON instead of doing the standard conversion. The encoder checks this property at run time with a `type assertion` like:

  ``` go
  m, ok := val.(json.Marshaler)
  ```

  If it's necessary only to ask whether a type implements an interface, without actually using the interface itself, perhaps as part of an error check, use the blank identifier to ignore the type-asserted value:

  ``` go
  if _, ok := val.(json.Marshaler); ok {
    fmt.Printf("value %v of type %T implements json.Marshaler\n", val, val)
  }
  ```

## Embedding

* Go does not provide the typical, type-driven notion of subclassing, but it does have the ability to "borrow" pieces of an implementation by embedding types within a struct or interface.

  ``` go
  type Reader interface {
    Read(p []byte) (n int, err error)
  }

  type Writer interface {
    Write(p []byte) (n int, err error)
  }
  ```

  ``` go
  //ReadWriter is the interface that combines the Reader and Writer interfaces.
  type ReadWriter interface {
    Reader
    Writer
  }
  ```

  It's a union of the embedded interfaces (which must be disjoint sets of methods). Only interfaces can be embedded within interfaces.

* The `bufio` package has two struct types, `bufio.Reader` and `bufio.Writer`, each of which of course implements the analogous interfaces from package `io`. And `bufio` also implements a buffered reader/writer, which it does by combining a reader and a writer into one struct using embedding: __it lists the types within the struct but does not give them field names__.

  ``` go
  // ReadWriter stores pointers to a Reader and a Writer.
  // It implements io.ReadWriter.
  type ReadWriter struct {
    *Reader  // *bufio.Reader
    *Writer  // *bufio.Writer
  }
  ```

  The methods of embedded types come along for free, which means that `bufio.ReadWriter` not only has the methods of `bufio.Reader` and `bufio.Writer`, it also satisfies all three interfaces: `io.Reader`, `io.Writer`, and `io.ReadWriter`.

  There's an important way in which embedding differs from subclassing. When we embed a type, the methods of that type become methods of the outer type, __but when they are invoked the receiver of the method is the inner type__, not the outer one.

* Embedding can also be a simple convenience. This example shows an embedded field alongside a regular, named field.

  ``` go
  type Job struct {
    Command string
    *log.Logger
  }
  ```

  The `Job` type now has the `Log`, `Logf` and other methods of `*log.Logger`. We could have given the `Logger` a field name, of course, but it's not necessary to do so. And now, once initialized, we can log to the `Job`:

  ``` go
  job.Log("starting now...")
  ```

  The Logger is a regular field of the Job struct, so we can initialize it in the usual way inside the constructor for `Job`, like this:

  ``` go
  func NewJob(command string, logger *log.Logger) *Job {
    return &Job{command, logger}
  }
  ```

  or with a composite literal,

  ``` go
  job := &Job{command, log.New(os.Stderr, "Job: ", log.Ldate)}
  ```

* If we need to refer to an embedded field directly, the type name of the field, ignoring the package qualifier, serves as a field name, as it did in the `Read` method of our `ReaderWriter` struct. Here, if we needed to access the `*log.Logger` of a Job variable job, we would write `job.Logger`, which would be useful if we wanted to refine the methods of Logger.

  ``` go
  func (job *Job) Logf(format string, args ...interface{}) {
    job.Logger.Logf("%q: %s", job.Command, fmt.Sprintf(format, args...))
  }
  ```

* Embedding types introduces the problem of name conflicts but the rules to resolve them are simple. First, a field or method X hides any other item X in a more deeply nested part of the type. If log.Logger contained a field or method called Command, the Command field of Job would dominate it.

* Second, if the same name appears at the same nesting level, it is usually an error; it would be erroneous to embed log.Logger if the Job struct contained another field or method called Logger. However, if the duplicate name is never mentioned in the program outside the type definition, it is OK. This qualification provides some protection against changes made to types embedded from outside; there is no problem if a field is added that conflicts with another field in another subtype if neither field is ever used.

## Concurrency

### Share by communicating

* One way to think about this model is to consider a typical single-threaded program running on one CPU. It has no need for synchronization primitives. Now run another such instance; it too needs no synchronization. Now let those two communicate; if the communication is the synchronizer, there's still no need for other synchronization. Unix pipelines, for example, fit this model perfectly. Although Go's approach to concurrency originates in Hoare's Communicating Sequential Processes (CSP), it can also be seen as a type-safe generalization of Unix pipes.

### Goroutines

* They're called goroutines because the existing terms—threads, coroutines, processes, and so on—convey inaccurate connotations. A goroutine has a simple model: it is a function executing concurrently with other goroutines in the same address space. It is lightweight, costing little more than the allocation of stack space. And the stacks start small, so they are cheap, and grow by allocating (and freeing) heap storage as required.

* Goroutines are multiplexed onto multiple OS threads so if one should block, such as while waiting for I/O, others continue to run. Their design hides many of the complexities of thread creation and management.

* Prefix a function or method call with the go keyword to run the call in a new goroutine. When the call completes, the goroutine exits, silently. (The effect is similar to the Unix shell's & notation for running a command in the background.)

  ``` go
  go list.Sort()  // run list.Sort concurrently; don't wait for it.
  ```

* A function literal can be handy in a goroutine invocation.

  ``` go
  func Announce(message string, delay time.Duration) {
    go func() {
      time.Sleep(delay)
      fmt.Println(message)
    }()  // Note the parentheses - must call the function.
  }
  ```

  In Go, function literals are closures: the implementation makes sure the variables refered to by the function survive as long as they are active.

### Channels

* Like maps, `channels` are allocated with `make`, and the resuling value acts as a reference to an underlying data structure. If an optional integer parameter is provided, it sets the buffer size for the channel. The default is zero, for an unbuffered or synchronous channel.

  ``` go
  ci := make(chan int)             // unbuffered channel of integers
  cj := make(chan int, 0)          // unbuffered channel of integers
  cs := make(chan *os.File, 100)   // buffered channel of pointers to Files
  ```

* Unbuffered channels combine communication-the exchange of a value-with synchronization-guranteeing that two calculations (goroutines) are in a known stage.

* A channel can allow the launching goroutine to wait for the sort to complete.

  ``` go
  c := make(chan int)  // Allocate a channel.
  // Start the sort in a goroutine; when it completes, signal on the channel.
  go func() {
    list.Sort()
    c <- 1  // Send a signal; value does not matter.
  }()
  doSomethingForAWhile()
  <-c  // Wait for sort to finish; discard sent value.
  ```

* Receivers always block until there is data to receive. If the channel is unbuffered, the sender blocks until the receiver has received the value. If the channel has a buffer, the sender blocks only until the value has been copied to the buffer; if the buffer is full, this means waiting until some receiver has retrieved a value.

* A buffered channel can be used to limit throughput.

  ``` go
  var sem = make(chan int, MaxOutstanding)

  func handle(r *Request) {
    <-sem       // Wait for active queue to drain.
    process(r)  // May take a long time.
    sem <- 1    // Done; enable next request to run.
  }

  func init() {
    for i := 0; i < MaxOutstanding; i++ {
      sem <- 1
    }
  }

  func Serve(queue chan *Request) {
    for {
      req := <-queue
      go handle(req)  // Don't wait for handle to finish.
    }
  }
  ```

  __NOTE__: Because data synchronization occurs on a receive from a channel (that is, the send "happens before" the receive), acquisition of the `sem` must be on a channel _receive_, not _send_.

  This design has a problem, though: `Serve` creates a new goroutine for every incoming request, even though only MaxOutstanding of them can run at any moment. As a result, the program can consume unlimited resources if the requests come in too fast. We can address that deficiency by changing `Serve` to gate the creation of the goroutines.

  Here's an obvious solution, but beware it has a bug we'll fix subsequently:

  ``` go
  func Server(queue chan *Request) {
    for req := range queue {
      <-sem
      go func() {
        process(req)  // Buggy; see explanation below.
        sem <- 1
      }()
    }
  }
  ```

  The bug is that in a Go for loop, the loop variable is reused for each iteration, so the `req` variable is shared across all goroutines. That's not what we want. We need to make sure that req is unique for each goroutine. Here's one way to do that, passing the value of req as an argument to the closure in the goroutine:

  ``` go
  func Server(queue chan *Request) {
    for req := range queue {
      <-sem
      go func(req *Request) {
        process(req)
        sem <- 1
      }(req)
    }
  }
  ```

  Another solution is just to create a new variable with the same name, as in this example:

  ``` go
  func Server(queue chan *Requeset) {
    for req := range queue {
      <-sem
      req := req    // Create new instance of req for the gorouine.
      go func() {
        process(req)
        sem <- 1
      }()
    }
  }
  ```

  With `req := req`, you get a fresh version of the variable with the same name, deliberately shadowing the loop variable locally but unique to each goroutine.

* Another approach that manages resources well is to start a fixed number of `handle` goroutines all reading from the request channel. The number of goroutines limits the number of simultaneous calls to `process`. This `Server` function also accepts a channel on which it will be told to exit; after launching the goroutines it blocks receiving from the channel.

  ``` go
  func handle(queue chan *Request) {
    for r := range queue {
      process(r)
    }
  }

  func Server(clientRequests chan *Request, quit chan bool) {
    // Start handlers
    for i := 0; i < MaxOutStanding; i++ {
      go handle(clientRequests)
    }
    <-quit   // Wait to be told to exit.
  }
  ```

## Channels of channels

* In the example in the previous section, handle was an idealized handler for a request but we didn't define the type it was handling. If that type includes a channel on which to reply, each client can provide its own path for the answer. Here's a schematic definition of type `Request`.

  ``` go
  type Request struct {
    args       []int
    f          func([]int) int
    resultChan chan int
  }
  ```

  The client provides a function and its arguments, as well as a channel inside the request object on which to receive the answer.

  ``` go
  func sum(a []int) (s int) {
    for _, v := range a {
      s += v
    }
    return
  }

  request := &Request{[]int{3, 4, 5}, sum, make(chan int)}
  // Send request
  clientRequests <- request
  // Wait for response.
  fmt.Printf("answer: %d\n", <-request.resultChan)
  ```

  On the server side, the handler function is the only thing that changes.

  ``` go
  func handle(queue chan *Request) {
    for req := range queue {
      req.resultChan <- req.f(req.args)
    }
  }
  ```
* There's clearly a lot more to do to make it realistic, but this code is a framework for a rate-limited, parallel, non-blocking RPC system, and there's not a mutex in sight.

### Parallelization

* Another approach of these ideas is to parallelize a calculation across multiple CPU cores. If the calculation can be broken into separate pieces that can execute independently, it can be parallelized, with a channel to signal when each piece completes.

* Let's say we have an expensive operation to perform on a vector of items, and that the value of the operation on each item is independent, as in this idealized example.

  ``` go
  type Vector []float64

  // Apply the operation to v[i], v[i+1] ... up to v[n-1]
  func (v Vector) DoSome(i, n int, u Vector, c chan int) {
    for ; i < n; i++ {
      v[i] += u.Op(v[i])
    }
    c <- 1  // Signal that this piece is done
  }
  ```

  We launch the pieces independently in a loop, one per CPU. They can complete in any order but it doesn't matter; we just count the completion signals by draining the channel after launching all the goroutines.

  ``` go
  const NCPU = 4  // number of CPU cores

  func (v Vector) DoAll(u Vector) {
    c := make(chan int, NCPU)   // Buffering optional but sensible.
    for i := 0; i < NCPU; i++ {
      go v.DoSome(i*len(v)/NCPU, (i+1)*len(v)/NCPU, u, c)
    }

    // Drain the channel.
    for i := 0; i < NCPU; i++ {
      <-c  // wait for one task to complete
    }
    // All done.
  }
  ```

  __The current implementation of the Go runtime will not parallelize this code by default.__ It dedicates only a single core to user-level processing.  An arbitrary number of goroutines can be blocked in system calls, but by default only one can be executing user-level code at any time. It should be smarter and one day it will be smarter, but until it is if you want CPU parallelism you must tell the run-time how many goroutines you want executing code simultaneously.

  There are two related ways to do this. Either run your job with environment variable `GOMAXPROCS` set to the number of cores to use or import the runtime package and call `runtime.GOMAXPROCS(NCPU)`. A helpful value might be `runtime.NumCPU()`, which reports the number of logical CPUs on the local machine. Again, this requirement is expected to be retired as the scheduling and run-time improve.

### A leaky buffer

* The tools of concurrent programming can even make non-concurrent ideas easier to express.

* Here's an example abstracted from an RPC package. The client goroutine loops receiving data from some source, perhaps a network. To avoid allocating and freeing buffers, it keeps a free list, and uses a buffered channel to represent it. If the channel is empty, a new buffer gets allocated. Once the message buffer is ready, it's sent to the server and `serverChan`.

  ``` go
  var freeList = make(chan *Buffer, 100)
  var serverChan = make(chan *Buffer)

  func client() {
    for {
      var b *Buffer
      // Grab a buffer if available; allocate if not.
      select {
      case b = <-freeList:
        // Got One; nothing more to do.
      default:
        // None free, so allocate a new one.
        b = new(Buffer)
      }
    }
    load(b)          // Read next message from the net.
    serverChan <- b  // Send to server.
  }
  ```

  The server loop receives each message from the client, processes it, and returns the bfufer to the free list.

  ``` go
  func server() {
    for {
      b := <-serverChan   // Wait for work.
      process(b)
      // Reuse buffer if there's room.
      select {
      case freeList <- b:
        // Buffer on free list; nothing more to do.
      default:
        // Free list full, just carry on.
      }
    }
  }
  ```

  This implementation builds a leaky bucket free list in just a few lines, relying on the buffered channel and the garbage collector for bookkeeping.

## Errors

* By convention, errors have type `error`, a simple built-in interface.

  ``` go
  type error interface {
    Error() string
  }
  ```

* A library writer is free to implement this interface with a richer model under the covers, making it possible not only to see the error but also to provide some context. For example `os.Open` returns an `os.PathError`.

  ``` go
  // PathError records an error and the operation and
  // file path that caused it.
  type PathError struct {
    Op string     // "open", "unlink", etc.
    Path string   // The associated file.
    Err error     // Returned by the system call.
  }

  func (e *PathError) Error() string {
    return e.Op + " " + e.Path + ": " + e.Err.Error()
  }
  ```

  PathError's Error generates a string like this:

  ```
  open /etc/passwx: no such file or directory
  ```

* Callers that care about the precise error details can use a type switch or a type assertion to look for specific errors and extract details. For `PathErrors` this might include examing the internal `Err` field for recoverable failures.

  ``` go
  for try := 0; try < 2; try++ {
    file, err = os.Create(filename)
    if err == nil {
      return
    }
    if e, ok := err.(*os.PathError); ok && e.Err == syscall.ENOSPC {
      deleteTemFiles()   // Recover some space.
      continue
    }
    return
  }
  ```

### Panic

* There is a built-in function `panic` that in effect creates a run-time error that will stop the program (but see the next section). The function takes a single argument of arbitrary type-often a string-to be printed as the program dies. It's also a way to indicate that something impossible has happended, such as exiting an infinite loop.

  ``` go
  // A toy implementation of cube root using Newton's method.
  func CubeRoot(x float64) float64 {
    z := x/3     // Arbitrary initial value
    for i := 0; i < 1e6; i++ {
      prevz := z
      z -= (z*z*z-x) / (3*z*z)
      if veryClose(z, prevz) {
        return z
      }
    }

    // A million iterations has not converged; something is wrong.
    panic(fmt.Srpintf("CubeRoot(%g) did not converge", x))
  }
  ```

* This is only an example but real library functions should avoid panic. If the problem can be masked or worked around, it's always better to let things continue to run rather than taking down the whole program. One possible counterexample is during initialization: if the library truly cannot set itself up, it might be reasonable to panic, so to speak.

  ``` go
  var user = os.Getenv("USER")

  func init() {
    if user == "" {
      panic("no value for $USER")
    }
  }
  ```

### Recover

* When `panic` is called, including implicitly for run-time errors such as indexing a slice out of bounds or failing a type assertion, it immediately stops execution of the current function and begins unwinding the stack of the goroutine, running any deferred functions along the way. If that unwinding reaches the top of the goroutine's stack, the program dies. However, it is possible to use the built-in function `recover` to regain control of the goroutine and resume normal execution.

* A call to `recover` stops the unwinding and returns the argument passed to panic. Because the only code that runs while unwinding is inside deferred functions, recover is only useful inside deferred functions.

  ``` go
  func server(workChan <-chan *work) {
    for work := range workChan {
      go safelyDo(work)
    }
  }

  func safelyDo(work *Work) {
    defer func() {
      if err := recover(); err != nil {
        log.Println("work failed:", err)
      }
    }()
    do(work)
  }
  ```
