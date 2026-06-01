# Swift metadata sections

Read Swift's [lexicon](https://github.com/swiftlang/swift/blob/main/docs/Lexicon.md)
before working through this guide.

## Relative pointers

Swift metadata makes extensive use of relative pointers. Instead of storing an
absolute address, a field stores a signed 32-bit offset relative to its own
address. This reduces relocation work during dynamic linking and the space
required for many metadata references.

The implementation is documented in
[`RelativePointer.h`](https://github.com/swiftlang/swift/blob/main/include/swift/Basic/RelativePointer.h).
As simplified pseudocode, a direct relative pointer is resolved like this:

```c
dstAddress = fieldAddress + (int32)offset;
```

When analyzing a Mach-O binary that uses the Swift runtime, you will usually
find several `__swift5_*` sections. These sections refer to metadata records,
many of which reside in `__TEXT.__const`.

## `__TEXT.__swift5_protos`

Contains a list of relative pointers to **Protocol Descriptors**. Each
descriptor represents a declared Swift protocol and commonly resides in
`__TEXT.__const`.

The implementation is defined in
[`Metadata.h`](https://github.com/swiftlang/swift/blob/main/include/swift/ABI/Metadata.h).
A simplified layout is:

```swift
type ProtocolDescriptor struct {
    Flags                      ContextDescriptorFlags
    Parent                     TargetRelativeContextPointer
    Name                       TargetRelativeDirectPointer
    NumRequirementsInSignature uint32
    NumRequirements            uint32
    AssociatedTypeNames        RelativeDirectPointer
}
```

See [Protocols and witness tables](protocols-and-witness-tables.md) for the
requirement records that follow this fixed layout.

## `__TEXT.__swift5_proto`

Contains a list of relative pointers to **Protocol Conformance Descriptors**.
Each descriptor records a type's conformance to a protocol and commonly resides
in `__TEXT.__const`.

A parser script is available at
[`fix_proto_conf_desc.py`](https://github.com/doronz88/ida-scripts/blob/main/fix_proto_conf_desc.py).
The fixed part of a conformance descriptor is:

```c
/// The Protocol Descriptor being conformed to.
TargetRelativeContextPointer<Runtime, TargetProtocolDescriptor> Protocol;

// Some description of the type that conforms to the protocol.
TargetTypeReference<Runtime> TypeRef;

// The witness table pattern, which may also serve as the witness table.
RelativeDirectPointer<const TargetWitnessTable<Runtime>> WitnessTablePattern;

// Various flags, including the kind of conformance.
ConformanceFlags Flags;
```

The first three fields are relative references. The fourth contains flags.
Trailing objects may follow depending on those flags. See the
[practical example](practical-example.md) for a worked IDA walkthrough.

## `__TEXT.__swift5_types`

Contains relative pointers to type context descriptors. Several descriptor
kinds have a common prefix, but their trailing fields differ. Inspect the
descriptor flags before deciding which layout to apply.

```swift
type EnumDescriptor struct {
    Flags                               uint32
    Parent                              int32
    Name                                int32
    AccessFunction                      int32
    FieldDescriptor                     int32
    NumPayloadCasesAndPayloadSizeOffset uint32
    NumEmptyCases                       uint32
}

type StructDescriptor struct {
    Flags                   uint32
    Parent                  int32
    Name                    int32
    AccessFunction          int32
    FieldDescriptor         int32
    NumFields               uint32
    FieldOffsetVectorOffset uint32
}

type ClassDescriptor struct {
    Flags                       uint32
    Parent                      int32
    Name                        int32
    AccessFunction              int32
    FieldDescriptor             int32
    SuperclassType              int32
    MetadataNegativeSizeInWords uint32
    MetadataPositiveSizeInWords uint32
    NumImmediateMembers         uint32
    NumFields                   uint32
}
```

Other possible kinds include `TargetExtensionContextDescriptor`,
`TargetAnonymousContextDescriptor`, and `TargetOpaqueTypeDescriptor`.

## `__TEXT.__swift5_typeref`

Contains symbolic references used by the runtime for instantiation and
reflection. For example, the first argument to functions such as
`swift_instantiateConcreteTypeFromMangledName` may refer to data containing a
relative pointer into `__swift5_typeref`.

A type can contain other types, so symbolic reference definitions may be
concatenated. This area is still a work in progress: confirm the encoding
against the runtime version used by the target.

```text
switch(first_byte):
  case 0x01:
    Direct reference to a context type descriptor
    1 byte - type (0x01)
    4 bytes - relative pointer

  case 0x02:
    Indirect reference to a context type descriptor
    1 byte - type (0x02)
    4 bytes - relative pointer

  case 0xFF:
    1 byte - type (0xFF)
    1 byte - discriminator
    4 bytes - relative pointer to the metadata access function

  case 0x53 ('S'):
    standard-substitutions ::= 'S' KNOWN-TYPE-KIND
```

References are terminated by a null byte. Consult Swift's mangling
documentation for the complete substitution grammar.

## `__TEXT.__swift5_fieldmd`

Contains an array of field descriptors. A field descriptor contains field
records for a single class, struct, or enum declaration. Its length depends on
the number of records.

```swift
type FieldRecord struct {
    Flags           uint32
    MangledTypeName int32
    FieldName       int32
}

type FieldDescriptor struct {
    MangledTypeName int32
    Superclass      int32
    Kind            uint16
    FieldRecordSize uint16
    NumFields       uint32
    FieldRecords    []FieldRecord
}
```

## When names and layouts are visible

The presence of field names and enum case names is useful during reversing, but
it is not guaranteed. Reflection metadata controls what you can recover by
parsing metadata. Library evolution separately controls which layout assumptions
compiled clients may make across a framework boundary.

Xcode's
[`SWIFT_REFLECTION_METADATA_LEVEL`](https://developer.apple.com/documentation/xcode/build-settings-reference#Reflection-Metadata-Level)
setting controls reflection metadata emission:

- `All` emits type information and names for stored properties of Swift structs
  and classes, as well as Swift enum cases.
- `Without Names` emits type information for stored properties and enum cases,
  but omits their names. This corresponds to `-disable-reflection-names`.
- `None` omits reflection metadata. This corresponds to
  `-disable-reflection-metadata`.

For reversing, this means `__swift5_fieldmd` may expose stored-property names and
enum case names, expose records without useful names, or provide little useful
information for a type. Missing names do not prove that a type has no fields or
cases. Names may still appear elsewhere as symbols or string literals for
unrelated reasons.

[Library evolution](https://www.swift.org/blog/library-evolution/) controls
which layout assumptions are valid across a framework boundary:

- A non-`@frozen` struct or enum has an opaque layout across a resilience
  boundary. Client code must defer layout-dependent operations to runtime
  metadata and accessors.
- An `@frozen` struct publishes its stored-property layout to clients.
- An `@frozen` enum promises that cases will not be added, removed, or reordered.
  Client code may switch over its known cases exhaustively.
- A non-`@frozen` enum may gain new cases. Client code compiled across the
  resilience boundary must account for an unknown case.

Library evolution is disabled by default. Even when it is enabled, code within
the framework itself has full knowledge of its own types. Treat visible metadata
as evidence about the analyzed build, not as proof that a layout is stable
across releases.
