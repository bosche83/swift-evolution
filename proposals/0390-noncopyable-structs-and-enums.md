# `@noncopyable` structs and enums

* Proposal: [SE-0390](0390-noncopyable-structs-and-enums.md)
* Authors: [Joe Groff](https://github.com/jckarter), [Michael Gottesman](https://github.com/gottesmm), [Andrew Trick](https://github.com/atrick), [Kavon Farvardin](https://github.com/kavon)
* Review Manager: [Stephen Canon](https://github.com/stephentyrone)
* Status: **Active review (February 21 ... March 7, 2023)**
* Implementation: on main as `@_moveOnly` behind the `-enable-experimental-move-only` option
* Review: [(pitch)](https://forums.swift.org/t/pitch-noncopyable-or-move-only-structs-and-enums/61903)[(review)](https://forums.swift.org/t/se-0390-noncopyable-structs-and-enums/63258)

## Introduction

This proposal introduces the concept of **noncopyable** types (also known
as "move-only" types). An instance of a noncopyable type always has unique
ownership, unlike normal Swift types which can be freely copied.

## Motivation

All currently existing types in Swift are **copyable**, meaning it is possible
to create multiple identical, interchangeable representations of any value of
the type. However, copyable structs and enums are not a great model for
unique resources. Classes by contrast *can* represent a unique resource,
since an object has a unique identity once initialized, and only references to
that unique object get copied. However, because the references to the object are
still copyable, classes always demand *shared ownership* of the resource. This
imposes overhead in the form of heap allocation (since the overall lifetime of
the object is indefinite) and reference counting (to keep track of the number
of co-owners currently accessing the object), and shared access often
complicates or introduces unsafety or additional overhead into an object's
APIs. Swift does not yet have a mechanism for defining types that
represent unique resources with *unique ownership*.

## Proposed solution

We propose to allow for `struct` and `enum` types to be declared with the
`@noncopyable` attribute, which specifies that the declared type is
*noncopyable*.  Values of noncopyable type always have unique ownership, and
can never be copied (at least, not using Swift's implicit copy mechanism).
Since values of noncopyable structs and enums have unique identities, they can
also have `deinit` declarations, like classes, which run automatically at the
end of the unique instance's lifetime.

For example, a basic file descriptor type could be defined as:

```swift
@noncopyable
struct FileDescriptor {
  private var fd: Int32

  init(fd: Int32) { self.fd = fd }

  func write(buffer: Data) {
    buffer.withUnsafeBytes { 
      write(fd, $0.baseAddress!, $0.count)
    }
  }

  deinit {
    close(fd)
  }
}
```

Like a class, instances of this type can provide managed access to a file
handle, automatically closing the handle once the value's lifetime ends. Unlike
a class, no object needs to be allocated; only a simple struct containing the
file descriptor ID needs to be stored in the stack frame or aggregate value
that uniquely owns the instance.

## Detailed design

### Declaring noncopyable types

A `struct` or `enum` type can be declared as noncopyable using the `@noncopyable`
attribute:

```swift
@noncopyable
struct FileDescriptor {
  private var fd: Int32
}
```

If a `struct` has a stored property of noncopyable type, or an `enum` has
a case with an associated value of noncopyable type, then the containing type
must also be declared `@noncopyable`:

```swift
@noncopyable
struct SocketPair {
  var in, out: FileDescriptor
}

@noncopyable
enum FileOrMemory {
  // write to an OS file
  case file(FileDescriptor)
  // write to an array in memory
  case memory([UInt8])
}

// ERROR: copyable value type cannot contain noncopyable members
struct FileWithPath {
  var file: FileDescriptor
  var path: String
}
```

Classes, on the other hand, may contain noncopyable stored properties without
themselves becoming noncopyable:

```swift
class SharedFile {
  var file: FileDescriptor
}
```

### Restrictions on use in generics

Noncopyable types may have generic type parameters:

```swift
// A type that reads from a file descriptor consisting of binary values of type T
// in sequence.
@noncopyable
struct TypedFile<T> {
  var rawFile: FileDescriptor

  func read() -> T { ... }
}

let byteFile: TypedFile<UInt8> // OK
```

However, at this time, noncopyable types themselves are not allowed to be used
as a generic type. This means a noncopyable type _cannot_:

- conform to any protocols, except for `Sendable`.
- serve as a type witness for an `associatedtype` requirement.
- be used as a type argument when instantiating generic types or calling generic functions.
- be cast to (or from) `Any` or any other existential.
- be accessed through reflection.
- appear in a tuple.

The reasons for these restrictions and ways of lifting them are discussed under
Future Directions. The key implication of these restrictions is that a
noncopyable struct or enum is only a subtype of itself, because all other types
it might be compatible with for conversion would also permit copying.

#### The `Sendable` exception

The need for preventing noncopyable types from conforming to
protocols is rooted in the fact that all existing constrained generic types 
(like `some P` types) and existentials (`any P` types) are assumed to be 
copyable. Recording any conformances to these protocols would be invalid for
noncopyable types.

But, an exception is made where noncopyable types can conform to `Sendable`.
Unlike other protocols, the `Sendable` marker protocol leaves no conformance
record in the output program. Thus, there will be no ABI impact if a future 
noncopyable version of the `Sendable` protocol is created.

The big benefit of allowing `Sendable` conformances is that noncopyable types 
are compatible with concurrency. Keep in mind that despite their ability to 
conform to `Sendable`, noncopyable structs and enums are still only a subtype 
of themselves. That means when the noncopyable type conforms to `Sendable`, you
still cannot convert it to `any Sendable`, because copying that existential 
would copy its underlying value:

```swift
extension FileDescriptor: Sendable {} // OK

@noncopyable struct RefHolder: Sendable {
  var ref: Ref  // ERROR: stored property 'ref' of 'Sendable'-conforming struct 'RefHolder' has non-sendable type 'Ref'
}

func openAsync(_ path: String) async throws -> FileDescriptor {/* ... */}
func sendToSpace(_ s: some Sendable) {/* ... */}

@MainActor func example() async throws {
  // OK. FileDescriptor can cross actors because it is Sendable
  var fd: FileDescriptor = try await openAsync("/dev/null")

  // ERROR: noncopyable types cannot be conditionally cast
  // WARNING: cast from 'FileDescriptor' to unrelated type 'any Sendable' always fails
  if let sendy: Sendable = fd as? Sendable {

    // ERROR: noncopyable types cannot be conditionally cast
    // WARNING: cast from 'any Sendable' to unrelated type 'FileDescriptor' always fails
    fd = sendy as! FileDescriptor
  }

  // ERROR: noncopyable type 'FileDescriptor' cannot be used with generics
  sendToSpace(fd)
}
```

#### Working around the generics restrictions

Since a good portion of Swift's standard library rely on generics, there are a
a number of common types and functions that will not work with today's 
noncopyable types:

```swift
// ERROR: Cannot use noncopyable type FileDescriptor in generic type Optional
let x = Optional(FileDescriptor(open("/etc/passwd", O_RDONLY)))

// ERROR: Cannot use noncopyable type FileDescriptor in generic type Array
let fds: [FileDescriptor] = []

// ERROR: Cannot use noncopyable type FileDescriptor in generic type Any
print(FileDescriptor(-1))

// ERROR: Noncopyable struct SocketEvent cannot conform to Error
@noncopyable enum SocketEvent: Error {
  case requestedDisconnect(SocketPair)
}
```

For example, the `print` function expects to be able to convert its argument to
`Any`, which is a copyable value. Internally, it also relies on either 
reflection or conformance to `CustomStringConvertible`. Since a noncopyable type
can't do any of those, a suggested workaround is to explicitly define a 
conversion to `String`: 

```swift
extension FileDescriptor /*: CustomStringConvertible */ {
  var description: String {
    return "file descriptor #\(fd)"
  }
}

let fd = FileDescriptor(-1)
print(fd.description)
```

A more general kind of workaround to mix generics and noncopyable types
is to wrap the value in an ordinary class instance, which itself can participate
in generics. To transfer the noncopyable value in or out of the wrapper class
instance, using `Optional<FileDescriptor>` for the class's field would be 
ideal. But until that is supported, a concrete noncopyable enum can represent
the case where the value of interest was taken out of the instance:

```swift
@noncopyable
enum MaybeFileDescriptor {
  case some(FileDescriptor)
  case none
}

class WrappedFile {
  var file: MaybeFileDescriptor

  enum Err: Error { case noFile }

  init(_ fd: consuming FileDescriptor) {
    file = .some(fd)
  }

  func consume() throws -> FileDescriptor {
    if case let .some(fd) = file { // consume `self.file`
      file = .none // must reinitialize `self.file` before returning
      return fd
    }
    throw Err.noFile
  }
}

func example(_ fd1: consuming FileDescriptor, 
             _ fd2: consuming FileDescriptor) -> [WrappedFile] {
  // create an array of descriptors
  return [WrappedFile(fd1), WrappedFile(fd2)]
}
```

All of this boilerplate melts away once noncopyable types support generics.
Even before then, one major improvement would be to eliminate the need to define
types like `MaybeFileDescriptor` through a noncopyable `Optional` 
(see Future Directions).


### Using values of `@noncopyable` type

As the name suggests, values of noncopyable type cannot be copied, a major break
from most other types in Swift. Many operations are currently defined as
working as pass-by-value, and use copying as an implementation technique
to give that semantics, but they need to be defined more precisely in terms of
how they *borrow* or *consume* their operands in order to define their
effects on values that cannot be copied. 

We use the term **consume** to refer to an operation
that invalidates the value that it operates on. It may do this by directly
destroying the value, freeing its resources such as memory and file handles,
or forwarding ownership of the value to yet another owner who takes
responsibility for keeping it alive. Performing a consuming operation on
a noncopyable value generally requires having ownership of the value to begin
with, and invalidates the value the operation was performed on after it is
completed.

We use the term **borrow** to refer to
a shared borrow of a single instance of a value; the operation that borrows
the value allows other operations to borrow the same value simultaneously, and
it does not take ownership of the value away from its current owner. This
generally means that borrowers are not allowed to mutate the value, since doing
so would invalidate the value as seen by the owner or other simultaneous
borrowers. Borrowers also cannot *consume* the value. They can, however,
initiate arbitrarily many additional borrowing operations on all or part of
the value they borrow.

Both of these conventions stand in contrast to **mutating** (or **inout**)
operations, which take an *exclusive* borrow of their operands. The behavior
of mutating operations on noncopyable values is much the same as `inout`
parameters of copyable type today, which are already subject to the
"law of exclusivity". A mutating operation has exclusive access to its operand
for the duration of the operation, allowing it to freely mutate the value
without concern for aliasing or data races, since not even the owner may
access the value simultaneously. A mutating operation may pass its operand
to another mutating operation, but transfers exclusivity to that other operation
until it completes. A mutating operation may also pass its operand to
any number of borrowing operations, but cannot assume exclusivity while those
borrows are enacted; when the borrowing operations complete, the mutating
operation may assume exclusivity again. Unlike having true ownership of a
value, mutating operations give ownership back to the owner at the end of an
operation.  A mutating operation therefore may consume the current value of its
operand, but if it does, it must replace it with a new value before completing.

For copyable types, the distinction between borrowing and consuming operations
is largely hidden from the programmer, since Swift will implicitly insert
copies as needed to maintain the apparent value semantics of operations; a
consuming operation can be turned into a borrowing one by copying the value and
giving the operation the copy to consume, allowing the program to continue
using the original. This of course becomes impossible for values that cannot
be copied, forcing the distinction.

Many code patterns that are allowed for copyable types also become errors for
noncopyable values because they would lead to conflicting uses of the same
value, without the ability to insert copies to avoid the conflict. For example,
a copyable value can normally be passed as an argument to the same function
multiple times, even to a `borrowing` and `consuming` parameter of the same
call, and the compiler will copy as necessary to make all of the function's
parameters valid according to their ownership specifiers:

```
func borrow(_: borrowing Value, and _: borrowing Value) {}
func consume(_: consuming Value, butBorrow _: borrowing Value) {}
let x = Value()
borrow(x, and: x) // this is fine, multiple borrows can share
consume(x, butBorrow: x) // also fine, we'll copy x to let a copy be consumed
                         // while the other is borrowed
```

By contrast, a noncopyable value *must* be passed by borrow or consumed,
without copying. This makes the second call above impossible for a noncopyable
`x`, since attempting to consume `x` would end the binding's lifetime while
it also needs to be borrowed:

```
func borrow(_: borrowing FileDescriptor, and _: borrowing FileDescriptor) {}
func consume(_: consuming FileDescriptor, butBorrow _: borrowing FileDescriptor) {}
let x = FileDescriptor()
borrow(x, and: x) // still OK to borrow multiple times
consume(x, butBorrow: x) // ERROR: consuming use of `x` would end its lifetime
                         // while being borrowed
```

A similar effect happens when `inout` parameters take noncopyable arguments.
Swift will copy the value of a variable if it is passed both by value and
`inout`, so that the by-value parameter receives a copy of the current value
while leaving the original binding available for the `inout` parameter to
exclusively access:

```
func update(_: inout Value, butBorrow _: borrow Value) {}
func update(_: inout Value, butConsume _: consume Value) {}
var x = Value()
update(&x, butBorrow: x) // this is fine, we'll copy `x` in the second parameter
update(&x, butConsume: x) // also fine, we'll also copy
```

But again, for a noncopyable value, this implicit copy is impossible, so
these sorts of calls become exclusivity errors:

```
func update(_: inout FileDescriptor, butBorrow _: borrow FileDescriptor) {}
func update(_: inout FileDescriptor, butConsume _: consume FileDescriptor) {}

var y = FileDescriptor()
update(&y, butBorrow: y) // ERROR: cannot borrow `y` while exclusively accessed
update(&y, butConsume: y) // ERROR: cannot consume `y` while exclusively accessed
```

The following sections attempt to classify existing language operations
according to what ownership semantics they have when performed on noncopyable
values.

### Consuming operations

The following operations are consuming:

- assigning a value to a new `let` or `var` binding, or setting an existing
  variable or property to the binding:

    ```swift
    let x = FileDescriptor()
    let y = x
    use(x) // ERROR: x consumed by assignment to `y`
    ```

    ```swift
    var y = FileDescriptor()
    let x = FileDescriptor()
    y = x
    use(x) // ERROR: x consumed by assignment to `y`
    ```

    ```swift
    class C {
      var property = FileDescriptor()
    }
    let c = C()
    let x = FileDescriptor()
    c.property = x
    use(x) // ERROR: x consumed by assignment to `c.property`
    ```
- passing an argument to a `consuming` parameter of a function:

    ```swift
    func consume(_: consuming FileDescriptor) {}
    let x1 = FileDescriptor()
    consume(x1)
    use(x1) // ERROR: x1 consumed by call to `consume`
    ```

- passing an argument to an `init` parameter that is not explicitly
  `borrowing`:

    ```swift
    @noncopyable
    struct S {
      var x: FileDescriptor, y: Int
    }
    let x = FileDescriptor()
    let s = S(x: x, y: 219)
    use(x) // ERROR: x consumed by `init` of struct `S`
    ```

- invoking a `consuming` method on a value, or accessing a property of the
  value through a `consuming get` or `consuming set` accessor:

    ```swift
    extension FileDescriptor {
      consuming func consume() {}
    }
    let x = FileDescriptor()
    x.consume()
    use(x) // ERROR: x consumed by method `consume`
    ```

- explicitly consuming a value with the `consume` operator:

    ```swift
    let x = FileDescriptor()
    _ = consume x
    use(x) // ERROR: x consumed by explicit `consume`
    ```

- `return`-ing a value;

- pattern-matching a value with `switch`, `if let`, or `if case`:

    ```swift
    let x: Optional = getValue()
    if let y = consume x { ... }
    use(x) // ERROR: x consumed by `if let`

    @noncopyable
    enum FileDescriptorOrBuffer {
      case file(FileDescriptor)
      case buffer(String)
    }

    let x = FileDescriptorOrBuffer.file(FileDescriptor())

    switch x {
    case .file(let f):
      break
    case .buffer(let b):
      break
    }

    use(x) // ERROR: x consumed by `switch`
    ```

- iterating a `Sequence` with a `for` loop:

    ```swift
    let xs = [1, 2, 3]
    for x in consume xs {}
    use(xs) // ERROR: xs consumed by `for` loop
    ```

(Although noncopyable types are not currently allowed to conform to
protocols, preventing them from implementing the `Sequence` protocol,
and cannot be used as generic parameters, preventing the formation of
`Optional` noncopyable types, these last two cases are listed for completeness,
since they would affect the behavior of other language features that
suppress implicit copying when applied to copyable types.)

The `consume` operator can always transfer ownership of its operand when the
`consume` expression is itself the operand of a consuming operation.

Consuming is flow-sensitive, so if one branch of an `if` or other control flow
consumes a noncopyable value, then other branches where the value
is not consumed may continue using it:

```swift
let x = FileDescriptor()
guard let condition = getCondition() else {
  consume(x)
}
// We can continue using x here, since only the exit branch of the guard
// consumed it
use(x)
```

### Borrowing operations

The following operations are borrowing:

- Passing an argument to a `func` or `subscript` parameter that does not
  have an ownership modifier, or an argument to any `func`, `subscript`, or
  `init` parameter which is explicitly marked `borrow`. The
  argument is borrowed for the duration of the callee's execution.
- Borrowing a stored property of a struct or tuple borrows the struct or tuple
  for the duration of the access to the stored property. This means that one
  field of a struct cannot be borrowed while another is being mutated, as in
  `call(struc.fieldA, &struc.fieldB)`. Allowing for fine-grained subelement
  borrows in some circumstances is discussed as a Future Direction below.
- A stored property of a class may be borrowed using a dynamic exclusivity
  check, to assert that there are no aliasing mutations attempted during the
  borrow, as discussed under "Noncopyable stored properties in classes" below.
- Invoking a `borrowing` method on a value, or a method which is not annotated
  as any of `borrowing`, `consuming` or `mutating`, borrows the `self` parameter
  for the duration of the callee's execution.
- Accessing a computed property or subscript through `borrowing` or
  `nonmutating` getter or setter borrows the `self` parameter for the duration
  of the accessor's execution.
- Capturing an immutable local binding into a nonescaping closure borrows the
  binding for the duration of the callee that receives the nonescaping closure.

### Mutating operations

The following operations are mutating uses:

- Passing an argument to a `func` parameter that is `inout`. The argument is
  exclusively accessed for the duration of the call.
- Projecting a stored property of a struct for mutation is a mutating use of
  the entire struct.
- A stored property of a class may be mutated using a dynamic exclusivity
  check, to assert that there are no aliasing mutations, as happens today.
  For noncopyable properties, the assertion also enforces that no borrows
  are attempted during the mutation, as discussed under "Noncopyable stored
  properties in classes" below.
- Invoking a `mutating` method on a value is a mutating use of the `self`
  parameter for the duration of the callee's execution.
- Accessing a computed property or subscript through a `mutating` getter and/or
  setter is a mutating use of `self` for the duration of the accessor's
  execution.
- Capturing a mutable local binding into a nonescaping closure is a mutating
  use of the binding for the duration of the callee that receives the
  nonescaping closure.

### Declaring functions and methods with noncopyable parameters

When noncopyable types are used as function parameters, the ownership
convention becomes a much more important part of the API contract.
As such, when a function parameter is declared with an noncopyable type, it
**must** declare whether the parameter uses the `borrowing`, `consuming`, or
`inout` convention:

```swift
// Redirect a file descriptor
// Require exclusive access to the FileDescriptor to replace it
func redirect(_ file: inout FileDescriptor, to otherFile: borrowing FileDescriptor) {
  dup2(otherFile.fd, file.fd)
}

// Write to a file descriptor
// Only needs shared access
func write(_ data: [UInt8], to file: borrowing FileDescriptor) {
  data.withUnsafeBytes {
    write(file.fd, $0.baseAddress, $0.count)
  }
}

// Close a file descriptor
// Consumes the file descriptor
func close(file: consuming FileDescriptor) {
  close(file.fd)
}
```

Methods of the noncopyable type are considered to be `borrowing` unless
declared `mutating` or `consuming`:

```swift
extension FileDescriptor {
  mutating func replace(with otherFile: borrowing FileDescriptor) {
    dup2(otherFile.fd, self.fd)
  }

  // borrowing by default
  func write(_ data: [UInt8]) {
    data.withUnsafeBytes {
      write(file.fd, $0.baseAddress, $0.count)
    }
  }

  consuming func close() {
    close(fd)
  }
}
```

### Declaring properties of noncopyable type

A class or noncopyable struct may declare stored `let` or `var` properties of
noncopyable type. A noncopyable `let` stored property may only be borrowed,
whereas a `var` stored property may be both borrowed and mutated. Stored
properties cannot generally be consumed because doing so would leave the
containing aggregate in an invalid state.

Any type may also declare computed properties of noncopyable type. The `get`
accessor returns an owned value that the caller may consume, like a function
would. The `set` accessor receives its `newValue` as a `consuming` parameter,
so the setter may consume the parameter value to update the containing
aggregate.

Accessors may use the `consuming` and `borrowing` declaration modifiers to
affect the ownership of `self` while the accessor executes. `consuming get`
is particularly useful as a way of forwarding ownership of part of an aggregate,
such as to take ownership away from a wrapper type:

```
@noncopyable
struct FileDescriptorWrapper {
  private var _value: FileDescriptor

  var value: FileDescriptor {
    consuming get { return _value }
  }
}
```

However, a `consuming get` cannot be paired with a setter when the containing
type is `@noncopyable`, because invoking the getter consumes the aggregate,
leaving nothing to write a modified value back to.

Because getters return owned values, non-`consuming` getters generally cannot
be used to wrap noncopyable stored properties, since doing so would require
copying the value out of the aggregate:

```
class File {
  private var _descriptor: FileDescriptor

  var descriptor: FileDescriptor {
    return _descriptor // ERROR: attempt to copy `_descriptor`
  }
}
```

These limitations could be addressed in the future by exposing the ability for
computed properties to also provide "read" and "modify" coroutines, which would
have the ability to yield borrowing or mutating access to properties without
copying them.

### Using stored properties and enum cases of `@noncopyable` type

When classes or `@noncopyable` types contain members that are of `@noncopyable`
type, then the container is the unique owner of the member value. Outside of
the type's definition, client code cannot perform consuming operations on
the value, since it would need to take away the container's ownership to do
so:

```
@noncopyable
struct Inner {}

@noncopyable
struct Outer {
  var inner = Inner()
}

let outer = Outer()
let i = outer.inner // ERROR: can't take `inner` away from `outer`
```

However, when code has the ability to mutate the member, it may freely modify,
reassign, or replace the value in the field:

```
var outer = Outer()
let newInner = Inner()
// OK, transfers ownership of `newInner` to `outer`, destroying its previous
// value
outer.inner = newInner
```

Note that, as currently defined, `switch` to pattern-match an `enum` is a
consuming operation, so it can only be performed inside `consuming` methods
on the type's original definition:

```
@noncopyable
enum OuterEnum {
  case inner(Inner)
  case file(FileDescriptor)
}

// Error, can't partially consume a value outside of its definition
let enum = OuterEnum.inner(Inner())
switch enum {
case .inner(let inner):
  break
default:
  break
}
```

Being able to borrow in pattern matches would address this shortcoming.

### Noncopyable stored properties in classes

Since objects may have any number of simultaneous references, Swift uses
dynamic exclusivity checking to prevent simultaneous writes of the same
stored property. This dynamic checking extends to borrows of noncopyable
stored properties; the compiler will attempt to diagnose obvious borrowing
failures, as it will for local variables and value types, but a runtime error
will occur if an uncaught exclusivity error occurs, such as an attempt to mutate
an object's stored property while it is being borrowed:

```
class Foo {
  var fd: FileDescriptor

  init(fd: FileDescriptor) { self.fd = fd }
}

func update(_: inout FileDescriptor, butBorrow _: borrow FileDescriptor) {}

func updateFoo(_ a: Foo, butBorrowFoo b: Foo) {
  update(&a.fd, butBorrow: b.fd)
}

let foo = Foo(fd: FileDescriptor())

// Will trap at runtime when foo.fd is borrowed and mutated at the same time
updateFoo(foo, butBorrowFoo: foo)
```

`let` properties do not allow mutating accesses, and this continues to hold for
noncopyable types. The value of a `let` property in a class therefore does not
need dynamic checking, even if the value is noncopyable; the value behaves as
if it is always borrowed, since there may potentially be a borrow through
some reference to the object at any point in the program. Such values can
thus never be consumed or mutated.

The dynamic borrow state of properties is tracked independently for every
stored property in the class, so it is safe to mutate one property while other
properties of the same object are also being mutated or borrowed:

```
class SocketTriple {
  var in, middle, out: FileDescriptor
}

func update(_: inout FileDescriptor, and _: inout FileDescriptor,
            whileBorrowing _: borrowing FileDescriptor) {}

// This is OK
let object = SocketTriple(...)
update(&object.in, and: &object.out, whileBorrowing: object.middle)
```

This dynamic tracking, however, cannot track accesses at finer resolution
than properties, so in circumstances where we might otherwise eventually be
able to support independent borrowing of fields in structs, tuples, and enums,
that support will not extend to fields within class properties, since the
entire property must be in the borrowing or mutating state.

Dynamic borrowing or mutating accesses require that the enclosing object be
kept alive for the duration of the assertion of the access. Normally, this 
is transparent to the developer, as the compiler will keep a copy of a
reference to the object retained while these accesses occur. However, if
we introduce noncopyable bindings to class references, such as [the `borrow`
and `inout` bindings](https://forums.swift.org/t/pitch-borrow-and-inout-declaration-keywords/62366)
currently being pitched, this would manifest as a borrow of the noncopyable
reference, preventing mutation or consumption of the reference during
dynamically-asserted accesses to its properties:

```
class SocketTriple {
  var in, middle, out: FileDescriptor
}

func borrow(_: borrowing FileDescriptor,
            whileReplacingObject _: inout SocketTriple) {}

var object = SocketTriple(...)

// This is OK, since ARC will keep a copy of the `object` reference retained
// while `object.in` is borrowed
borrow(object.in, whileReplacingObject: &object)

inout objectAlias = &object

// This is an error, since we aren't allowed to implicitly copy through
// an `inout` binding, and replacing `objectAlias` without keeping a copy
// retained might invalidate the object while we're accessing it.
borrow(objectAlias.in, whileReplacingObject: &objectAlias)
```

### Noncopyable variables captured by escaping closures

Nonescaping closures have scoped lifetimes, so they can borrow their captures,
as noted in the "borrowing operations" and "consuming operations" sections
above. Escaping closures, on the other hand, have indefinite lifetimes, since
they can be copied and passed around arbitrarily, and multiple escaping closures
can capture and access the same local variables alongside the local context
from which those captures were taken. Variables captured by escaping closures
thus behave like class properties; immutable captures are treated as always
borrowed both inside the closure body and in the capture's original context
after the closure was formed.

```
func escape(_: @escaping () -> ()) {...}

func borrow(_: borrowing FileDescriptor) {}
func consume(_: consuming FileDescriptor) {}

func foo() {
  let x = FileDescriptor()

  escape {
    borrow(x) // OK
    consume(x) // ERROR: cannot consume captured variable
  }

  // OK
  borrow(x)

  // ERROR: cannot consume variable after it's been captured by an escaping
  // closure
  consume(x)
}
```

Mutable captures are subject to dynamic exclusivity checking like class
properties are.

```
var escapedClosure: (@escaping (inout FileDescriptor) -> ())?

func foo() {
  var x = FileDescriptor()

  escapedClosure = { _ in borrow(x) }

  // Runtime error when exclusive access to `x` dynamically conflicts
  // with attempted borrow of `x` during `escapedClosure`'s execution
  escapedClosure!(&x)
}
```

### Deinitializers

A `@noncopyable` struct or enum may declare a `deinit`, which will run
implicitly when the lifetime of the value ends (unless explicitly suppressed
as noted below):

```swift
@noncopyable
struct FileDescriptor {
  private var fd: Int32

  deinit {
    close(fd)
  }
}
```

Like a class `deinit`, a struct or enum `deinit` may not propagate any uncaught
errors. The body of `deinit` has exclusive access to `self` for the duration
of its execution, so `self` behaves as in a `mutating` method; it may be
modified by the body of `deinit`, but must remain valid until the end of the
deinit. (Allowing for partial invalidation inside a `deinit` is explored as
a future direction.)

A value's lifetime ends, and its `deinit` runs if present, in the following
circumstances:

- For a local `var` or `let` binding, or `consuming` function parameter, that
  is not itself consumed, `deinit` runs after the last non-consuming use.
  If, on the other hand, the binding is consumed, then responsibility for
  deinitialization gets forwarded to the consumer (which may in turn forward
  it somewhere else).

    ```swift
    do {
      var x = FileDescriptor(42)

      x.close() // consuming use
      // x's deinit doesn't run here (but might run inside `close`)
    }

    do {
      var x = FileDescriptor(42)
      x.write([1,2,3]) // borrowing use
      // x's deinit runs here

      print("done writing")
    }
    ```

- When a `struct`, `enum`, or `class` contains a member of noncopyable type,
  the member is destroyed, and its `deinit` is run, after the container's
  `deinit` if any runs.

    ```swift
    @noncopyable
    struct Inner {
      deinit { print("destroying inner") }
    }

    @noncopyable
    struct Outer {
      var inner = Inner()
      deinit { print("destroying outer") }
    }

    do {
      _ = Outer()
    }
    ```

    will print:

    ```swift
    destroying outer
    destroying inner
    ```

### Suppressing `deinit` in a `consuming` method

It is often useful for noncopyable types to provide alternative ways to consume
the resource represented by the value besides the `deinit`. However,
under normal circumstances, a `consuming` method will still invoke the type's
`deinit` after the last use of `self`, which is undesirable when the method's
own logic already invalidates the value:

```swift
@noncopyable
struct FileDescriptor {
  private var fd: Int32

  deinit {
    close(fd)
  }

  consuming func close() {
    close(fd)

    // The lifetime of `self` ends here, triggering `deinit` (and another call to `close`)!
  }
}
```

In the above example, the double-close could be avoided by having the
`close()` method do nothing on its own and just allow the `deinit` to
implicitly run. However, we may want the method to have different behavior
from the deinit, such as raising an error (which a normal `deinit` is unable to
do) if the `close` system call triggers an OS error :

```swift
@noncopyable
struct FileDescriptor {
  private var fd: Int32

  consuming func close() throws {
    // POSIX close may raise an error (which still invalidates the
    // file descriptor, but may indicate a condition worth handling)
    if close(fd) != 0 {
      throw CloseError(errno)
    }

    // We don't want to trigger another close here!
  }
}
```

or it could be useful to take manual control of the file descriptor back from
the type, such as to pass to a C API that will take care of closing it:

```swift
@noncopyable
struct FileDescriptor {
  // Take ownership of the C file descriptor away from this type,
  // returning the file descriptor without closing it
  consuming func take() -> Int32 {
    return fd

    // We don't want to trigger close here!
  }
}
```

We propose to introduce a special operator, `forget self`, which ends the
lifetime of `self` without running its `deinit`:

```swift
@noncopyable
struct FileDescriptor {
  // Take ownership of the C file descriptor away from this type,
  // returning the file descriptor without closing it
  consuming func take() -> Int32 {
    let fd = self.fd
    forget self
    return fd
  }
}
```

`forget self` can only be applied to `self`, in a consuming method defined in
the type's defining module. (This is in contrast to Rust's similar special
function, [`mem::forget`](https://doc.rust-lang.org/std/mem/fn.forget.html),
which is a standalone function which can be applied to any value, anywhere.
Although the Rust documentation notes that this operation is "safe" on the
principle that destructors may not run at all, due to reference cycles,
process termination, etc., in practice the ability to forget arbitrary values
creates semantic issues for many Rust APIs, particularly when there are
destructors on types with lifetime dependence on each other like `Mutex`
and `LockGuard`. As such, we
think it is safer to restrict the ability to forget a value to the
core API of its type. We can relax this restriction if experience shows a
need to.)

Even with the ability to `forget self`, care would still need be taken when
writing destructive operations to avoid triggering the deinit on alternative
exit paths, such as early `return`s, `throw`s, or implicit propagation of
errors from `try` operations. For instance, if we write:

```swift
@noncopyable
struct FileDescriptor {
  private var fd: Int32

  consuming func close() throws {
    // POSIX close may raise an error (which still invalidates the
    // file descriptor, but may indicate a condition worth handling)
    if close(fd) != 0 {
      throw CloseError(errno)
      // !!! Oops, we didn't forget self on this path, so we'll deinit!
    }

    // We don't need to deinit self anymore
    forget self
  }
}
```

then the `throw` path exits the method without `forget`-ing `self`, so
`deinit` will still execute if an error occurs. To avoid this mistake, we
propose that if any path through a method uses `forget self`, then **every**
path must choose either to `forget` or to explicitly `consume self` using
the standard `deinit`. This will make
the above code an error, alerting that the code should be rewritten to ensure
`forget self` always executes:

```swift
@noncopyable
struct FileDescriptor {
  private var fd: Int32

  consuming func close() throws {
    // Save the file descriptor and give up ownership of it
    let fd = self.fd
    forget self

    // We can now use `fd` below without worrying about `deinit`:

    // POSIX close may raise an error (which still invalidates the
    // file descriptor, but may indicate a condition worth handling)
    if close(fd) != 0 {
      throw CloseError(errno)
    }
  }
}
```

The [consume operator](https://github.com/apple/swift-evolution/blob/main/proposals/0377-parameter-ownership-modifiers.md)
must be used to explicitly end the value's lifetime using its `deinit` if
`forget` is used to conditionally destroy the value on other paths through
the method.

```
@noncopyable
struct MemoryBuffer {
  private var address: UnsafeRawPointer

  init(size: Int) throws {
    guard let address = malloc(size) else {
      throw MallocError()
    }
    self.address = address
  }

  deinit {
    free(address)
  }

  consuming func takeOwnership(if condition: Bool) -> UnsafeRawPointer? {
    if condition {
      // Save the memory buffer and give it to the caller, who
      // is promising to free it when they're done.
      let address = self.address
      forget self
      return address
    } else {
      // We still want to free the memory if we aren't giving it away.
      _ = consume self
      return nil
    }
  }
}
```

## Source compatibility

For existing Swift code, this proposal is additive.

## Effect on ABI stability

### Adding or removing `@noncopyable` breaks ABI

An existing copyable struct or enum cannot be made `@noncopyable` without
breaking ABI, since existing clients may copy values of the type.

Ideally, we would allow noncopyable types to become copyable without breaking
ABI; however, we cannot promise this, due to existing implementation choices we
have made in the ABI that cause the copyability of a type to have unavoidable
knock-on effects. In particular, when properties are declared in classes,
protocols, or public non-`@frozen` structs, we define the property's ABI to use
accessors even if the property is stored, with the idea that it should be
possible to change a property's implementation to change it from a stored to
computed property, or vice versa, without breaking ABI.

The accessors used as ABI today are the traditional `get` and `set`
computed accessors, as well as a `_modify` coroutine which can optimize `inout`
operations and projections into stored properties. `_modify` and `set` are
not problematic for noncopyable types. However, `get` behaves like a
function, producing the property's value by returning it like a function would,
and returning requires *consuming* the return value to transfer it to the
caller. This is not possible for noncopyable stored properties, since the
value of the property cannot be copied in order to return a copy without
invalidating the entire containing struct or object.

Therefore, properties of noncopyable type need a different ABI in order to
properly abstract them. In particular, instead of exposing a `get` accessor
through abstract interfaces, they must use a `_read` coroutine, which is the
read-only analog to `_modify`, allowing the implementation to yield a borrow of
the property value in-place instead of returning by value. This allows for
noncopyable stored properties to be exposed while still being abstracted enough
that they can be replaced by a computed implementation, since a `get`-based
implementation could still work underneath the `read` coroutine by evaluating
the getter, yielding a borrow of the returned value, then disposing of the
temporary value.

As such, we cannot simply say that making a noncopyable type copyable is an
ABI-safe change, since doing so will have knock-on effects on the ABI of any
properties of the type. We could potentially provide a "born noncopyable"
attribute to indicate that a copyable type should use the noncopyable ABI
for any properties, as a way to enable the evolution into a copyable type
while preserving existing ABI. However, it also seems unlikely to us that many
types would need to evolve between being copyable or not frequently.

### Adding, removing, or changing `deinit` in a struct or enum

An noncopyable type that is not `@frozen` can add or remove its deinit without
affecting the type's ABI. If `@frozen`, a deinit cannot be added or removed,
but the deinit implementation may change (if the deinit is not additionally
`@inlinable`).

### Adding noncopyable fields to classes

A class may add fields of noncopyable type without changing ABI.

## Effect on API resilience

Introducing new APIs using noncopyable types is an additive change. APIs that
adopt noncopyable types have some notable restrictions on how they can further
evolve while maintaining source compatibility.

A noncopyable type can be made copyable while generally maintaining source
compatibility. Values in client source would acquire normal ARC lifetime
semantics instead of eager-move semantics when those clients are recompiled
with the type as copyable, and that could affect the observable order of
destruction and cleanup. Since copyable value types cannot directly define
`deinit`s, being able to observe these order differences is unlikely, but not
impossible when references to classes are involved.

A `consuming` parameter of noncopyable type can be changed into a `borrowing`
parameter without breaking source for clients (and likewise, a `consuming`
method can be made `borrowing`). Conversely, changing
a `borrowing` parameter to `consuming` may break client source. (Either direction
is an ABI breaking change.) This is because a consuming use is required to
be the final use of a noncopyable value, whereas a borrowing use may or may not
be.

Adding or removing a `deinit` to a noncopyable type does not affect source
for clients.

## Alternatives considered

### Naming the attribute "move-only"

We have frequently referred to these types as "move-only types" in various
vision documents. However, as we've evolved related proposals like the
`consume` operator and parameter modifiers, the community has drifted away
from exposing the term "move" in the language elsewhere. When explaining these
types to potential users, we've also found that the name "move-only" incorrectly
suggests that being noncopyable is a new capability of types, and that there
should be generic functions that only operate on "move-only" types, when really
the opposite is the case: all existing types in Swift today conform to
effectively an implicit "Copyable" requirement, and what this feature does is
allow types not to fulfill that requirement. When generics grow support for
move-only types, then generic functions and types that accept noncopyable
type parameters will also work with copyable types, since copyable types
are strictly more capable.This proposal prefers the term "noncopyable" to make
the relationship to an eventual `Copyable` constraint, and the fact that annotated
types lack the ability to satisfy this constraint, more explicit.

### Spelling as a generic constraint

It's a reasonable question why declaring a type as noncopyable isn't spelled
like a protocol constraint:

```
struct Foo: NonCopyable {}
```

As noted in the previous discussion, an issue with this notation is that it
implies that `NonCopyable` is a new capability or requirement, rather than
really being the lack of a `Copyable` capability. For an example of why
this might be misleading, consider what would happen if we expand
standard library collection types to support noncopyable elements. Value types
like `Array` and `Dictionary` would become copyable only when the elements they
contain are copyable. However, we cannot write this in terms of `NonCopyable`
conditional requirements, since if we write:

```
extension Dictionary: NonCopyable where Key: NonCopyable, Value: NonCopyable {}
```

this says that the dictionary is noncopyable only when both the key and value
are noncopyable, which is wrong because we also can't copy the dictionary if only
the keys or only the values are noncopyable. If we flip the constraint to
`Copyable`, the correct thing would fall out naturally:

```
extension Dictionary: Copyable where Key: Copyable, Value: Copyable {}
```

However, for progressive disclosure and source compatibility reasons, we still
want the majority of types to be `Copyable` by default, without making them
explicitly declare it; noncopyable types are likely to remain the exception
rather than the rule, with automatic lifetime management via ARC by the
compiler being sufficient for most code like it is today.

We could conversely borrow another idea from Rust, which uses the syntax
`?Trait` to declare that a normally implicit trait is not required by
a generic declaration, or not satisfied by a concrete type. So in Swift we
might write:

```
struct Foo: ?Copyable {
}
```

`Copyable` is currently the only such implicit constraint we are considering,
so the `@noncopyable` attribute is appealing as a specific solution to address
this case. If we were to consider making other requirements implicit (perhaps
`Sendable` in some situations?) then a more general opt-out syntax would be
called for.

### English language bikeshedding

Some dictionaries specify that "copiable" is the standard spelling for "able to
copy", although the Oxford English Dictionary and Merriam-Webster both also
list "copyable" as an accepted alternative. We prefer the more regular "copyable"
spelling.

The negation could just as well be spelled `@uncopyable` instead of `@noncopyable`.
Swift has precedent for favoring `non-` in modifiers, including `@nonescaping`
parameters and and `nonisolated` actor members, so we choose to follow that
precedent.

## Future directions

### Noncopyable tuples

It should be possible for a tuple to contain noncopyable elements, rendering
the tuple noncopyable if any of its elements are. Since tuples' structure is
always known, it would be reasonable to allow for the elements within a tuple
to be independently borrowed, mutated, and consumed, as the language allows
today for the elements of a tuple to be independently mutated via `inout`
accesses. (Due to the limitations of dynamic exclusivity checking, this would
not be possible for class properties, globals, and escaping closure captures.)

### Noncopyable `Optional`

This proposal initiates support for noncopyable types without any support for
generics at all, which precludes their use in most standard library types,
including `Optional`. We expect the lack of `Optional` support in particular
to be extremely limiting, since `Optional` can be used to manage dynamic
consumption of noncopyable values in situations where the language's static
rules cannot soundly support consumption. For instance, the static rules above
state that a stored property of a class can never be consumed, because it is
not knowable if other references to an object exist that expect the property
to be inhabited. This could be avoided using `Optional` with `mutating`
operation that forwards ownership of the `Optional` value's payload, if any,
writing `nil` back. Eventually this could be written as an extension method
on `Optional`:

```
@noncopyable
extension Optional {
  mutating func take() -> Wrapped {
    switch self {
    case .some(let wrapped):
      self = nil
      return wrapped
    case .none:
      fatalError("trying to take from an Optional that's already empty")
    }
  }
}

class Foo {
  var fd: FileDescriptor?

  func close() {
    // We normally would not be able to close `fd` except via the
    // object's `deinit` destroying the stored property. But using
    // `Optional` assignment, we can dynamically end the value's lifetime
    // here.
    fd = nil
  }

  func takeFD() -> FileDescriptor {
    // We normally would not be able to forward `fd`'s ownership to
    // anyone else. But using
    // `Optional.take`, we can dynamically end the value's lifetime
    // here.
    return fd.take()
  }
}
```

Without `Optional` support, the alternative would be for every noncopyable type
to provide its own ad-hoc `nil`-like state, which would be very unfortunate,
and go against Swift's general desire to encourage structural code correctness
by making invalid states unrepresentable. Therefore, `Optional` is likely to
be worth considering as a special case for noncopyable support, ahead of full
generics support for noncopyable types.

### Generics support for noncopyable types

This proposal comes with an admittedly severe restriction that noncopyable types
cannot conform to protocols or be used at all as type arguments to generic
functions or types, including common standard library types like `Optional`
and `Array`. All generic parameters in Swift today carry an implicit assumption
that the type is copyable, and it is another large language design project to
integrate the concept of noncopyable types into the generics system. Full
integration will very likely also involve changes to the Swift runtime and
standard library to accommodate noncopyable types in APIs that weren't
originally designed for them, and this integration might then have backward
deployment restrictions. We believe that, even with these restrictions,
noncopyable types are a useful self-contained addition to the language for
safely and efficiently modeling unique resources, and this subset of the feature
also has the benefit of being adoptable without additional runtime requirements,
so developers can begin making use of the feature without giving up backward
compatibility with existing Swift runtime deployments.

### Conditionally copyable types

This proposal states that a type, including one with generic parameters, is
currently always copyable or always noncopyable. However, some types may
eventually be generic over copyable and non-copyable types, with the ability
to be copyable for some generic arguments but not all. A simple case might be
a tuple-like `Pair` struct:

```
@noncopyable
// : ?Copyable is strawman syntax for declaring T and U don't require copying
struct Pair<T: ?Copyable, U: ?Copyable> {
  var first: T
  var second: U
}
```

We will need a way to express this conditional copyability, perhaps using
conditional conformance style declarations:

```
extension Pair: Copyable where T: Copyable, U: Copyable {}
```

### Finer-grained destructuring in `consuming` methods and `deinit`

As currently specified, noncopyable types are (outside of `init` implementations)
always either fully initialized or fully destroyed, without any support
for incremental destruction inside of `consuming` methods or deinits. A
`deinit` may modify, but not invalidate, `self`, and a `consuming` method may
`forget self`, forward ownership of all of `self`, or destroy `self`, but cannot
yet partially consume parts of `self`. This would be particularly useful for
types that contain other noncopyable types, which may want to relinquish
ownership of some or all of the resources owned by those members. In the current
proposal, this isn't possible without allowing for an intermediate invalid
state:

```swift
@noncopyable
struct SocketPair {
  let input, output: FileDescriptor

  // Gives up ownership of the output end, closing the input end
  consuming func takeOutput() -> FileDescriptor {
    // We would like to do something like this, taking ownership of
    // `self.output` while leaving `self.input` to be destroyed.
    // However, we can't do this without being able to either copy
    // `self.output` or partially invalidate `self`
    let output = self.output
    forget self
    return output
  }
}
```

Analogously to how `init` implementations use a "definite initialization"
pass to allow the value to initialized field-by-field, we can implement the
inverse dataflow pass to allow `deinit` implementations, as well as `consuming`
methods that `forget self`, to partially invalidate `self`.

### `read` and `modify` accessor coroutines for computed properties.

The current computed property model allows for properties to provide a getter,
which returns the value of the property on read to the caller as an owned value,
and optionally a setter, which receives the `newValue` of the property as
a parameter with which to update the containing type's state. This is
sometimes inefficient for value types, since the get/set pattern requires
returning a copy, modifying the copy, then passing the copy back to the setter
in order to model an in-place update, but it also limits what computed
properties can express for noncopyable types. Because a getter has to return
by value, it cannot pass along the value of a stored noncopyable property
without also destroying the enclosing aggregate, so `get`/`set` cannot be used
to wrap logic around access to a stored noncopyable property.

The Swift stable ABI for properties internally uses **accessor coroutines**
to allow for efficient access to stored properties, while still providing
abstraction that allows library evolution to change stored properties into
computed and back. These coroutines **yield** access to a value in-place for
borrowing or mutating, instead of passing copies of values back and forth.
We can expose the ability for code to implement these coroutines directly,
which is a good optimization for copyable value types, but also allows for
more expressivity with noncopyable properties.
