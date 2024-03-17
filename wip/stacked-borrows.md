# Stacked Borrows

**Note:** This document is not normative nor endorsed by the UCG WG.  It is maintained by @RalfJung to reflect what is currently implemented in [Miri].

This is not a guide!
It is more of a reference.
For more background, see the [paper on Stacked Borrows](https://plv.mpi-sws.org/rustbelt/stacked-borrows/).

Stacked Borrows is also the subject of the following blog-posts:

* [Stacked Borrows 0.1](https://www.ralfj.de/blog/2018/08/07/stacked-borrows.html) is the initial idea of what Stacked Borrows might look like before anything got implemented. This post is interesting for some of the historical context it gives, but is largely superseded by the next post.
* [Stacked Borrows 1.0](https://www.ralfj.de/blog/2018/11/16/stacked-borrows-implementation.html) is the first version that got implemented. This post is a self-contained, improved introduction to Stacked Borrows.
* [Stacked Borrows 1.1](https://www.ralfj.de/blog/2018/12/26/stacked-borrows-barriers.html) extends Stacked Borrows 1 with partial support for two-phase borrows and explains the idea of "barriers".
* We took some notes when [discussing Stacked Borrows 1.1 at the 2019 Rust All-Hands][all-hands].
* [Stacked Borrows 2.0](https://www.ralfj.de/blog/2019/04/30/stacked-borrows-2.html) is a re-design of Stacked Borrows 1 that maintains the original core ideas, but changes the mechanism to support more precise tracking of shared references.
* [Stacked Borrows 2.1](https://www.ralfj.de/blog/2019/05/21/stacked-borrows-2.1.html) slightly tweaks the rules for read and write accesses and describes a high-level way of thinking about the new shape of the "stack" in Stacked Borrows 2.

Changes from to the latest post (2.1) to the paper:

* Retags are "shallow" instead of recursively looking for references inside compound types.
* Reborrowing of a shared reference, when searching for `UnsafeCell`, no longer reads enum discriminants. It treats enums like unions now.

Changes since publication of the paper:

* Items with `SharedReadWrite` permission are not protected even with `FnEntry` retagging.
* Removal of `Untagged`.
* Addition of "weak protectors" to justify `noalias` on `Box` parameters.

[Miri]: https://github.com/solson/miri/
[all-hands]: https://paper.dropbox.com/doc/Topic-Stacked-borrows--AXAkoFfUGViWL_PaSryqKK~hAg-2q57v4UM7cIkxCq9PQc22

## Extra state

Stacked Borrows adds some extra state to the Rust abstract machine.
Every pointer value has a *tag* (in addition to the location in memory that the pointer points to), and every memory location carries a *stack* (in addition to the byte of data stored at that location).
Moreover, there is a per-call-frame `CallId` as well as some global tracking state.

```rust
/// PtrId: uniquely identifying a pointer.
// `nat` is the type of mathematical natural numbers, meaning we don't want to think about overflow.
// NOTE: Miri just uses `NonZeroU64` which, realistically, will not overflow because we only ever increment by 1.
type PtrId = nat;
/// For historic reasons we often call pointer IDs "tags".
type Tag = PtrId;

/// CallId: uniquely identifying a stack frame.
type CallId = nat;

/// Indicates which permission is granted (by this item to some pointers)
pub enum Permission {
    /// Grants unique mutable access.
    Unique,
    /// Grants shared mutable access.
    SharedReadWrite,
    /// Grants shared read-only access.
    SharedReadOnly,
    /// Grants no access, but separates two groups of SharedReadWrite so they are not
    /// all considered mutually compatible.
    Disabled,
}

/// The flavor of the protector.
enum ProtectorKind {
    /// Protected against aliasing violations from other pointers.
    ///
    /// Items protected like this cause UB when they are invalidated, *but* the pointer itself may
    /// still be used to issue a deallocation.
    ///
    /// This is required for LLVM IR pointers that are `noalias` but *not* `dereferenceable`.
    WeakProtector,

    /// Protected against any kind of invalidation.
    ///
    /// Items protected like this cause UB when they are invalidated or the memory is deallocated.
    /// This is strictly stronger protection than `WeakProtector`.
    ///
    /// This is required for LLVM IR pointers that are `dereferenceable` (and also allows `noalias`).
    StrongProtector,
}

/// An item on the per-location stack, controlling which pointers may access this location and how.
pub struct Item {
    /// The permission this item grants.
    perm: Permission,
    /// The pointers the permission is granted to.
    tag: Tag,
    /// An optional protector, ensuring the item cannot get popped until `CallId` is over.
    protector: Option<(ProtectorKind, CallId)>,
}

/// Per-location stack of borrow items.
pub struct Stack {
    /// Used *mostly* as a stack; never empty.
    /// Invariants:
    /// * Above a `SharedReadOnly` there can only be more `SharedReadOnly`.
    /// * No tag occurs in the stack more than once.
    borrows: Vec<Item>,
}

/// Extra per-call-frame state: the ID of this function call.
pub struct FrameExtra {
    id: CallId,
}

/// Extra global state: the next `PtrId`, as well as the next `CallId`.
/// Both are just monotonically increasing counters, ensuring they are unique.
/// Also see the code for `new_ptr` and `new_call` below.
pub struct Tracking {
    next_ptr_id: PtrId,
    next_call_id: CallId,
}
```

The tag and the stack exist separately, i.e., when a pointer is stored in memory, then we both have a tag stored as part of this pointer value (remember, [bytes are more than `u8`](https://www.ralfj.de/blog/2018/07/24/pointers-and-bytes.html)), and every byte occupied by the pointer has a stack regulating access to this location.
Also these two do not interact, i.e., when loading a pointer from memory, we just load the tag that was stored as part of this pointer.
The stack of a location, and the tag of a pointer stored at some location, do not have any effect on each other.

## Retag statement

Stacked Borrows introduces a new operation (a new MIR statement) on the Rust abstract machine.
*Retagging* operates on a place (Rust's equivalent of a C lvalue; see [the glossary](../reference/src/glossary.md)), and it also carries a flag indicating the kind of retag that is being performed:

```rust
pub enum RetagKind {
    /// The initial retag when entering a function
    FnEntry,
    /// Retag preparing for a two-phase borrow
    TwoPhase,
    /// Retagging raw pointers
    Raw,
    /// A "normal" retag
    Default,
}
```

`Retag` is inserted into the MIR for the following situations:

* A retag happens after every assignment MIR statement where the assigned type may be of reference or box type.
  This is usually a `Default` retag.  However, if the RHS of this assignment is a `Ref` which allows two-phase borrows, then this is a `TwoPhase` retag.

    Currently, if the LHS of the assignment involves a `Deref`, no `Retag` is inserted.
    That's just a limitation of the current implementation: after executing this assignment, evaluating the place (the LHS) again could yield a different location in memory, which means we would retag the wrong thing.
    Proper retagging here requires either a copy through a temporary, or making retagging integral part of the semantics of assignment.

* A `Raw` retag happens after every assignment where the RHS is a cast from a reference to a raw pointer.

* A `FnEntry` retag happens in the first basic block of every function, retagging each argument.

* A `Default` retag happens on the return value of every function that gets called (i.e., this is the first statement in the basic block that the call will return to).

* The automatically generated drop shims (generated as the body of `ptr::real_drop_in_place`) perform a `Raw` retag of their argument because they use it as a raw pointer.

## Operational semantics

### Generating IDs

Whenever we need to generate a new pointer or function call ID, that means we effectively call one of the following methods:

```rust
impl Tracking {
    fn new_ptr(&mut self) -> PtrId {
        let val = self.next_ptr_id;
        self.next_ptr_id += 1;
        val
    }
    
    fn new_call(&mut self) -> CallId {
        let id = self.next_call_id;
        self.next_call_id += 1;
        id
    }
}
```

These methods will never return the same value twice.

### Tracking function calls

To attach metadata to a particular function call, we assign a fresh ID to every call stack (so this distinguishes multiple calls to the same function).
In other words, the per-stack-frame `CallId` is initialized by `Tracking::new_call`.

We say that a `CallId` is *active* if the call stack contains a stack frame with that ID.

**Note**: Miri uses a slightly more complex system to track the set of active `CallId`; that is just an optimization to avoid having to scan the call stack all the time.

### Preliminaries for items

For brevity, we will write `(tag: perm)` to represent `Item { tag, perm, protector: None }`, and `(tag: perm; kind, call)` to represent `Item { tag, perm, protector: Some((kind, call)) }`.

The following defines whether a permission grants a particular kind of memory access to a pointer with the right tag:
`Unique` and `SharedReadWrite` grant all accesses, `SharedReadOnly` grants only read access.

```rust
pub enum AccessKind {
    Read,
    Write,
}

/// This defines for a given permission, whether it permits the given kind of access.
fn grants(self: Permission, access: AccessKind) -> bool {
    // Disabled grants nothing. Otherwise, all items grant read access, and except for SharedReadOnly they grant write access.
    self != Permission::Disabled && (access == AccessKind::Read || self != Permission::SharedReadOnly)
}
```

Based on this, we define the *granting* item in a stack (for a given tag and access) to be the topmost item that grants the given access to this tag:

```rust
/// Find the item granting the given kind of access to the given tag, and return where that item is in the stack.
fn find_granting(self: &Stack, access: AccessKind, tag: Tag) -> Option<usize> {
    self.borrows.iter()
        .enumerate() // we also need to know *where* in the stack
        .rev() // search top-to-bottom
        // Return permission of first item that grants access.
        .find_map(|(idx, item)|
            if item.perm.grants(access) && tag == item.tag {
                Some(idx)
            } else {
                None
            }
        )
}
```

In general, the structure of the stack looks as follows:
On the top, we might have a bunch of `SharedReadOnly` items. Below that, we have "blocks" consisting of either a single `Unique` item, or a bunch of consecutive `SharedReadWrite`.
`Disabled` items serve to separate two blocks of `SharedReadWrite` that would otherwise be considered one block.
Using any item within a block is equivalent to using any other item in that same block.

### Allocating memory

When allocating memory, we have to initialize the `Stack` associated with the new locations, and we have to choose a `Tag` for the initial pointer to this memory.

- Stack memory is handled by an environment (which is part of the information carried in a stack frame of the Rust abstract machine) that maps each local variable to a place.
  A place is a pointer together with some other data that is not relevant here -- the key point is that a place, just like every other pointer, carries a tag.
  When the local variable becomes live and its backing memory gets allocated, we generate a new pointer ID `id` by calling `Tracking::new_ptr` and use `id` as tag for the place of this local variable.
  We also initialize the stack of all the memory locations in this new memory allocation with `Stack { borrows: vec![(id: Unique)] }`.
- For heap allocations, we pick a fresh `id` at allocation time. The stack of each freshly allocated memory location is `Stack { borrows: vec![(id: SharedReadWrite)] }`, and the initial pointer to that memory has tag `id`.
- For global allocations (`static`, environment and program argument data, ...), we pick a fresh `id` associated with the global, and each time a pointer to the global is created, it gets tagged `id`.
  The stacks in that memory are initialized with `Stack { borrows: vec![(id: SharedReadWrite)] }`.

### Accessing memory

On every memory access (reads and writes -- see below for deallocation),
we perform the following extra operation for every location that gets accessed (i.e., for a 4-byte access, this happens for each of the 4 bytes):

1. Find the granting item. If there is none, this is UB.
2. Check if this is a read access or a write access.
    - For write accesses, pop all *blocks* above the one containing the granting item. That is, remove all items above the granting one, except if the granting item is a `SharedReadWrite` in which case the consecutive `SharedReadWrite` above it are kept (but everything beyond is popped). If any of the popped items is protected (weakly or strongly) with a `CallId` of an active call, we have UB.
    - For read accesses, disable all `Unique` items above the granting one: change their permission to `Disabled`.  This means they cannot be used any more.  We do not remove them from the stack to avoid merging two blocks of `SharedReadWrite`.  If any disabled item is protected (weakly or strongly) with a `CallId` of an active call, we have UB.

### Reborrowing

Adding new permissions to the stack happens by reborrowing pointers.

**Granting a pointer permission to a location.**
To grant new permissions to a location, we need a parent tag (the tag of the pointer from which the new pointer is derived), and an `Item` for the newly created pointer that should be added to the stack (this indicates both which pointer is granted access and what the permission is).
As a Rust signature, this would be:
```rust
fn grant(
    self: &mut Stacks,
    derived_from: Tag,
    new: Item,
)
```
We proceed as follows:

1. We consider this operation as corresponding to a write access if `new.perm.grants(AccessKind::Write)`, and to a read access otherwise.
2. Find the granting item for this access and the parent tag. If there is none, this is UB.
3. Check if we are adding a `SharedReadWrite`.
    - If yes, add the new item on top of the current block.
    - If no, perform the actions of the corresponding access.
      Then push the new item to the top of the stack.

**Reborrowing a pointer.**
To reborrow a pointer, we are given:
- a (typed) place, i.e., a location in memory, a tag and the type of the data we expect there (from which we can compute the size);
- which kind of reference/pointer this is (`Unique`, `Shared` or a raw pointer which might be mutable or not);
- a `new_tag: Tag` for the reborrowed pointer;
- whether this reborrow needs to be protected, and if yes, how (weak or strong protection).

The type of the place and the kind of reference/pointer together give the full type of the reference/pointer (or as much of it was we need).
As a Rust signature, this would be:
```rust
pub enum RefKind {
    /// `&mut` and `Box`.
    Unique { two_phase: bool },
    /// `&` with or without interior mutability.
    Shared,
    /// `*mut`/`*const` (raw pointers).
    Raw { mutable: bool },
}

fn reborrow(
    self: &mut MiriInterpContext,
    place: MPlaceTy<Tag>,
    kind: RefKind,
    new_tag: Tag,
    protect: Option<ProtectorKind>,
)
```

We will grant `new_tag` permission for all the locations covered by this place, by calling `grant` for each location.
The parent tag (`derived_from`) is given by the place.
The interesting question is which permission to use for the new item:
- For non-two-phase `Unique`, the permission is `Unique`.
- For mutable raw pointers and two-phase `Unique`, the permission is `SharedReadWrite`.
- For `Shared` and immutable raw pointers, the permission is different for locations inside of and outside of `UnsafeCell`.
  Inside `UnsafeCell`, it is `SharedReadWrite`; outside it is `SharedReadOnly`.
  - The `UnsafeCell` detection is entirely static: it recurses through structs,
    tuples and the like, but when hitting an `enum` or `union` or so, it treats
    the entire field as an `UnsafeCell` unless its type is frozen. This avoids
    hard-to-analyze recursive behavior caused by Stacked Borrows itself doing
    memory accesses that are subject to Stacked Borrows rules.
- For immutable raw pointers, the rules are the same as for `Shared`.

If the reborrow is protected and we are not inside an `UnsafeCell` behind a `Shared` or an immutable raw pointer,
the new item will have its protector set to the `CallId` of the current function call (i.e., of the topmost frame in the call stack).
Otherwise the new item will not have a protector.

So, basically, for every location, we call `grant` like this:
```rust
let (perm, protect) = match ref_kind {
    RefKind::Unique { two_phase: false } =>
        (Permission::Unique, protect),
    RefKind::Raw { mutable: true } |
    RefKind::Unique { two_phase: true } =>
        (Permission::SharedReadWrite, protect),
    RefKind::Raw { mutable: false } |
    RefKind::Shared =>
        if inside_unsafe_cell { (Permission::SharedReadWrite, /* do not protect */ None) }
        else { (Permission::SharedReadOnly, protect) }
};
let protector = protect.map(|kind| (kind, current_call_id()));

location.stack.grant(
  place.tag,
  Item { tag: new_tag, perm, protector }
);
```

### Retagging

When executing `Retag(kind, place)`, we check if `place` holds a reference (`&[mut] _`) or box (`Box<_>`), and if `kind == Raw` we also check each raw pointer (`*[const,mut] _`).
For those we perform the following steps:

1. We compute a fresh tag: `Tracking::new_ptr_id()`.
2. We determine if and how will want to protect the items we are going to generate:
   If `kind == FnEntry`, then a protector will be added; for references, we use a `StrongProtector`, for box a `WeakProtector`.
   (This means for both of them there is UB if the pointer gets invalidated while the call is active; and for references, additionally there is UB if the memory the pointer points to gets deallocated in anyway -- even if the pointer itself is used for that deallocation.)
   For other `kind`, no protector is added.
3. We perform reborrowing of the memory this pointer points to with the new tag and indicating whether we want protection, treating boxes as `RefKind::Unique { two_phase: false }`.

We do not recurse into fields of structs or other compound types, only "bare" references/... get retagged.

**Note**: Miri offers a flag, `-Zmiri-retag-fields`, that changes this behavior to also recurse into compound types to search for references to retag.
We never recurse through a pointer indirection.

### Deallocating memory

Memory deallocation first acts like a write access through the pointer used for deallocation.
After that is done, we additionally check all *strong* protectors remaining in the stack: if any of them is still active, we have undefined behavior.
(Weak protectors do not matter here.)

## Adjustments to libstd

libstd needed/needs some patches to comply with this model. These provide a good opportunity to review if we are okay with the requirements that Stacked Borrows places onto unsafe code. For an up-to-date list of **"violations of Stacked Borrows"**, please refer to [this list maintained in the Miri repo](https://github.com/rust-lang/miri#bugs-found-by-miri).

* [`VecDeque` creating overlapping mutable references](https://github.com/rust-lang/rust/pull/56161)
* [Futures turning a shared reference into a mutable one](https://github.com/rust-lang/rust/pull/56319)
* [`str` turning a shared reference into a mutable one](https://github.com/rust-lang/rust/pull/58200)
* [`BTreeMap` creating mutable references that overlap with shared references](https://github.com/rust-lang/rust/pull/58431)
* [`LinkedList` creating overlapping mutable references](https://github.com/rust-lang/rust/pull/60072)
* [`VecDeque` invalidates a protected shared reference](https://github.com/rust-lang/rust/issues/60076)

## Biggest conceptual issues

The two biggest conceptual issues with this model are the following:

- Raw pointer casts generate fresh tags.
  This is a problem because it means we need to detect the *transitions* from references to raw pointers, which is not always easy.
  (In contrast, for all other retags we can just retag whenever we see a reference, no matter where it comes from.)
  It is also frequently surprising to programmers, e.g. when `addr_of_mut!(local)` is invalidated by direct writes to `local`.
  Finally it leads to the raw pointer type at the moment of transition being significant, which again defies the usual intuition and general goal of raw pointers that their type is not semantically relevant.
- On reads we do not follow a proper stack discipline.
  Instead, we just disable all `Unique` above the item that granted the read access.
  This is obviously ugly, but more importantly it means that the first issue cannot be easily fixed:
  if raw pointer casts just retained the original tag, then a raw pointer derived from `&mut` would become invalidated when the `&mut` becomes invalidated, and that just breaks way too much code.
  (Currently, the raw pointer instead becomes a `SharedReadWrite` on top of the `Unique`, so the `Unique` can be invalidated while the raw pointer remains usable.)

The best known alternative for the second point is to go the Tree Borrows route of *freezing* all `Unique` (and their children) above a read-granting item.
This basically means we would be popping such `Unique` (and everything above them) *only for writes* but not for reads---much nicer than the current situation.
However, this breaks *tons* of code that looks like this:
```rust
ptr::copy_nonoverlapping(src.as_ptr(), dest.as_mut_ptr(), dest.len());
```
Here `dest` is a slice.
The issue is that calling `dest.len()` *after* `dest.as_mut_ptr()` does a shared-read-only reborrow of `dest`, which freezes the raw pointer returned by `as_mut_ptr` and thus makes later writes to it UB.

It's not clear how this could be fixed without going all the way to [trees](https://perso.crans.org/vanille/treebor/).
