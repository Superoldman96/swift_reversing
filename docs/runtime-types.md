# Runtime types and storage

This guide collects common runtime representations encountered while reversing
Swift Mach-O binaries on ARM64. Confirm layouts against the Swift runtime
version used by your target.

## Primitive helper types

The included [`ida_header.h`](../ida_header.h) defines a few convenient IDA
types:

```c
typedef long long s64;
typedef unsigned long long u64;

typedef s64 Int;
typedef u64 Bool;

struct Swift_String
{
  u64 _countAndFlagsBits;
  void *_object;
};

union Swift_ElementAny {
    Swift_String stringElement;
};

struct Swift_Any {
    Swift_ElementAny element;
    u64 unknown;
    s64 type;
};

struct Swift_ArrayAny {
    s64 length;
    Swift_Any *items;
};
```

## `Swift::String`

Swift strings are common during reversing, but their storage representation can
be tricky to identify. `_countAndFlagsBits` and `_object` indicate where storage
resides.

The following patterns are useful heuristics for the binaries examined here:

- If `string->_object >> 60 == 0xE`, the value is stored in place inside the
  `_countAndFlagsBits` and `_object` members.
- If `string->_countAndFlagsBits >> 60 == 0xD`, the character data is commonly
  found at `(string->_object & 0xffffffffffffff) + 0x20`.

Confirm these assumptions against the target runtime before automating them.

### Value passing versus storage objects

`String` is a value type, not a class. When passed as an ordinary small value in
the binaries examined here, its two words commonly travel in registers such as
`X0` and `X1`.

Do not conclude that every pointer related to a string is another spelling of
that two-word value. Swift strings may use reference-backed storage, and code
may also operate through a method receiver, closure context, or temporary
address. If a string-related call uses `X20`, inspect the callee and surrounding
data flow:

- A Swift method or closure can receive an out-of-band `swiftself` or context
  parameter in `X20`.
- A value-type method may receive `self` using a lowered value representation or
  indirectly through an address.
- A pointer to reference-backed string storage is an implementation detail. It
  is not the `String` value itself and does not make `String` a class.

There are two common patterns to distinguish in assembly:

```asm
; Materialize the two-word String value, then pass its address as indirect self.
stp x0, x1, [sp, #local_string]
add x20, sp, #local_string
bl  string_method
```

In this case, `X20` points to a temporary buffer containing both words of the
`String` value. This can occur when the lowered call needs an address for
value-type `self`, such as a mutating operation.

```asm
; Move String's internal object word into a callee-saved register.
mov x20, x1
bl  storage_helper
```

In this case, no two-word `String` value was converted into `X20`. The code is
operating on the internal storage reference from `_object`, retaining a useful
value across calls, or preparing a different receiver or context. Inspect the
stores and loads around the call before assigning a type to `X20`.

