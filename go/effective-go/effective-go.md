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
