# BTF Extensions for Rust Support

## LPC 2026 - BTF for Rust: Bringing Type Information to Rust Kernel Modules

**Authors**: Arnaldo Carvalho de Melo, Alan Maguire

**Date**: 2026-09-16

---

## Executive Summary

This document proposes extensions to the BPF Type Format (BTF) specification to natively support Rust constructs that cannot be adequately represented using current BTF kinds. The current workaround uses struct/union representations that are lossy and require special handling by consumers.

---

## 1. Current State and Limitations

### 1.1 Current Workaround

Currently, pahole represents Rust enums using a struct with a union:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

**Current BTF encoding** (workaround):
```
[1] STRUCT 'Option<T>' size=8 vlen=2
    'discriminant' type_id=2 bits_offset=0   (u32)
    '(anon)' type_id=3 bits_offset=32        (union)

[3] UNION '(anon)' size=4 vlen=2
    '(anon)' type_id=4 bits_offset=0    (Some payload)
    '(anon)' type_id=5 bits_offset=0    (None payload)
```

### 1.2 Limitations of Current Approach

1. **Lossy representation**: The discriminant's semantic meaning (which variant is active) is not explicit
2. **Consumer burden**: BTF consumers must know to interpret unions as variants
3. **No variant names**: Anonymous unions lose variant names like "Some", "None"
4. **No discriminant type**: The discriminant's actual enum type is lost
5. **Limited to C-like unions**: Cannot represent variant-specific type constraints

### 1.3 Tag-Based Representation Alternative

The following sections also describe a no-new-kind alternative based on the
existing `BTF_KIND_DECL_TAG` and `BTF_KIND_TYPE_TAG` kinds.  Tags in this
proposal use the `rust:type:<annotation>` namespace.  A type tag describes the
Rust meaning of the type to which it points; a declaration tag describes a
particular STRUCT or UNION member through its `component_idx`.  Consumers that
do not understand the namespace retain the ordinary, layout-correct BTF view.

For tags that carry a variant discriminant, this document uses
`rust:type:variant:<name>:discriminant:<value>`.  `<value>` is the Rust
discriminant written as a signed decimal integer or `0x`-prefixed unsigned
hexadecimal integer.  The spelling is illustrative, but a stable grammar is
needed before producers rely on it.

---

## 2. Proposed BTF Extensions

### 2.1 New BTF Kind: BTF_KIND_VARIANT_PART (Kind #20)

Represents Rust's discriminated union (enum with data), DW_TAG_variant_part.

**Type Structure** (`struct btf_type`):
```c
struct btf_type {
    __u32 name_off;       // For named enums (e.g., "Option")
    __u32 info;           // kind = 20 (BTF_KIND_VARIANT_PART)
                          // kind_flag: 0
                          // vlen: number of variants
    __u32 size;           // Total size of the discriminated union
    union {
        __u32 type;      // Points to discriminant type (BTF_KIND_ENUM or BTF_KIND_ENUM64)
        __u32 unused;
    };
};
```

**Variant Structure** (`struct btf_variant`, follows `struct btf_type`):
```c
struct btf_variant {
    __u32 name_off;       // Variant name (e.g., "Some", "None")
    __u32 type;           // Points to variant's payload type (struct/union)
    __u32 offset;        // Discriminant value that activates this variant
};
```

**Example**:
```
[1] VARIANT_PART 'Option' size=8 vlen=2
    type=2    (discriminant type: u32)
    [0] 'Some' type_id=4 offset=1    (Some(T))
    [1] 'None' type_id=5 offset=0    (None)
```

**Alternative using existing tags**: Retain the current STRUCT+UNION layout,
but mark its Rust roles and name each variant.  In particular, tag the actual
discriminant declaration; this makes it unambiguous which integer selects the
active union member.
```
[1] STRUCT 'Option<T>' size=8 vlen=2
    'discriminant' type_id=2 bits_offset=0
    '(anon)'       type_id=3 bits_offset=32
[2] DECL_TAG 'rust:type:discriminant' type_id=1 component_idx=0
[3] DECL_TAG 'rust:type:enum-payload' type_id=1 component_idx=1
[4] UNION '(anon)' size=4 vlen=2
    '(anon)' type_id=5 bits_offset=0       (Some payload)
    '(anon)' type_id=6 bits_offset=0       (None payload)
[5] TYPE_TAG 'rust:type:enum' type_id=1
[6] TYPE_TAG 'rust:type:enum-variants' type_id=4
[7] DECL_TAG 'rust:type:variant:Some:discriminant:1' type_id=4 component_idx=0
[8] DECL_TAG 'rust:type:variant:None:discriminant:0' type_id=4 component_idx=1
```

