# Protocols and witness tables

Swift protocols define requirements that conforming classes, structs, and enums
must satisfy. In a binary, protocol declarations, conformances, and runtime
dispatch are represented by related but distinct structures.

## Mental model for reversing

Keep these structures separate while naming data in IDA:

- **Type metadata** describes one concrete type, such as `Person` or
  `StructTest`. Runtime code uses it for layout-dependent operations.
- A **Value Witness Table (VWT)** describes how to copy, move, destroy, and
  allocate storage for values of a type.
- A **Protocol Descriptor** describes a protocol declaration and its
  requirements, such as `FullyNamed`.
- A **Protocol Conformance Descriptor** records that one concrete type conforms
  to one protocol, such as `Person: FullyNamed`.
- A **Protocol Witness Table (PWT)** provides the implementations used when
  dispatching that conformance through the protocol.

For a protocol existential, such as a value typed as `any FullyNamed`, a useful
simplified picture is:

```text
existential container
  value buffer
  type metadata pointer  ---> metadata for the concrete type
  PWT pointer            ---> witnesses for ConcreteType: Protocol
```

The metadata and PWT pointers answer different questions. Metadata answers
"what concrete value is stored here?" The PWT answers "which implementation
should be called for this protocol requirement?"

See [Existential containers](runtime-types.md#existential-containers) for a
plain-language definition and a source-level example.

## Dispatch patterns to distinguish

Do not assume that every method invocation places an object in `X20` and reads a
PWT from an existential container. Several different lowering patterns can
appear.

### Concrete class method

For a class instance method, `X20` commonly carries the instance as `swiftself`:

```asm
mov x20, x0       ; class instance
bl  class_method
```

This path does not inherently involve a PWT. Class dispatch may be direct,
devirtualized, or use class metadata depending on the call site.

### Protocol existential

For a value stored as `any StructProtocol`, code commonly starts from a pointer
to an existential container:

```asm
ldr x8, [x0, #pwt_offset]
ldr x9, [x8, #requirement_slot]
; Prepare self from the existential value buffer as required by the thunk.
blr x9
```

The existential container stores the PWT for the concrete conformance. A
different concrete value stored as `any StructProtocol` carries a different PWT.
The witness or thunk receives `self` in the form required by the lowered call;
do not assume that this is always the original object pointer in `X20`.

For a common opaque existential with one PWT, these loads are normally
offset-based because the compiler knows the container representation:

```text
container + 0x00  three-word value buffer
container + 0x18  concrete type metadata
container + 0x20  first PWT
```

For protocol compositions, further PWT pointers follow the first one. Do not
hard-code these offsets for class-bounded existentials, existential metatypes,
or special cases such as errors. Those representations differ.

Swift runtime code also exposes representation-aware operations such as
`projectValue` and `getWitnessTable(i)` through existential type metadata. You
may encounter helper calls when the representation is not statically convenient
to inline. In ordinary optimized code for a known existential shape, direct
offset loads are common.

### Generic `T: Protocol`

Generic code does not necessarily construct an existential container:

```swift
func callFirst<T: StructProtocol>(_ value: T) -> Int {
    value.first()
}
```

The generic function commonly receives `value` according to its lowered
representation plus additional runtime arguments for `T`, including type
metadata and the witness table required by the constraint. The PWT may therefore
arrive as a separate generic argument rather than being loaded from an
existential container.

At a specialized or optimized call site, the compiler may know `T` and emit a
direct call instead. Confirm the actual register assignments from the caller and
callee rather than assigning a universal register to the PWT.

## Property and method requirements

Protocols can require properties and methods. For example, `FullyNamed`
requires a readable `String` property:

```swift
protocol FullyNamed {
    var fullName: String { get }
}

struct Person: FullyNamed {
    var fullName: String
}

let john = Person(fullName: "John Appleseed")
```

The conforming type must provide a property with the required name and type. A
`{ get set }` requirement also requires the property to be settable.

A method requirement specifies a callable interface without dictating its
implementation:

```swift
protocol RandomNumberGenerator {
    func random() -> Double
}
```

A type can conform to more than one protocol:

```swift
struct SomeStructure: FirstProtocol, AnotherProtocol {
    // ...
}
```

## Protocol descriptors

Declared protocols are referenced from `__TEXT.__swift5_protos`. A protocol
descriptor records the protocol name and the number of generic-signature and
protocol requirements that follow its fixed fields.

```swift
type TargetProtocolDescriptor struct {
    TargetContextDescriptor
    NameOffset                 RelativeDirectPointer
    NumRequirementsInSignature uint32
    NumRequirements            uint32
    AssociatedTypeNamesOffset  RelativeDirectPointer
}
```

The generic-signature requirements are followed by `NumRequirements` protocol
requirements:

```swift
type TargetGenericRequirementDescriptor struct {
    Flags                                  GenericRequirementFlags
    ParamOff                               RelativeDirectPointer
    TypeOrProtocolOrConformanceOrLayoutOff RelativeIndirectablePointer
}

type TargetProtocolRequirement struct {
    Flags                 ProtocolRequirementFlags
    DefaultImplementation RelativeDirectPointer
}
```

## Protocol extensions

A protocol extension can provide a default implementation for a requirement:

```swift
protocol RandomNumberGenerator {
    func random() -> Double
}

extension RandomNumberGenerator {
    func random() -> Double {
        return 1.0
    }
}
```

Unless a conforming type provides its own implementation of `random()`, calling
the requirement uses the extension's default implementation and returns `1.0`.

## Protocol conformance descriptors

Protocol conformances are referenced from `__TEXT.__swift5_proto`. A conformance
descriptor records which type conforms to which protocol and how its witness
table is obtained.

```swift
type ProtocolConformanceDescriptor struct {
    ProtocolDescriptor    int32 // relative reference
    NominalTypeDescriptor int32 // relative reference
    ProtocolWitnessTable  int32 // relative reference
    ConformanceFlags      uint32
}
```

Depending on `ConformanceFlags`, trailing records may include a retroactive
context reference, resilient witness records, and a generic witness-table
record. The [practical example](practical-example.md) walks through a real
descriptor in IDA.

## Protocol Witness Tables

Protocol methods are dispatched through Protocol Witness Tables (PWTs). When
calling a protocol requirement through an existential value, Swift looks up the
appropriate function pointer in the conforming type's PWT.

A simplified dispatch sequence may resemble:

```asm
; x0 points to an existential container.
ldr x8, [x0, #pwt_offset]
ldr x9, [x8, #requirement_slot]
blr x9
```

The selected PWT belongs to one concrete conformance, such as
`StructTest: StructProtocol`. If a second type conforms to the same protocol,
its existential container contains a different PWT pointer. Calling the same
slot therefore reaches a different implementation.

Property requirements also produce witness entries. For `FullyNamed`, a call
through the protocol uses the witness for the `fullName` getter supplied by the
conforming type.

The exact offsets depend on the existential shape, protocol requirements,
resilience, and compiler output. In optimized code, the compiler may devirtualize
the call or use a thunk, so the loads may not appear literally.

The [library evolution model](https://www.swift.org/blog/library-evolution/)
adds an important caveat across resilience boundaries. Protocol requirements may
be reordered, and new requirements may be added when a default implementation
exists. Client code may therefore call exported dispatch thunks instead of
assuming a fixed witness-table offset.

When the protocol and conformance are defined in the same framework, the
compiler has more static information and may generate a more direct pattern.
Confirm the dispatch sequence at the call site.

## Value Witness Tables

Value Witness Tables (VWTs) define operations needed to manage a value's
storage. Common entries include:

```text
initializeBufferWithCopyOfBuffer
destroy
initializeWithCopy
assignWithCopy
initializeWithTake
assignWithTake
getEnumTagSinglePayload
storeEnumTagSinglePayload
```

VWTs and PWTs serve different purposes:

- The VWT describes storage operations such as copying, moving, and destroying
  values.
- The PWT dispatches protocol requirements for a conforming type.

An existential container stores type metadata and one or more PWT references.
The type metadata provides access to the VWT. See
[Runtime types and storage](runtime-types.md#existential-containers).

When reversing a copy or destruction of an opaque value, you may see a sequence
similar to:

```asm
; x0 points to an existential container.
ldr x8, [x0, #metadata_offset]
ldr x9, [x8, #-8]            ; common metadata-to-VWT relationship
ldr x10, [x9, #witness_slot]
blr x10
```

This is conceptually different from a PWT dispatch. The selected function
manages storage; it does not implement a source-level protocol requirement.
Treat `#-8` and the witness slot as patterns to verify against the metadata kind
and runtime version rather than universal constants.

## Associated types

The `__TEXT.__swift5_assocty` section contains associated type descriptors.
Each descriptor maps associated types to type witnesses for a conformance.

```swift
type AssociatedTypeRecord struct {
    Name                int32
    SubstitutedTypeName int32
}

type AssociatedTypeDescriptor struct {
    ConformingTypeName       int32
    ProtocolTypeName         int32
    NumAssociatedTypes       uint32
    AssociatedTypeRecordSize uint32
    AssociatedTypeRecords    []AssociatedTypeRecord
}
```
