# Layout of Rust `enum` types

**Disclaimer:** Some parts of this section were decided in RFCs, but
others represent the consensus from issue [#10]. The text will attempt
to clarify which parts are "guaranteed" (owing to the RFC decision)
and which parts are still in a "preliminary" state, at least until we
start to open RFCs ratifying parts of the Unsafe Code Guidelines
effort.

**Note:** This document has not yet been updated to
[RFC 2645](https://github.com/rust-lang/rfcs/blob/master/text/2645-transparent-unions.md).

[#10]: https://github.com/rust-rfcs/unsafe-code-guidelines/issues/10

## Categories of enums

**Empty enums.** Enums with no variants can never be instantiated and
are equivalent to the `!` type. They do not accept any `#[repr]`
annotations.

**Fieldless enums.** The simplest form of enum is one where none of
the variants have any fields:

```rust
enum SomeEnum {
  Variant1,
  Variant2,
  Variant3,
}
```

Such enums correspond quite closely with enums in the C language
(though there are important differences as well). Presuming that they
have more than one variant, these sorts of enums are always
represented as a simple integer, though the size will vary.

Fieldless enums may also specify the value of their discriminants
explicitly:

```rust
enum SomeEnum {
  Variant22 = 22,
  Variant44 = 44,
  Variant45,
}
```

As in C, discriminant values that are not specified are defined as
either 0 (for the first variant) or as one more than the prior
variant.

**Data-carrying enums.** Enums with at least one variant with fields are called
"data-carrying" enums. Note that for the purposes of this definition, it is not
relevant whether the variant fields are zero-sized. Therefore this enum is
considered "data-carrying":

```rust
enum Foo {
  Bar(()),
  Baz,
}
```

## repr annotations accepted on enums

In general, enums may be annotated using the following `#[repr]` tags:

- A specific integer type (called `Int` as a shorthand below):
  - `#[repr(u8)]`
  - `#[repr(u16)]`
  - `#[repr(u32)]`
  - `#[repr(u64)]`
  - `#[repr(i8)]`
  - `#[repr(i16)]`
  - `#[repr(i32)]`
  - `#[repr(i64)]`
- C-compatible layout:
  - `#[repr(C)]`
- C-compatible layout with a specified discriminant size:
  - `#[repr(C, u8)]`
  - `#[repr(C, u16)]`
  - etc

Note that manually specifying the alignment using `#[repr(align)]` is
not permitted on an enum.

The set of repr annotations accepted by an enum depends on its category,
as defined above:

- Empty enums: no repr annotations are permitted.
- Fieldless enums: `#[repr(Int)]`-style and `#[repr(C)]` annotations are permitted, but `#[repr(C, Int)]` annotations are not.
- Data-carrying enums: all repr annotations are permitted.

## Enum layout rules

The rules for enum layout vary depending on the category.

### Layout of an empty enum

An **empty enum** is an enum with no variants; empty enums can never
be instantiated and are logically equivalent to the "never type"
`!`. `#[repr]` annotations are not accepted on empty enums. Empty
enums are guaranteed to have the same layout as `!` (zero size and
alignment 1).

### Layout of a fieldless enum

If there is no `#[repr]` attached to a fieldless enum, the compiler
will represent it using an integer of sufficient size to store the
discriminants for all possible variants -- note that if there is only
one variant, then 0 bits are required, so it is possible that the enum
may have zero size. In the absence of a `#[repr]` annotation, the
number of bits used by the compiler are not defined and are subject to
change.

When a `#[repr(Int)]`-style annotation is attached to a fieldless enum
(one without any data for its variants), it will cause the enum to be
represented as a simple integer of the specified size `Int`. This must
be sufficient to store all the required discriminant values.

The `#[repr(C)]` annotation is equivalent, but it selects the same
size as the C compiler would use for the given target for an
equivalent C-enum declaration.

Combining a `C` and `Int` `repr` (e.g., `#[repr(C, u8)]`) is
not permitted on a fieldless enum.

The values used for the discriminant will match up with what is
specified (or automatically assigned) in the enum definition. For
example, the following enum defines the discriminants for its variants
as 22 and 23 respectively:

```rust
enum Foo { 
  // Specificy discriminant of this variant as 22:
  Variant22 = 22,
  
  // Default discriminant is one more than the previous,
  // so 23 will be assigned.
  Variant23
}
```

**Note:** some C compilers offer flags (e.g., `-fshort-enums`) that
change the layout of enums from the default settings that are standard
for the platform. The integer size selected by `#[repr(C)]` is defined
to match the **default** settings for a given target, when no such
flags are supplied. If interop with code that uses other flags is
desired, then one should either specify the sizes of enums manually or
else use an alternate target definition that is tailored to the
compiler flags in use.

### Layout of a data-carrying enums with an explicit repr annotation

This section concerns data-carrying enums **with an explicit repr
annotation of some form**. The memory layout of such cases was
specified in [RFC 2195][] and is therefore normative.

[RFC 2195]: https://rust-lang.github.io/rfcs/2195-really-tagged-unions.html

The layout of data-carrying enums that do **not** have an explicit
repr annotation is generally undefined, but with certain specific
exceptions: see the next section for details.

#### Explicit repr annotation without C compatibility

When an enum is tagged with `#[repr(Int)]` for some integral type
`Int` (e.g., `#[repr(u8)]`), it will be represented as a C-union of a
series of `#[repr(C)]` structs, one per variant. Each of these structs
begins with an integral field containing the **discriminant**, which
specifies which variant is active. They then contain the remaining
fields associated with that variant.

**Example.** The following enum uses an `repr(u8)` annotation:

```rust
#[repr(u8)]
enum TwoCases {
    A(u8, u16),
    B(u16),
}
```

This will be laid out equivalently to the following more 
complex Rust types:

```rust
#[repr(C)]
union TwoCasesRepr {
    A: TwoCasesVariantA,
    B: TwoCasesVariantB,
}
        
# #[derive(Copy, Clone)]        
#[repr(u8)]
enum TwoCasesTag { A, B }

# #[derive(Copy, Clone)]
#[repr(C)]
struct TwoCasesVariantA(TwoCasesTag, u8, u16);

# #[derive(Copy, Clone)]
#[repr(C)]
struct TwoCasesVariantB(TwoCasesTag, u16);
```

Note that the `TwoCasesVariantA` and `TwoCasesVariantB` structs are
`#[repr(C)]`; this is needed to ensure that the `TwoCasesTag` value
appears at offset 0 in both cases, so that we can read it to determine
the current variant.
        
#### Explicit repr annotation with C compatibility

When the `#[repr]` tag includes `C`, e.g., `#[repr(C)]` or `#[repr(C,
u8)]`, the layout of enums is changed to better match C++ enums. In
this mode, the data is laid out as a tuple of `(discriminant, union)`,
where `union` represents a C union of all the possible variants. The
type of the discriminant will be the integral type specified (`u8`,
etc) -- if no type is specified, then the compiler will select one
based on what a size a fieldless enum would have with the same number of
variants.

This layout, while more compatible and arguably more obvious, is also
less efficient than the non-C compatible layout in some cases in terms
of total size. For example, the `TwoCases` example given in the
previous section only occupies 4 bytes with `#[repr(u8)]`, but would
occupy 6 bytes with `#[repr(C, u8)]`, as more padding is required.

**Example.** The following enum:

```rust,ignore
#[repr(C, Int)]
enum MyEnum {
    A(u32),
    B(f32, u64),
    C { x: u32, y: u8 },
    D,
}
```

is equivalent to the following Rust definition:

```rust,ignore
#[repr(C)]
struct MyEnumRepr {
    tag: MyEnumTag,
    payload: MyEnumPayload,
}

#[repr(Int)]
enum MyEnumTag { A, B, C, D }

#[repr(C)]
union MyEnumPayload {
   A: u32,
   B: MyEnumPayloadB,
   C: MyEnumPayloadC,
   D: (),
}

#[repr(C)]
struct MyEnumPayloadB(f32, u64);

#[repr(C)]
struct MyEnumPayloadC { x: u32, y: u8 }
```

This enum can also be represented in C++ as follows:

```c++
#include <stdint.h>

enum class MyEnumTag: CppEquivalentOfInt { A, B, C, D };
struct MyEnumPayloadB { float _0; uint64_t _1;  };
struct MyEnumPayloadC { uint32_t x; uint8_t y; };

union MyEnumPayload {
   uint32_t A;
   MyEnumPayloadB B;
   MyEnumPayloadC C;
};

struct MyEnum {
    MyEnumTag tag;
    MyEnumPayload payload;
};
```

### Layout of a data-carrying enums without a repr annotation

If no explicit `#[repr]` attribute is used, then the layout of a
data-carrying enum is typically **not specified**. However, in certain
select cases, there are **guaranteed layout optimizations** that may
apply, as described below.

#### Discriminant elision on Option-like enums

(Meta-note: The content in this section have been turned into stable guarantees
[via this
FCP](https://github.com/rust-lang/rust/pull/130628#issuecomment-2402761599).).

**Definition.** The fully monomorphized form of a 2-variant `enum` is called an **option-like enum**
if all of the following are satisfied:

- the `enum` has no explicit `#[repr(...)]`, and
- one variant has a single field with a type that guarantees discriminant
  elision (to be defined below), and
- the other variant has only 1-ZST fields (the "unit variant").

The simplest example is `Option<T>` itself, where the `Some` variant
has a single field (of type `T`), and the `None` variant has no
fields. But other enums that fit that same template also qualify, e.g.
`Result<T, ()>` or `Result<(), T>`.

**Definition.** The **payload** of an option-like enum is the single
field which it contains; in the case of `Option<T>`, the payload has
type `T`.

**Definition.** The following payload types have guaranteed discriminant
elision:

* `&T`
* `&mut T`
* `Box<T>`
* `extern "ABI" fn` (for arbitrary "ABI")
* `core::num::NonZero*`
* `core::ptr::NonNull<T>`
* `#[repr(transparent)] struct` around one of the types in this list.

(Meta-note: all these types have at least one bit pattern that is guaranteed be
invalid, and can therefore be used as a "[niche]" when computing the enum layout.
More types have this property, but the *guarantee* described here only applies
to the types listed here.)

**Option-like enums where the payload has guaranteed discriminant elision
are guaranteed to be represented using the same memory layout as their
payload.** This is called **discriminant elision**, as there is no
explicit discriminant value stored anywhere. Instead, [niche] values are
used to represent the unit variant.

The most common example is that `Option<&u8>` can be represented as an
nullable `&u8` reference -- the `None` variant is then represented
using the [niche] value zero. This is because a valid `&u8` value can
never be zero, so if we see a zero value, we know that this must be
`None` variant.

**Example.** The type `Option<&u32>` will be represented at runtime as
a nullable pointer. FFI interop often depends on this property.

**Example.** As `fn` types are non-nullable, the type `Option<extern
"C" fn()>` will be represented at runtime as a nullable function
pointer (which is therefore equivalent to a C function pointer) . FFI
interop often depends on this property.

**Example.** The following enum definition is **not** option-like,
as it has two unit variants:

```rust
enum Enum1<T> {
  Present(T),
  Absent1,
  Absent2,
}
```

**Example.** The following enum definition is **not** option-like,
as it has an explicit `repr` attribute.

```rust
#[repr(u8)]
enum Enum2<T> {
  Present(T),
  Absent1,
}
```

[niche]: ../glossary.md#niche

### Layout of enums with a single variant

> **NOTE**: the guarantees in this section have not been approved by an RFC process.

**Data-carrying** enums with a single variant without a `repr()` annotation have
the same layout as the variant field. **Fieldless** enums with a single variant
have the same layout as a unit struct. 

For example, here:

```rust
struct UnitStruct;
enum FieldlessSingleVariant { FieldlessVariant }

struct SomeStruct { x: u32 }
enum DataCarryingSingleVariant {
  DataCarryingVariant(SomeStruct),
}
```

* `FieldSingleVariant` has the same layout as `UnitStruct`,
* `DataCarryingSingleVariant` has the same layout as `SomeStruct`.

## Unresolved questions

See [Issue #79.](https://github.com/rust-rfcs/unsafe-code-guidelines/issues/79):
  
* Layout of multi-variant enums where only one variant is inhabited.