This form retains the discriminant's storage type and offset in the STRUCT,
and supplies the missing mapping from discriminant value to union member.
It is still a convention: unlike `VARIANT_PART`, BTF validation cannot require
that every enum has one discriminant and a complete, non-overlapping variant
mapping.

### 2.2 New BTF Kind: BTF_KIND_SLICE (Kind #21)

Represents Rust's dynamically-sized slice type `&[T]`, DW_TAG_pointer_type pointing to array.

**Type Structure** (`struct btf_type`):
```c
struct btf_type {
    __u32 name_off;       // 0 (anonymous) or element type name
    __u32 info;           // kind = 21 (BTF_KIND_SLICE)
                          // kind_flag: 0
                          // vlen: 0
    __u32 size;           // Total size = sizeof(ptr) + sizeof(len) = 16 on 64-bit
    union {
        __u32 type;       // Points to element type (T)
        __u32 unused;
    };
};
```

**Layout**: Two implicit fields at fixed offsets:
- Offset 0: `data_ptr` - Pointer to element 0 (type: `const T *`)
- Offset 8: `length` - Number of elements (type: `__u64`)

**Example**:
```
[1] SLICE 'u32' size=16 vlen=0
    type=2    (element type: u32)
    // implicit: data_ptr at offset 0, length at offset 8
```

**Note**: The implicit field approach is chosen to match how LLVM emits DWARF for slices. An alternative is to make fields explicit with vlen=2, which some reviewers may prefer for consistency with STRUCT/UNION.

**Alternative using existing tags**: Preserve the explicit, ABI-specific
STRUCT representation and identify the two fields semantically.
```
[1] STRUCT '&[T]' size=16 vlen=2
    'data_ptr' type_id=2 bits_offset=0
    'length'   type_id=3 bits_offset=64
[2] TYPE_TAG 'rust:type:slice' type_id=1
[3] DECL_TAG 'rust:type:slice-data' type_id=1 component_idx=0
[4] DECL_TAG 'rust:type:slice-length' type_id=1 component_idx=1
```

This avoids assuming pointer and `usize` widths or fixed offsets in a new BTF
kind, while allowing consumers to distinguish a slice from any coincidentally
similar two-field C struct.

### 2.3 New BTF Kind: BTF_KIND_VTABLE (Kind #22)

Represents Rust's vtable for trait objects (`&dyn Trait`).

**Type Structure** (`struct btf_type`):
```c
struct btf_type {
    __u32 name_off;       // Trait name (e.g., "Draw")
    __u32 info;           // kind = 22 (BTF_KIND_VTABLE)
                          // kind_flag: 0
                          // vlen: number of methods
    __u32 size;           // Total vtable size
    union {
        __u32 type;       // Points to trait definition (empty BTF_KIND_STRUCT)
        __u32 unused;
    };
};
```

**Vtable Method Structure** (`struct btf_vtable_method`, follows `struct btf_type`):
```c
struct btf_vtable_method {
    __u32 name_off;        // Method name
    __u32 type;            // Function pointer type
    __u32 vtable_offset;   // Offset in vtable (for ordering verification)
};
```

**Example**:
```
[1] VTABLE 'Draw' size=32 vlen=3
    type=2    (trait definition: empty struct)
    [0] 'draw' type_id=func_proto offset=0
    [1] 'clone' type_id=func_proto offset=8
    [2] 'drop_in_place' type_id=func_proto offset=16
```

**ABI Considerations**: Method ordering must match the compiler's vtable layout. The `vtable_offset` field provides verification but does not affect CO-RE relocations.

