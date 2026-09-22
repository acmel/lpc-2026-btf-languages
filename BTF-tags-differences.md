# BTF tag representations; gcc and clang

BTF tags - once in BTF - are identical.  However how they are
handled in DWARF differs between clang and gcc.

## DWARF representation

Clang emits:

```
  DW_TAG_formal_parameter "arg"
    DW_TAG_LLVM_annotation
      DW_AT_name        "btf_decl_tag"
      DW_AT_const_value "parameter"

  DW_TAG_pointer_type
    DW_TAG_LLVM_annotation
      DW_AT_name        "btf_type_tag"
      DW_AT_const_value "one"

```


GCC emits the annotation DIE elsewhere, then links it from the annotated DIE:

```
  DW_TAG_formal_parameter "arg"
    DW_AT_GNU_annotation -> DW_TAG_GNU_annotation
                                    name = "btf_decl_tag"
                                    value = "parameter"

  DW_TAG_pointer_type
    DW_AT_GNU_annotation -> DW_TAG_GNU_annotation
                                    name = "btf_type_tag"
                                    value = "one"
                                    DW_AT_GNU_annotation -> next tag
```

In pahole:

  - For decl tags, Clang annotations are collected by walking child DIEs; GCC
    annotations by following the GNU attribute chain.

  - The accepted annotation name is explicitly limited to btf_decl_tag.
  - Both become BTF_KIND_DECL_TAG(value, target_btf_id, component_idx).
      - component_idx = -1: function, variable, type, or whole declaration.
      - component_idx >= 0: parameter or struct/union member index.

  - For type tags, pahole makes a chain of BTF_KIND_TYPE_TAGs between the
    pointer and pointee.

Ordering needs compiler-specific handling. For:

```
  int __attribute__((btf_type_tag("outer")))
      __attribute__((btf_type_tag("inner"))) *p;
```

the desired BTF is:

```
  PTR -> TYPE_TAG "inner" -> TYPE_TAG "outer" -> int
```

Clang emits child annotations in source order (outer, inner), so pahole
prepends them while loading. GCC’s GNU reference chain is already in BTF
wrapper order, so pahole appends them.

## Support differences

  - Clang has supported btf_decl_tag since clang 14 and emits LLVM child
    annotations.

  - GCC’s decl-tag support is newer (the dwarves test gates it at GCC 16+) and
    uses GNU annotation chains.

  - GCC currently does not emit decl tags for an entire struct/union or typedef;
    pahole tolerates that absence. Clang can represent those through child
    annotations.

  - GCC may place annotation DIEs in a DWZ alternate file (DW_FORM_GNU_ref_alt);
    pahole has special handling to retain those referenced DIEs during
    alternate-CU pruning.

# Summary

BTF support is identical, but DWARF representations differ.  In particular,
even a tag-supporting gcc (16+) does now allow a decl tag to target a type
(a struct or a typedef) as a whole (component_idx=-1).  gcc views decl tags
as declaration attributes _not_ type attributes.

However again that does not restrict BTF representation for languages
like Rust since we are not deriving BTF declaration tags from DWARF.

