# ARM64 calling-convention notes

The examples in this repository focus on Mach-O ARM64 binaries. Treat the
patterns below as analysis heuristics and confirm register use at each call
site. Compiler version, optimization, type layout, and context can change the
generated code.

## Swift-specific registers

LLVM's
[AArch64 calling-convention definitions](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Target/AArch64/AArch64CallingConvention.td)
assign several registers special roles for Swift calls on Darwin:

- `X20`: `swiftself`. Carries the `self` or context parameter when the function
  has one.
- `X21`: `swifterror`. Carries the Swift error value at call boundaries for
  functions that use Swift error handling.
- `X22`: `swiftasync`. Carries the asynchronous context parameter for async
  functions.

`X19`-`X28` are callee-saved under the
[AAPCS64](https://github.com/ARM-software/abi-aa/blob/main/aapcs64/aapcs64.rst).
Swift assigns additional meaning to selected registers, but only when the
corresponding parameter exists. For example, do not label every use of `X20` as
`self` or every value in `X21` as a thrown error without checking the function
and call site.

These assignments are target-specific. The Windows ARM64 Swift convention uses
different registers for `swifterror` and `swiftasync`. This guide focuses on
Darwin Mach-O binaries.

`swiftself` does not imply a class instance. For a class instance method, `self`
is a single pointer to the instance. For a value-type method, `self` remains a
value and may lower to multiple registers or indirect passing. Closures can use
the same out-of-band mechanism for their context. See
[`Swift::String`](runtime-types.md#swiftstring) for a common reversing pitfall.

## Aggregate returns

Small aggregate values are commonly returned in `X0`-`X3`. Larger values are
commonly returned indirectly through a caller-provided result buffer, often
passed in `X8`.

For example, `Swift::String` has two words in the layouts examined here and is
commonly returned in `X0`-`X1`.

## Generic functions

Swift generic functions receive additional runtime arguments as required by the
generic signature. These may include type metadata and witness tables. They
should not be mistaken for ordinary source-level arguments.

A simplified signature may look like:

```c
// _allocateUninitializedArray<A>(_:)
Swift_ArrayAny *__fastcall _allocateUninitializedArray_A(
    u64 count,
    void *arrayType
);
```

## Variadic arguments and `print`

When calling a variadic function such as `print`, the compiler can use
`_allocateUninitializedArray<A>(_:)` to create an `Array<Any>` and pass it as a
single argument. The included IDA header models this as `Swift_ArrayAny`.

A simplified `print(_:separator:terminator:)` signature is:

```c
void __fastcall print___separator_terminator__(
    Swift_ArrayAny *items,
    Swift_String separator,
    Swift_String terminator
);
```

## Error handling

For Darwin ARM64 Swift calls that use error handling, LLVM associates the
[`swifterror`](https://llvm.org/docs/LangRef.html#parameter-attributes)
parameter with `X21` at call boundaries. `swift_unexpectedError()` may be used
when an error unexpectedly escapes, while `swift_allocError()` allocates an
error object using type metadata.

Confirm the specific error path in the target binary before applying labels.

## Closures and GCD

Closures may be passed to GCD methods such as:

- `OS_dispatch_queue.sync<A>(execute:)`
- `OS_dispatch_queue.sync<A>(flags:execute:)`

A decompiler signature for `OS_dispatch_queue.sync<A>(execute:)` may resemble:

```c
void *__usercall OS_dispatch_queue_sync_A__execute__@<X0>(
    _QWORD *__return_ptr a1@<X8>,
    void *dispatchQueue@<X20>,
    void *callback@<X0>,
    id params@<X1>,
    void *returnType@<X2>
);
```

For this pattern:

- `X8` contains the return buffer.
- `X20` contains the `OS_dispatch_queue` instance.
- `X0` contains the invoked function.
- `X1` contains captured parameters.
- `X2` contains return-type metadata.

Captured parameters are commonly serialized into a stack block. This may
trigger dynamic stack allocation by subtracting the required size from `SP`.
When IDA misses that allocation, open `Edit function (Alt+P)` and increase
`Local Variables Area` by the dynamically allocated stack size.

The invoked function may resemble:

```c
void __usercall callback(id *params@<X20>, void *retval@<X8>);
```