**Alternative using existing tags**: A compiler-emitted STRUCT for the vtable
can be marked as a Rust vtable, with declaration tags identifying method slots.
The trait name remains the BTF type name (or another compiler-provided name).
```
[1] STRUCT 'Draw::vtable' size=32 vlen=3
    'draw'          type_id=fn_ptr bits_offset=0
    'clone'         type_id=fn_ptr bits_offset=64
    'drop_in_place' type_id=fn_ptr bits_offset=128
[2] TYPE_TAG 'rust:type:vtable' type_id=1
[3] DECL_TAG 'rust:type:vtable-method' type_id=1 component_idx=0
[4] DECL_TAG 'rust:type:vtable-method' type_id=1 component_idx=1
[5] DECL_TAG 'rust:type:vtable-method' type_id=1 component_idx=2
```

Tags make the layout recognizable but do not themselves standardize Rust's
vtable ABI or express a method's trait definition beyond the underlying
function prototype and type name.

### 2.4 New BTF Kind: BTF_KIND_FATPTR (Kind #23)

Represents fat pointers (trait objects) as distinct from regular pointers.

**Type Structure** (`struct btf_type`):
```c
struct btf_type {
    __u32 name_off;       // 0 (anonymous)
    __u32 info;           // kind = 23 (BTF_KIND_FATPTR)
                          // kind_flag: 0
                          // vlen: 0
    __u32 size;           // Total size = 16 on 64-bit
    union {
        __u32 type;       // Points to vtable type (BTF_KIND_VTABLE)
        __u32 unused;
    };
};
```

**Layout**: Two implicit fields at fixed offsets:
- Offset 0: `data_ptr` - Pointer to data (type: `const void *`)
- Offset 8: `vtable_ptr` - Pointer to vtable (type: `const BTF_KIND_VTABLE *`)

**Example**:
```
[1] FATPTR '' size=16 vlen=0
    type=2    (vtable type: VTABLE 'Draw')
    // implicit: data_ptr at offset 0, vtable_ptr at offset 8
```

**Alternative using existing tags**: Retain the two-field STRUCT and label its
data and metadata pointers.  For a trait object, the metadata field points at
the separately tagged vtable above.
```
[1] STRUCT '&dyn Draw' size=16 vlen=2
    'pointer' type_id=2 bits_offset=0
    'vtable'  type_id=3 bits_offset=64
[2] TYPE_TAG 'rust:type:trait-object' type_id=1
[3] DECL_TAG 'rust:type:trait-object-data' type_id=1 component_idx=0
[4] DECL_TAG 'rust:type:trait-object-vtable' type_id=1 component_idx=1
```

This distinguishes a trait-object fat pointer from an unrelated pointer pair,
but a type tag alone cannot require that the metadata pointer targets a vtable.

### 2.5 Extended BTF_KIND_ENUM64 for 128-bit Discriminants

