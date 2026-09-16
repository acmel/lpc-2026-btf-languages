<!-- SPDX-License-Identifier: GPL-2.0-only -->
# Research notes: BTF for more languages (LPC 2026 eBPF track)

Companion to `index.html`. Facts checked Sep 2026; Alan speaks, so
flag anything uncertain to him before the talk.

## LLVM encoding (talk: "LLVM already encodes it" slide)

- llvm-project PR 155783: DW_TAG_variant_part → STRUCT wrapper +
  anonymous UNION holding discriminant + variants. Merged Oct 2025.
  https://github.com/llvm/llvm-project/pull/155783
- Deliberate deviation (talk point): LLVM emits non-zero-offset
  discriminants inside the UNION; the kernel verifier rejects those
  (`is_union && offset`) and libbpf won't create them. pahole puts
  real storage in the outer STRUCT, drops payload-overlaps.

## Compiler BTF (talk: "Compilers emit BTF too" slide)

- GCC `-gbtf` since GCC 12 (any target, default on BPF), `-gctf`
  sibling, `-mco-re` for CO-RE.
- LLVM `-gbtf` (target-independent, PR #183929, 2025) + lld
  `--btf-merge`: pahole step optional someday, not today.
  https://github.com/llvm/llvm-project/pull/183929
- gccrs: NOT merged (mid-2026: out-of-tree, ~Rust 1.49 target).
  https://rust-gcc.github.io/
- RANDSTRUCT (kernel) / Clang `-frandomize-layout-seed`: without the
  seed, on-disk layouts lie to every consumer, pahole included. Say
  it plainly; someone in the room keeps such a kernel.

## Rust specifics

- rustc `-Z randomize-layout` (+ `-Z layout-seed`, nightly): shuffles
  `repr(Rust)` fields to catch layout assumptions.
- Layout guarantees: https://doc.rust-lang.org/reference/type-layout.html
  (default repr explicitly unstable — grounds the "don't infer"
  slide).
- rustc debuginfo: MIR → LLVM DIBuilder; enums as structure_type +
  variant_part (+discriminator).
  https://rustc-dev-guide.rust-lang.org/debuginfo/rust-codegen.html

## Traps: do NOT say on stage

- BTF carries layout, not semantics: DW_AT_discr_value per variant is
  NOT representable today — verifier cannot check matches. The
  "ask to the room" is a proposal, label it so.
- Niche rule: emit `__discriminant` only when it doesn't overlap the
  payload; overlapping members are legal in STRUCT, illegal in UNION.