See [Swift-specific registers](calling-conventions.md#swift-specific-registers)
for the Darwin ARM64 register roles.

## Structs

Swift structs are value types. Depending on their size and use, values may be
passed in registers, stored on the stack, or copied into global memory such as
the `__common` section.

On ARM64, small aggregate values are commonly returned in `X0`-`X3`. Larger
values are commonly returned indirectly through a caller-provided result buffer,
often passed in `X8`. The exact behavior depends on the ABI and type layout.

`Swift::String` is one example of a small struct. In the cases examined here, it
is returned in `X0`-`X1` and passed in two registers.

## Classes

Swift class representation resembles C++ in some respects. An allocating
initializer allocates memory using `swift_allocObject` before invoking the
user-defined initializer. The object's first word refers to its type metadata.

A simplified IDA model may look like this:

```c
struct SomeClassRTTI {
    Class classObject;
    Unknown metadata;

    (void (*)(SomeClass *self)) someMethod1;
    (void (*)(SomeClass *self)) someMethod2;
};

struct SomeClass {
    SomeClassRTTI *rtti;

    u64 ivar1;
    u64 ivar2;
};
```

Treat this as an analysis aid rather than a complete ABI definition. Accessors
and dispatch strategy depend on context.

## Existential containers

An **existential value** is a value used through a protocol type rather than its
concrete type. In modern Swift syntax, this is written as `any Protocol`.

For example:

```swift
let value: any StructProtocol = StructTest(a: 1, b: 2, c: 3)
```

The variable's static type says only that the stored value conforms to
`StructProtocol`. At runtime, Swift must also retain the concrete value and the
information needed to operate on it. The runtime representation used to package
those pieces together is an **existential container**.

This differs from a generic parameter such as `T: StructProtocol`. Generic code
still works with a particular concrete `T` for each invocation, even when that
type is not known until runtime.

When values of different concrete types are stored through the same protocol
type, Swift needs a common representation. A simplified existential container
for one protocol is:

```text
8 bytes - payload_1
8 bytes - payload_2
8 bytes - payload_3
8 bytes - pointer to type metadata
8 bytes - pointer to the Protocol Witness Table (PWT)
```

This is the common **opaque**, non-class-bounded existential layout. Swift's
[`TargetOpaqueExistentialContainer`](https://github.com/swiftlang/swift/blob/main/include/swift/Runtime/ExistentialContainer.h)
stores:

```text
ValueBuffer
concrete type metadata pointer
zero or more trailing PWT pointers
```

For one protocol with a Swift witness table, the metadata pointer follows the
three-word value buffer and the PWT pointer follows the metadata pointer. For a
protocol composition, additional PWT pointers trail the first one.

Do not apply that layout to every existential representation. A class-bounded
existential has a smaller shape:

```text
object reference
zero or more trailing PWT pointers
```

The object reference already identifies a class instance, so this container
does not store the separate concrete metadata word used by an opaque
existential. Existential metatypes and special cases such as errors also have
different representations.

`payload_1`, `payload_2`, and `payload_3` are slots in the existential
container's value buffer. They describe the container's in-memory layout, not a
fixed assignment to `X0`, `X1`, and `X2`.

A non-class-bounded existential may contain an address-only value. Swift's
[calling-convention notes](https://github.com/swiftlang/swift/blob/main/docs/ABI/CallingConvention.rst)
therefore classify it as address-only, so a parameter of that existential type
is passed indirectly as a pointer to the container. You may still see the
payload slots loaded into registers while code constructs, copies, or projects
the container. Those loads are local data movement, not a stable
`payload_1 -> X0` calling rule.

Concrete small structs and class-bounded existentials can lower differently.
Confirm the function signature and register mapping at each call site.

If a value fits in the three-word value buffer, it may be stored inline. If it
does not fit, the first payload word may point to heap storage while the
remaining payload words are unused.

The type metadata references a Value Witness Table (VWT). Operations on the
value buffer use that VWT. Calls to protocol requirements are dispatched through
the PWT. See [Protocols and witness tables](protocols-and-witness-tables.md).

### Inline and heap-backed example

Consider two structs that conform to the same protocol:

```swift
protocol StructProtocol {
    var a: Int { get }
    func first() -> Int
    func second() -> Int
}

struct StructTest: StructProtocol {
    var a: Int
    var b: Int
    var c: Int

    func first() -> Int { return 1 }
    func second() -> Int { return 2 }
}

struct StructTestLarge: StructProtocol {
    var a: Int
    var b: Int
    var c: Int
    var d: Int

    func first() -> Int { return 8 }
    func second() -> Int { return 9 }
}
```

`StructTest` contains three `Int` values, so its payload fits in the
three-word value buffer:

```text
payload_1 - structTesting.a
payload_2 - structTesting.b
payload_3 - structTesting.c
metadata  - type metadata for StructTest
pwt       - protocol witness table for StructTest
```

`StructTestLarge` contains four `Int` values, so its payload does not fit. The
container instead stores a pointer to heap-backed storage:

```text
payload_1 - pointer to heap-backed value
payload_2 - unused
payload_3 - unused
metadata  - type metadata for StructTestLarge
pwt       - protocol witness table for StructTestLarge
```

Both existential containers have the same shape even though their concrete
payloads have different sizes. Use the VWT when operating on the payload and the
PWT when dispatching protocol requirements. This fixed shape allows an array of
`StructProtocol` values to store existential containers contiguously even when
their underlying concrete values have different sizes.

## Type metadata

The Swift runtime keeps metadata for used types. A metadata pointer identifies a
concrete runtime type and gives runtime code access to layout-dependent
information. It is used for RTTI, generic functions, allocation, field offsets,
and other runtime operations.

For more detail, read
[`TypeMetadata.rst`](https://github.com/swiftlang/swift/blob/main/docs/ABI/TypeMetadata.rst),
but use the following patterns as a starting point while reversing.

When initializing some values, you may see a pattern similar to:

```c
void *typeMetadata =
    __swift_instantiateConcreteTypeFromMangledName(&metadataCache);
void *storage = __swift_allocate_value_buffer(typeMetadata, &globalVar);
void *valueAddress = __swift_project_value_buffer(typeMetadata, &globalVar);
```

Treat names beginning with `__swift_` in this example as descriptive
decompiler-style labels. Exact symbol names and inlining depend on the binary.
The underlying runtime behavior is represented by methods such as
[`Metadata::allocateBufferIn`](https://github.com/swiftlang/swift/blob/main/stdlib/public/runtime/Metadata.cpp)
and `Metadata::projectBufferFrom`.

The steps are:

1. `__swift_instantiateConcreteTypeFromMangledName(&metadataCache)` resolves a
   symbolic or mangled type reference and returns metadata for the concrete
   runtime type. A cache avoids repeating the work after the first lookup.
2. `__swift_allocate_value_buffer(typeMetadata, &globalVar)` consults the type's
   VWT to determine whether the value fits in a three-word `ValueBuffer`.
   - If it fits, the function returns `&globalVar`.
   - If it does not fit, the function allocates out-of-line storage, writes that
     pointer into the first word of `globalVar`, and returns the allocated
     address.
3. `__swift_project_value_buffer(typeMetadata, &globalVar)` returns the address
   where the value currently resides.
   - If the value is inline, it returns `&globalVar`.
   - If the value is out of line, it reads and returns the pointer stored in the
     first word of `globalVar`.

Allocation and projection answer different questions:

- **Allocate:** ensure storage exists for an uninitialized value.
- **Project:** find the address of storage that already exists.

`allocateBufferIn` already returns the usable storage address. Code does not
need to call projection immediately afterward just to obtain the same pointer.
The sequential calls above are a recognition aid: projection is useful later
when code has retained the `ValueBuffer` and needs to recover the value address
again.

Projection does not allocate memory. It is useful because later code can use one
address regardless of whether the value is inline:

```text
inline value:
  ValueBuffer [ value bytes ... ]
  project(ValueBuffer) ---> ValueBuffer

out-of-line value:
  ValueBuffer [ pointer ---> allocated value bytes ]
  project(ValueBuffer) ----------------^
```

When the buffer is no longer needed, the matching cleanup path must destroy the
value as required by its VWT and deallocate out-of-line storage when present.
The ABI method `Metadata::deallocateBufferIn` performs the latter operation. Do
not confuse this temporary out-of-line allocation with a copy-on-write
existential box; Swift's ABI comments distinguish those cases.

You may also encounter metadata through:

- a metadata accessor emitted for a nominal type;
- `swift_instantiateConcreteTypeFromMangledName`;
- an existential container's metadata field;
- an additional runtime argument passed to generic code.

When reversing, a metadata pointer is useful because it lets you connect code
that appears unrelated:

```text
metadata accessor or runtime instantiation
  ---> metadata pointer for ConcreteType
       ---> VWT used for storage operations
       ---> field offsets or other type-specific runtime information
```

For many value metadata layouts, the VWT pointer is stored immediately before
the metadata address used by runtime code. This often produces a load such as
`ldr x9, [x8, #-8]`. Confirm the metadata kind and runtime version before naming
that word as a VWT pointer.

Type metadata is not the same thing as:

- a **Protocol Descriptor**, which describes a protocol declaration;
- a **Protocol Conformance Descriptor**, which records a type's conformance;
- a **PWT**, which dispatches protocol requirements for one conformance.

See [Protocols and witness tables](protocols-and-witness-tables.md) for concrete
dispatch patterns.

IDA may fail to resolve the pointer passed to
`__swift_instantiateConcreteTypeFromMangledName`. Check whether it is a signed
32-bit relative pointer and resolve it manually if necessary.

Dynamic stack allocation may also produce calls to `__chkstk_darwin()`. The
space between those calls can correspond to local variables.
