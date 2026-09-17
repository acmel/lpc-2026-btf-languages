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
    __u32   val_hi32_2;    // Upper 32 bits (for 128-bit values)
};
```

**Status**: Currently pahole tracks discriminant length via `discr_value_len` field in `struct variant`. Full 128-bit storage requires extending `btf_enum64` to 4 x 32-bit words.

---

## 3. Comparison: Current vs Proposed

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

---

## 4. Implementation Plan

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

---

## 6. Open Questions for Discussion

1. **Implicit vs Explicit fields for SLICE/FATPTR**: Current proposal uses implicit fields (data_ptr, length at fixed offsets). Alternative: make fields explicit with vlen=N. Implicit is simpler but less consistent with STRUCT/UNION.

2. **Vtable method ordering**: Who defines the canonical ordering? Currently relying on compiler (LLVM) consistency. Should we add a method_order attribute?

3. **BTF_KIND_VARIANT_PART size field**: Should the size include the discriminant, or just the payload? Current proposal includes discriminant in total size.

4. **Discriminant type reference**: For BTF_KIND_VARIANT_PART, should discriminant type be mandatory or optional (for C-style enums with no payload variants)?

5. **Backward compatible transition**: Should pahole emit both old (STRUCT+UNION) and new (VARIANT_PART) simultaneously during transition period?

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
   - Emit new BTF kinds from DWARF variant_part
   - Emit BTF_KIND_SLICE for slices
   - Emit BTF_KIND_VTABLE for vtables
   - Emit BTF_KIND_FATPTR for trait objects

3. **libbpf/bpftool**:
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
| DW_TAG_variant_part | STRUCT + UNION | BTF_KIND_VARIANT_PART | 20 |
| DW_TAG_variant | UNION member | BTF_VARIANT struct | - |
| &[T] slice | STRUCT(ptr, len) | BTF_KIND_SLICE | 21 |
| &dyn Trait (data) | STRUCT(ptr, vtable) | BTF_KIND_FATPTR | 23 |
| vtable | STRUCT(fn*) | BTF_KIND_VTABLE | 22 |
| DW_AT_discr_value | Implicit in union | Explicit in variant | - |
| 128-bit discriminant | N/A | BTF_KIND_ENUM64 (extended) | 19 |