BTF_KIND_ENUM64 (Kind #19) currently supports 64-bit values via `struct btf_enum64`:
```c
struct btf_enum64 {
    __u32   name_off;
    __u32   val_lo32;      // Lower 32 bits
    __u32   val_hi32;      // Higher 32 bits
};
```

**Extension for 128-bit support**:
```c
struct btf_enum128 {
    __u32   name_off;
    __u32   val_lo32;      // Lower 32 bits
    __u32   val_hi32;      // Next 32 bits
    __u32   val_hi32_2;    // Next 32 bits
    __u32   val_hi32_3;    // Upper 32 bits (for 128-bit values)
};
```

**Status**: Currently pahole tracks discriminant length via `discr_value_len` field in `struct variant`. Full 128-bit storage requires extending `btf_enum64` to 4 x 32-bit words.

**Alternative using existing tags**: The current integer or surrogate storage
can be tagged `rust:type:discriminant` and, where useful,
`rust:type:discriminant-width:128`.  This tells consumers that the value is a
Rust enum discriminant and records its intended width, but it cannot recover
the omitted high bits of a value.  Tags therefore complement, rather than
replace, a 128-bit value representation.

**Note** Perhaps separately we should explore a size-agnostic enum since
we already have 32, 64 variants.

### 2.6 Tag-Only Deployment Option

The alternatives above can be deployed independently of new BTF kinds.  A
producer may emit conventional STRUCT/UNION/INT/PTR BTF plus Rust tags, while
a consumer uses the tags when present and falls back to the normal layout when
they are absent.  This is particularly useful while BTF kind allocation and
kernel/libbpf support are under discussion.  The new kinds remain preferable
where the relationship must be machine-validated rather than merely conveyed.

---

## 3. Comparison: Current vs Tag-Based vs New-Kind Proposals

### 3.1 Option<T> Comparison

**Current (workaround)**:
```
[1] STRUCT 'Option<T>' size=8
    'discriminant' type_id=u32
    '(anon)' type_id=union

[2] UNION '(anon)' size=4
    [no variant names]
```

**Proposed**:
```
[1] VARIANT_PART 'Option' size=8
    discriminant_type=u32
    [0] 'Some' type_id=struct(T) offset=1
    [1] 'None' type_id=struct {} offset=0
```

**Tag-based alternative**:
```
STRUCT 'Option<T>' { discriminant: u32, payload: UNION }
TYPE_TAG 'rust:type:enum' -> Option<T>
DECL_TAG 'rust:type:discriminant' -> Option<T>.discriminant
DECL_TAG 'rust:type:enum-payload' -> Option<T>.payload
TYPE_TAG 'rust:type:enum-variants' -> UNION
DECL_TAG 'rust:type:variant:Some:discriminant:1' -> UNION[0]
DECL_TAG 'rust:type:variant:None:discriminant:0' -> UNION[1]
```

The tag-based form resolves the principal ambiguity of the workaround: it says
both which field is the discriminant and which union member each value selects.

### 3.2 &[T] Comparison

**Current (workaround)**:
```
[1] STRUCT '&[T]' size=16
    'data_ptr' type_id=ptr(T)
    'length' type_id=usize
    [no semantic marker]
```

**Proposed**:
```
[1] SLICE 'T' size=16
    element_type=ptr(T)
    [implicit] data_ptr, length fields
```

**Tag-based alternative**:
```
STRUCT '&[T]' { data_ptr: *T, length: usize }
TYPE_TAG 'rust:type:slice' -> &[T]
DECL_TAG 'rust:type:slice-data' -> data_ptr
DECL_TAG 'rust:type:slice-length' -> length
```

This retains an inspectable physical layout and makes its slice semantics
explicit without adding implicit BTF fields.

### 3.3 &dyn Trait Comparison

**Current (workaround)**:
```
[1] STRUCT '&dyn Draw' size=16
    'pointer' type_id=ptr
    'vtable' type_id=ptr
    [no semantic marker]
```

**Proposed**:
```
[1] FATPTR 'Draw' size=16
    type=2    (vtable)
    // implicit: data_ptr, vtable_ptr

[2] VTABLE 'Draw' size=32
    [0] 'draw' type_id=fn
    [1] 'clone' type_id=fn
```

**Tag-based alternative**:
```
STRUCT '&dyn Draw' { pointer: *void, vtable: *Draw::vtable }
TYPE_TAG 'rust:type:trait-object' -> &dyn Draw
DECL_TAG 'rust:type:trait-object-data' -> pointer
DECL_TAG 'rust:type:trait-object-vtable' -> vtable
TYPE_TAG 'rust:type:vtable' -> Draw::vtable
DECL_TAG 'rust:type:vtable-method' -> each method member
```

The tags identify the data/metadata relationship and the vtable's role, while
the dedicated kinds additionally make those relationships structural.

### 3.4 Summary of Trade-offs

| Construct | Current STRUCT/UNION view | Tag-based alternative | Dedicated kind |
|-----------|---------------------------|-----------------------|----------------|
| Data-carrying enum | Layout only; active variant and names are ambiguous | Tags identify enum, discriminant, payload, variant names, and values | `VARIANT_PART` makes variants and values first-class and validatable |
| Slice | Indistinguishable from a pointer-plus-length struct | Tags identify the slice and its data/length members | `SLICE` gives a compact canonical representation, but has implicit-layout questions |
| Trait object | Indistinguishable from an arbitrary pointer pair | Tags identify data, vtable metadata, and tagged vtable members | `FATPTR`/`VTABLE` make the relationship explicit in BTF's type graph |
| 128-bit discriminant | Values wider than 64 bits cannot be represented | Tags can state Rust role and intended width only | Extended value encoding is still required for lossless values |

Tags require no new kind numbers and can be ignored safely by existing
consumers. Their trade-off is that tag spelling, completeness, and cross-tag
relationships are conventions. Dedicated kinds cost ecosystem changes but can
be specified and validated as part of BTF itself.

---

## 4. Implementation Plan

### 4.0 Phase 0: Rust Type Tags (Independent, Early Deployment)

**Rationale**: Uses BTF kinds already understood by the kernel and libbpf,
and improves interpretation of BTF emitted today.

**Steps**:
1. Specify the `rust:type:<annotation>` grammar and the tag-to-`component_idx`
   attachment rules.
2. Add pahole emission for enum, slice, trait-object, and vtable tags.
3. Add btf_dump output and consumer helpers that recognize the namespace.
4. Add tests for tag presence, discriminant-to-variant mappings, and graceful
   fallback when a tag is absent or unrecognized.

**Kernel impact**: None. `BTF_KIND_DECL_TAG` and `BTF_KIND_TYPE_TAG` already
carry the metadata; this phase standardizes their Rust-specific interpretation.

### 4.1 Phase 1: BTF_KIND_VARIANT_PART (Kind #20) - High Priority

**Rationale**: Most impactful for Rust kernel modules, enables proper enum support.

**Steps**:
1. Propose kind #20 in linux/btf.h
2. Add `struct btf_variant` definition
3. Add BTF_KIND_VARIANT_PART to pahole's btf_encoder.c
4. Add to libbpf's btf_dump.c for pretty-printing
5. Add kernel verifier support (if needed for CO-RE)
6. Add tests with various enum forms

**Kernel impact**: Low - primarily affects type representation, verifier changes minimal.

### 4.2 Phase 2: BTF_KIND_SLICE (Kind #21)

**Rationale**: Essential for proper slice representation.

**Steps**:
1. Propose kind #21 in linux/btf.h
2. Update pahole to emit BTF_KIND_SLICE
3. Update consumers (bpftrace, libbpf)

**Kernel impact**: None - purely type information.

### 4.3 Phase 3: BTF_KIND_VTABLE (Kind #22)

**Rationale**: Enables proper trait object representation.

**Steps**:
1. Propose kind #22 in linux/btf.h
2. Add `struct btf_vtable_method` definition
3. Handle method ordering for ABI compatibility
4. Update CO-RE relocations if needed

**Kernel impact**: May affect CO-RE relocation handling.

### 4.4 Phase 4: BTF_KIND_FATPTR (Kind #23)

**Rationale**: Distinguishes trait objects from regular pointers.

**Steps**:
1. Propose kind #23 in linux/btf.h
2. Update pahole to emit BTF_KIND_FATPTR for `&dyn Trait`
3. Update consumers

**Kernel impact**: None - purely type information.

### 4.5 Phase 5: 128-bit ENUM64 (Incremental)

**Status**: BTF_KIND_ENUM64 already in kernel 6.x, pahole supports.

**Remaining work**:
1. Extend `struct btf_enum64` to 4 x 32-bit words for true 128-bit support
2. Complete 128-bit value storage (currently tracking length only)
3. Test with large Rust enums

---

## 5. Backward Compatibility

All proposed changes are purely additive:

1. **New kind numbers** only - no changes to existing kinds
2. **Consumers ignore unknown kinds** - current behavior
3. **pahole can emit both old and new** - gradual rollout
4. **No kernel verifier changes** required for type info (only for CO-RE)
5. **Tag-only form uses existing kinds** - old consumers retain the underlying
   STRUCT/UNION layout and ignore the `rust:type:*` annotations

---

## 6. Open Questions for Discussion

1. **Implicit vs Explicit fields for SLICE/FATPTR**: Current proposal uses implicit fields (data_ptr, length at fixed offsets). Alternative: make fields explicit with vlen=N. Implicit is simpler but less consistent with STRUCT/UNION.

2. **Vtable method ordering**: Who defines the canonical ordering? Currently relying on compiler (LLVM) consistency. Should we add a method_order attribute?

3. **BTF_KIND_VARIANT_PART size field**: Should the size include the discriminant, or just the payload? Current proposal includes discriminant in total size.

4. **Discriminant type reference**: For BTF_KIND_VARIANT_PART, should discriminant type be mandatory or optional (for C-style enums with no payload variants)?

5. **Backward compatible transition**: Should pahole emit both old (STRUCT+UNION) and new (VARIANT_PART) simultaneously during transition period?

6. **Tag grammar and completeness**: Which `rust:type:<annotation>` spellings
   are standardized, how are non-integer or 128-bit discriminant values
   encoded, and must every payload member have exactly one variant tag?

7. **Attachment point**: Should an enum's semantic tag apply to its enclosing
   STRUCT, its payload UNION, or both? This proposal uses both where each has
   a distinct role.

---

## 7. References

- [Linux BTF documentation](Documentation/bpf/btf.rst)
- [pahole GitHub](https://github.com/acmel/dwarves)
- [Rust DWARF extensions](https://github.com/rust-lang/rust/blob/master/compiler/rustc_dwarf/src/dwarf.rs)
- [LLVM DWARF to BTF](https://github.com/llvm/llvm-project)
- [bpftool source](https://github.com/libbpf/bpftool)
- [libbpf btf_dump](https://github.com/libbpf/libbpf)

## 8. Patch Submission Plan

### 8.1 Upstream Targets

1. **Linux kernel** (include/uapi/linux/btf.h):
   - Add BTF_KIND_VARIANT_PART (20)
   - Add BTF_KIND_SLICE (21)
   - Add BTF_KIND_VTABLE (22)
   - Add BTF_KIND_FATPTR (23)
   - Add struct btf_variant
   - Add struct btf_vtable_method

2. **pahole/dwarves** (btf_encoder.c):
   - Emit `rust:type:*` DECL_TAG/TYPE_TAG annotations for conventional Rust
     layouts, including discriminant-to-variant mappings
   - Emit new BTF kinds from DWARF variant_part
   - Emit BTF_KIND_SLICE for slices
   - Emit BTF_KIND_VTABLE for vtables
   - Emit BTF_KIND_FATPTR for trait objects

3. **libbpf/bpftool**:
   - Display and optionally interpret the `rust:type:*` tag namespace
   - btf_dump.c: Pretty-print new kinds
   - bpftool: Support for new kinds

### 8.2 Mailing List

Send RFC patches to:
- bpf@vger.kernel.org
- linux-kernel@vger.kernel.org
- rust-for-linux@vger.kernel.org (for Rust-specific feedback)

### 8.3 LPC 2026 BoF

Propose BoF session: "Extending BTF for Rust Support"

Target outcomes:
1. Get buy-in from BTF maintainers
2. Identify conflicts with other proposals
3. Coordinate with Rust toolchain team
4. Establish timeline for upstreaming

---

## 9. Appendix: DWARF to BTF Mapping

| DWARF Construct | Current BTF | Proposed BTF | Kind # |
|-----------------|-------------|-------------|--------|
| DW_TAG_variant_part | STRUCT + UNION | `rust:type:enum`, `rust:type:discriminant`, and `rust:type:enum-variants` tags; or BTF_KIND_VARIANT_PART | 20 |
| DW_TAG_variant | UNION member | `rust:type:variant:<name>:discriminant:<value>` DECL_TAG; or BTF_VARIANT struct | - |
| &[T] slice | STRUCT(ptr, len) | `rust:type:slice`, `rust:type:slice-data`, `rust:type:slice-length`; or BTF_KIND_SLICE | 21 |
| &dyn Trait (data) | STRUCT(ptr, vtable) | `rust:type:trait-object` and field tags; or BTF_KIND_FATPTR | 23 |
| vtable | STRUCT(fn*) | `rust:type:vtable` and `rust:type:vtable-method`; or BTF_KIND_VTABLE | 22 |
| DW_AT_discr_value | Implicit in union | Variant DECL_TAG carries name/value; dedicated variant carries value structurally | - |
| 128-bit discriminant | N/A | Tag role/width, plus BTF_KIND_ENUM64 (extended) for the value | 19 |
