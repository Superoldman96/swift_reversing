# Practical Swift example

Before starting, please read this: <https://github.com/swiftlang/swift/blob/main/docs/Lexicon.md>.

For this example, we will reverse the `CoreDeviceAction` framework contained in
an iOS Developer Disk Image (DDI). The same process can be applied to other
Swift libraries and binaries.

Load the binary into IDA, then open the Segments view.

Navigate to `__swift5_proto`. As described in the
[metadata guide](metadata-sections.md), this section contains
a list of signed 32-bit relative pointers to **Protocol Conformance
Descriptors**. A descriptor
records a class, struct, or enum's conformance to a protocol and commonly
resides in `__TEXT.__const`:

Before parsing the records, read the
[witness-table mental model](protocols-and-witness-tables.md#mental-model-for-reversing).
It explains the difference between type metadata, conformance descriptors,
VWTs, and PWTs.

```cpp
  /// The Protocol Descriptor being conformed to.
  TargetRelativeContextPointer<Runtime, TargetProtocolDescriptor> Protocol;

  // Some description of the type that conforms to the protocol.
  TargetTypeReference<Runtime> TypeRef;

  /// The witness table pattern, which may also serve as the witness table.
  RelativeDirectPointer<const TargetWitnessTable<Runtime>> WitnessTablePattern;

  /// Various flags, including the kind of conformance.
  ConformanceFlags Flags;
```

![screenshot](../media/proto_notfixed.png)

IDA does not define these relative pointers automatically. Convert each entry to
a double word (`int32_t`), then define its operand type as an offset with these
options:

![screenshot](../media/ida_options_off.png)

You should now have a list of references into `__TEXT.__const`. Each referenced
record is a **Protocol Conformance Descriptor**. Rename the entries as you
identify them to make later analysis easier:

![screenshot](../media/proto_fixed.png)

Follow each reference and define the fixed portion of its **Protocol Conformance
Descriptor**. The fixed portion contains four consecutive 32-bit fields. The
first three fields are relative references and the fourth contains flags.

For example, we'll work with this one:

![screenshot](../media/pcd_notifxed.png)

After defining the first four double words:

![screenshot](../media/pcd_wip.png)

In this example:

```text
`off_28B10` is the relative pointer to the `Protocol Descriptor`. Following it
leads to the descriptor for `_HasCustomAnyHashableRepresentation`.

`$s3__C19ProgressUserInfoKeyVMn` is the type reference. In this case, it refers
to a nominal type descriptor.

`0` is the `WitnessTablePattern` field. This conformance does not provide a
statically emitted witness table pattern.

`0x300c0` contains the conformance flags.
```

The record is larger than four words because the flags indicate that trailing
objects are present. See Swift's
[`ProtocolConformanceFlags`](https://github.com/swiftlang/swift/blob/main/include/swift/ABI/MetadataValues.h)
definition for the complete set of masks.

> **NOTE:** For a parser implementation, see Blacktop's
> [`readProtocolConformance`](https://github.com/blacktop/go-macho/search?q=readProtocolConformance)
> function.

`0x300c0` includes:

`IsRetroactiveMask = 0x01u << 6`

`IsSynthesizedNonUniqueMask = 0x01u << 7`

`HasResilientWitnessesMask = 0x01u << 16`

`HasGenericWitnessTableMask = 0x01u << 17`

![screenshot](../media/proto_parsed.png)

Because `IsRetroactiveMask` is set, the first trailing double word is the
retroactive context reference.

Because `HasResilientWitnessesMask` is set, the resilient-witness header follows.
Its `NumWitnesses` value determines how many resilient witness records follow.

Each resilient witness record contains two relative references: one to a
protocol requirement and one to its implementation.

For instance:

![screenshot](../media/pcd_renamed.png)

```text
off_28b48 points to the base conformance descriptor for _SwiftNewTypeWrapper: RawRepresentable.

unk_243c3 points into the typeref section, which contains the associated
conformance reference
associated_conformance_So21NSProgressUserInfoKeyas20_SwiftNewtypeWrapperSCSY.

off_28b50 points to the base conformance descriptor for _SwiftNewtypeWrapper: _HasCustomAnyHashableRepresentation.

unk_243cb points to associated_conformance_So21NSProgressInfoKeyas20_SwiftNewTypeWrapperSCs35_HasCustomAnyHashableRepresentation.
```

Because `HasGenericWitnessTableMask` is set, a generic witness-table record
appears after the resilient witnesses:

```text
struct GenericWitnessTable {
    WitnessTableSizeInWords                                uint16
    WitnessTablePrivateSizeInWordsAndRequiresInstantiation uint16
    Instantiator                                           int32
    PrivateData                                            int32
}
```

![screenshot](../media/pcd_renamed.png)

Do it with all Protocol Conformance Descriptors before proceeding.

**Once you are done, parse the `__swift5_assocty` section.**

Open `__swift5_assocty` from IDA's Segments view. This section is an array of
associated type descriptors. Each descriptor contains associated type records
for a conformance. Each record maps an associated type to the conformance's type
witness.

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

![screenshot](../media/assocty.png)

## Reversing Action.invoke

Now that the relevant sections have useful names and definitions, the remaining
analysis is easier to follow.

Navigate to `static Action.invoke(usingContentsOf:)(__int64 a1, char* a2,
__int64 a3)`. Unlike Objective-C message sends, this Swift call does not use a
selector.

The first question is: **Where is this defined in the binary?**

`Action` is a **Protocol Descriptor** that defines several method and property
requirements. In this case, `invoke` is a method requirement with a default
implementation supplied by an extension. Following cross-references leads back
to the protocol descriptor definition.
