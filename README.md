# heap.coffee

heap.coffee is a CoffeeScript dialect that has an optional C-like type system
with manual memory management which compiles to an explicit typed array. This
lets you write memory-efficient and GC pause-free code less painfully.

A word of warning: this is not a very good type system, or even does very
much. It's basically a glorified preprocessor to help compute offsets for
struct fields.

### Original CoffeeScript

```coffeescript
# A linked list type.
type node = struct
  val  :: int
  next :: *node

# Allocate 2 new nodes with new.
n :: *node
n = new node
m :: *node
m = new node

# Dot-notation now also dereference struct pointers.
n.val = 0
n.next = m
m.val = 1
m.next = null

# Free heap-allocated nodes with delete.
delete n
delete m

# Higher-order functions.
f :: (int) -> (int) -> int
f = (x) -> (y) -> x * y
```

### Compiled JavaScript

```javascript
// Generated by CoffeeScript 1.3.1
(function() {
  var f, m, n;
  // type node = struct { val: int, next: *node }

  // n :: *node
  n = malloc(H, 8);
  // m :: *node
  m = malloc(H, 8);

  H[n + 0] = 0;
  H[n + 4] = m;
  H[m + 0] = 1;
  H[m + 4] = null;

  free(H, n);
  free(H, m);

  // f :: (int) -> (int) -> int
  f = function(x) { // :: (int)
    return function(y) { // :: (int)
      return x * y;
    };
  };
}).call(this);
```

# Syntax

Type synonyms alias one type name to another. These may only appear at the
toplevel.

```coffeescript
type size_t = int
```

Type declarations tell the compiler what variable is what type. Our type
system is simple and C-like, so you must tell it the types of everything, it
doesn't really do any inference.

```coffeescript
i :: int
i = 0
```

Allocate using `new`:

```coffeescript
n :: *node
n = new node
```

Free using `delete`:

```coffeescript
delete n
```

Dereferencing in expressions and taking the address of variables are just like
in C:

```coffeescript
v  = *n
np = &n
```

Use dot where you would use arrow in C. Dereferencing of the struct
pointer is automatic.

```coffeescript
n = n.next
n = (*n).next # semantically equivalent to above
```

Use := to assign to the memory location of a pointer type. Unfortunately,
C-like syntax `*p = e` causes an ambiguity in the CoffeeScript grammar.

```coffeescript
i :: *int
i  = new int
i := 42
```

Pointer arithmetic work as it does in C.

# Types

heap.coffee includes the following primitive types:

- `byte` (1 byte)
- `short` (2 bytes)
- `int` (4 bytes)
- `uint` (4 bytes)

Struct types are declared using type synonyms and share syntax with creating
objects, except each property-value pair is a type declaration:

```coffeescript
type point = struct
  x :: int
  y :: int
```

Pointer types are prefixed by `*`:

```coffeescript
type intp = *int
```

Function types are composed using `->`. To keep things simple, pointers cannot
be taken of functions.

```coffeescript
type binop = (int, int) -> int
```

Normally, the user has to declare the types of parameters manually. There is a
special sugar in place for when a variable is given a function type
declaration and its value is found to be a _syntactic_ function. When this
holds, the compiler automatically inserts the declarations for the parameters.

```coffeescript
f :: (int, int) -> int
f = (x, y) -> x * y

# The above is the same as:
f :: (int, int) -> int
f = (x, y) :: (int, int) ->
  x * y
```

If the left-hand side of an assignment is typed, the right-hand side must also
be typed, and the two types must be unifiable.

```coffeescript
n :: *node
n = new node

i :: int
i = n  # error: incompatible types `int' and `*node'
i = {} # error: cannot assign untyped to typed
```