# Reversing Swift

This repository is a practical introduction to reversing Swift binaries in IDA.
It explains how common Swift concepts appear in compiled binaries and
highlights metadata structures that are useful during analysis.

The examples focus on Mach-O binaries and ARM64 calling conventions. Swift ABI
details can vary by compiler version and target platform, so treat the layouts
as starting points and confirm them against the binary you are analyzing.

## Start here

Read the guides in this order:

1. [Swift metadata sections](docs/metadata-sections.md)
2. [Runtime types and storage](docs/runtime-types.md)
3. [Protocols and witness tables](docs/protocols-and-witness-tables.md)
4. [ARM64 calling-convention notes](docs/calling-conventions.md)
5. [Practical Swift example](docs/practical-example.md)

The included [`ida_header.h`](ida_header.h) defines helper types that can be
imported into IDA.

## IDA helper script

Run [`swift.py`](https://github.com/doronz88/ida-scripts/blob/main/swift.py) in
IDA with `Alt+F7` to assist with analysis. The script adds the `Ctrl+5` hotkey to
parse `Swift::String` occurrences within the current function.

> **NOTE:** The script is a work in progress. Please submit a PR if you find a
> missing type or an unsupported pattern.

## Important caveats

- Visible metadata describes the analyzed build. It does not necessarily prove
  that a type's layout is stable across releases.
- Stored-property and enum-case names may be omitted depending on the target's
  reflection metadata settings.
- Register use depends on ABI details and generated code. Confirm assumptions at
  each call site.

## References

- [Swift documentation](https://www.swift.org/documentation/)
- [Swift library evolution](https://www.swift.org/blog/library-evolution/)
- [Swift ABI documentation](https://github.com/swiftlang/swift/tree/main/docs/ABI)
- [Swift lexicon](https://github.com/swiftlang/swift/blob/main/docs/Lexicon.md)
- [Swift protocol documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/)
- [Scott Knight: Exploring Swift Memory Layout](https://knight.sc/reverse%20engineering/2019/07/17/swift-metadata.html)
- [Hex-Rays: Custom calling conventions](https://hex-rays.com/blog/igors-tip-of-the-week-51-custom-calling-conventions/)
- [Blacktop go-macho Swift parser](https://github.com/blacktop/go-macho/blob/master/swift.go)
