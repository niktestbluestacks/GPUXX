# Project Goals

## Working name: **GPUXX** (placeholder — rename freely)

*A source-compatible C++ extension for GPU compute, implemented as a targeted
patch to Clang that lowers annotated functions to SPIR‑V and runs them through
Vulkan Compute.*

---

## 1. Problem statement

Existing GPU compute solutions force users into a different language or toolchain:

| Approach | Cost to the user |
|---|---|
| CUDA / HIP | Vendor lock-in, proprietary dialect of C++, separate compiler (`nvcc`), separate build system |
| SYCL / DPC++ | Whole-language rewrite (kernel lambdas, accessors, buffers), heavyweight runtime, large Clang fork to maintain |
| OpenCL C | A restricted C dialect, not C++; no templates, no RAII, no exceptions |
| `[[clang::gpu_kernel]]` + `--offload-arch=spirv64` (upstream today) | Requires LLVM's NVPTX/AMDGPU-based pipeline; SPIR-V backend is incomplete; no Vulkan runtime; not usable on shipping drivers |

**Our thesis:** none of this requires a new language. Everything we need can be
expressed with **standard C++ attributes plus one header**. The only thing missing
is a compiler that, when it sees such an annotation, emits *compute-shader* code
(SPIR‑V) instead of host object code — and a small runtime that submits it to a
Vulkan device.

Therefore we do **not** create a superset of C++. We create a **conservative,
explicitly-annotated subset mode**: unannotated code compiles exactly as before;
annotated code is validated against a documented restriction set and lowered to
SPIR‑V.

---

## 2. Primary goal (the one sentence)

> Given a standard-conforming C++ translation unit compiled by our patched Clang,
> any function marked `[[gpu::execute(...)]]` is compiled to a Vulkan-compatible
> SPIR‑V compute shader and can be launched from host code with a one-line runtime
> call, with **zero** changes to the language grammar and **zero** changes to
> unannotated code paths.

### Concrete acceptance example (this must compile and run in Milestone 3)

```cpp
// gpu/gpu.hpp is the ONLY added header. No new keywords, no new syntax.
#include <gpu/gpu.hpp>
#include <vector>

// Plain C++ function, marked as GPU-executable with execution parameters.
[[gpu::execute(.grid = {1024}, .block = {32}, .shared = 4096)]]
void saxpy(float a, std::span<float> x, std::span<float> y) {
    auto i = gpu::thread_id();                 // builtin, not a keyword
    if (i < x.size()) y[i] = gpu::fma(a, x[i], y[i]);
}

int main() {
    std::vector<float> x(1 << 20), y(1 << 20, 0.f);
    gpu::launch(saxpy, 2.0f, gpu::in_out(x), gpu::in_out(y));   // host-side launch
    // ...or: gpu::launch_async(...), gpu::graph, etc.
}
```

Note the design constraints visible above, which define the whole project:

* `[[...]]` is a **standard attribute** — the TU remains valid C++20/23 and can be
  compiled by any conforming compiler in "no-GPU" fallback mode (see §7).
* Execution parameters are carried in the attribute argument list, not in a new
  DSL. (Design note: GNU/Clang *designated initializers* inside attribute arguments
  are non-standard; final spelling TBD — see §9 Open Questions.)
* Thread/block indexing is a **library function**, not a built-in keyword.
* There is no `__global__`, no `<<<grid, block>>>`, no `.cu` file extension.

---

## 3. Non-goals

Stating these explicitly is as important as the goals — they are what keeps the
Clang patch small enough to survive upstream rebase.

1. **Not a new language / no grammar changes.** No new keywords, no preprocessing
   extensions, no alternate file suffixes, no CUDA-style launch syntax `f<<<a,b>>>`.
2. **Not a full C++ GPU compiler.** No dynamic allocation, virtual dispatch across
   the module boundary, exceptions, `std::function`, RTTI, coroutines, or host
   pointers inside kernels. These are *rejected with diagnostics*, not silently
   miscompiled.
3. **Not graphics.** No vertex/fragment/tessellation/geometry/ray-tracing stages,
   no swapchain, no rendering. Compute queues only.
4. **Not a general Vulkan toolkit.** We do not wrap Vulkan; we use the minimum
   surface needed: instance, physical-device selection, one device, one compute
   queue, descriptor sets, buffer memory, pipeline cache, fences/events.
5. **Not CUDA source migration.** We will not parse `.cu`. A best-effort
   `cuda→gpu` mechanical translator may exist as a *separate* tool later, out of scope.
6. **Not multi-vendor IR.** No PTX, no DXBC, no MSIL. SPIR‑V is the single
   compilation target. (Vulkan-on-NVIDIA/AMD/Intel executes SPIR‑V natively or via
   the driver's own translator — that is the vendor's problem, not ours.)
7. **Not upstream-critical initially.** We maintain a rebasable patch series against
   a pinned Clang release. Upstreaming is a stretch goal (§8, M6+), never a blocker.

---

## 4. Architectural strategy

Three components, cleanly separated so each can be tested alone:

```
        ┌──────────────────────────────────────────────────────────┐
        │  (A) Patched Clang  —  "compiler side"                   │
        │                                                          │
        │  Sema      attribute parsing/validation, restriction     │
        │            checking (diagnostics engine)                 │
        │  AST       implicit gpu:: declarations, kernel metadata  │
        │  CodeGen   LLVM IR with gpu.* intrinsics + kernel ABI    │
        │  Backend   SPIR-V lowering pass → .spv embedded in the   │
        │            object file as a dedicated section            │
        │            (.gcc.gpu_kernels / .note.gpu.*)              │
        └───────────────┬──────────────────────────────────────────┘
                        │  .o / .a containing host code + SPIR-V blobs
        ┌───────────────▼──────────────────────────────────────────┐
        │  (B) libgpu  —  "runtime side", ordinary C++ library     │
        │                                                          │
        │  Loader    scan sections, dlopen-free, no JIT            │
        │  Device    Vulkan instance/device/queue mgmt, selection   │
        │            heuristics, discrete > integrated > llvmpipe   │
        │  Memory    host-visible/device-local pools, UBO/PushConst │
        │            classification, explicit H2D/D2H, unified opt. │
        │  Dispatch  pipeline cache, descriptor binding, launch,    │
        │            capture/replay (VkComputePipelineKHR-free)     │
        │  Diagnostics  SPIR-V → source line mapping, validation    │
        │               layer hookup                                │
        └───────────────┬──────────────────────────────────────────┘
                        │
        ┌───────────────▼──────────────────────────────────────────┐
        │  (C) gpu/gpu.hpp  —  the single public header (+ a few   │
        │  subheaders under include/gpu/)                           │
        │  attributes, index/builtin functions, span-like views,    │
        │  launch API, math intrinsics, sync primitives             │
        └──────────────────────────────────────────────────────────┘
```

**Key decision: SPIR‑V lives in the object file.** Kernels are compiled at build
time and shipped as data sections next to host code. Consequences:

* No runtime JIT, no shader source string in the binary, works offline/air-gapped.
* `libgpu` needs no Clang/LLVM dependency → thin, embeddable, MIT/Apache-clean.
* Cross-TU kernel calls become a link-time problem we must solve explicitly (§6.4).
* Deterministic builds, reproducible SPIR‑V, easy caching by build systems.

**Key decision: LLVM IR is the pivot, not Clang AST → SPIR‑V directly.**
We emit normal LLVM IR from CodeGen using a documented intrinsic subset
(`llvm.spv.*` / our own `gpu.*` intrinsics), then lower with a pass. This lets us
reuse existing scalar optimizations, and keeps the Clang-side patch minimal
(attribute handling + ABI), which is where most forks rot.

---

## 5. Scope of the Clang modification (what we actually touch)

| Area | Change | Est. size | Rebasable? |
|---|---|---|---|
| `include/clang/Basic/Attr.td` | New `GpuExecute` attribute + args (grid, block, shared, private, limits, num_subgroups, requirements…) | XS | yes |
| `Sema/CheckExprLifetime`…/`AttrSema` | Parse/validate args; require external linkage, no varargs, no `[[noreturn]]` mismatch | S | yes |
| `Sema/GPURestrictions.cpp` (**new**) | Restriction checker: reject recursion beyond inline budget, virtual calls, exceptions, globals w/o address-space annotation, unsupported types | M | new file → safe |
| `AST/GPUKernelABI.h` (**new**) | Canonical parameter marshalling: scalars → push constants, spans/ptrs → storage buffers, aggregates → buffer referral | M | new file → safe |
| `CodeGen/TargetInfo` (`SPIRVTargetInfo`) | Kernel calling convention, `entry point` naming, `LocalSize`/`NumWorkgroups` decorations | S | mostly new |
| `CodeGenCGBuilder` hook | Emit kernel body with builtins mapped to `OpConstantComposite`/`BuiltIn` reads | M | localized |
| `llvm/lib/SPIRV/*` (**new pass**) | LLVM IR → SPIR‑V module builder (our own writer, not the third-party LLVM-SPIRV backend we replace/augment) | L | new lib → safe |
| Driver (`clang/lib/Driver/ToolChains`) | `-fgpu-target=vulkan`, section emission, `-fgpu-dump-spv`, link flags for `libgpu` | S | localized |

Everything else in Clang stays untouched. **Success metric: the patch series fits in
fewer than ~25 files and re-applies to the next Clang major within one engineer-week.**

---

## 6. Technical requirements

### 6.1 Language surface (header-only additions)
* `[[gpu::execute(...)]]` — marks a function as a compute entry point.
* Optional companion attributes: `[[gpu::device]]` (callable from kernels, inlined),
  `[[gpu::restrict(...)]]` (extra diagnostics), `[[gpu::require(.../*extension*/))]]`.
* Builtins as functions in namespace `gpu`: `thread_id()`, `block_id()`,
  `thread_idx()`, `group_dim()`, `num_groups()`, `barrier()`, `subgroup_*()`,
  `atomic_*()`, `shared<T>()`, `shuffle/ broadcast`, `clock()`.
* Views: `gpu::span<T>` (device-aware, non-owning), `gpu::array_ref`, `gpu::buffer<T>`.
* Launch API: `gpu::launch`, `gpu::launch_async`, `gpu::submit` (batch/graph),
  `gpu::map/unmap`, `gpu::alloc`, `gpu::device_info`.
* Math: `gpu::fma/sqrt/rsqrt/sin/exp/log/pow/clamp/round*` → GLSL.std.450 instructions.

### 6.2 Attribute parameter vocabulary (initial)
| Parameter | Meaning | SPIR‑V mapping |
|---|---|---|
| `grid = {x[,y[,z]]}` | workgroups, static or symbolic | `OpExecutionMode LocalSize` × block |
| `block = {x[,y[,z]]}` | invocations per workgroup | `LocalSizeId` |
| `shared = N` bytes | static shared memory | `Workgroup` storage class alloc |
| `private = N` bytes | per-invocation stack budget | `Private` + stack-size limit |
| `num_subgroups = N` | required subgroup size | `SubgroupsPerWorkgroup` |
| `max_work_group_size` | validation hint | driver check |
| `extensions = {...}` | required `SPV_*` extensions/enabled capabilities | `OpExtension`/`OpCapability` |
| `name_hint = "..."` | stable entry-point name for tooling | `OpEntryPoint` name |

Unknown/misspelled parameters must produce an error, not be ignored.

### 6.3 Compilation model & restrictions (must be enforced, not assumed)
* Address space model: `global`(StorageBuffer) / `shared`(Workgroup) / `local`(Private)
  explicit or inferred from pointer origin. Implicit conversions between them are errors.
* Types: standard-layout, trivially copyable, no pointers-to-host, size ≤ device limit.
* Control flow: unrestricted loops allowed but must be provably bounded or annotated
  `[[gpu::assume(bounded)]]`; unbounded → warning + optional hard error.
* Functions callable from a kernel: `constexpr`, `[[gpu::device]]`, or templates
  instantiated entirely within the kernel's SCC → force-inlined before SPIR‑V emission
  (SPIR‑V has no first-class function pointers).
* No undefined-behaviour escape hatches: integer overflow in kernels traps in debug
  builds (`-fgpu-sanitizer=undefined`).

### 6.4 Linkage
* Same TU: direct call to `[[gpu::device]]` → inline.
* Cross TU: kernel declared in header, defined in another TU → LTO-mode required in M3,
  automatic in M5 via `-flto` + custom ThinBTI hook. Documented limitation until then.

### 6.5 Runtime semantics
* `gpu::launch` is synchronous-by-default, `launch_async` returns a `gpu::future`.
* Argument marshalling rules are fixed and documented (push-constants ≤ device limit,
  spill to a uniform buffer automatically).
* Streams → `gpu::queue` objects mapping to Vulkan queues/fences; no more than one
  blocking sync per launch unless batched.
* Errors: `gpu::error_code` + `gpu::expected<T>` (no exceptions thrown across the API
  in `-fno-exceptions` friendly mode).
* Device selection: env vars `GPU_DEVICE`, `GPU_FORCE_LLVMPIPE=1`, `GPU_VALIDATE=1`.

---

## 7. Portability guarantee (the "superset-safe" property)

This is what distinguishes us from every prior attempt and is a **hard requirement**:

> Any program that compiles with our Clang must also compile with stock Clang/GCC/MSVC
> using the same header, by selecting a CPU fallback path.

Mechanism: `gpu/gpu.hpp` contains `#if __has_feature(gpu_execute)` guards. Without
compiler support, `[[gpu::execute]]` is an *unknown attribute* (ignored, warning
suppressible) and `gpu::launch` dispatches to a portable serial/thread-pool
implementation over the same body. Consequences:

* CI can build the project with three compilers; the GPU path is an optimization, not
  a fork of the codebase.
* Kernels remain testable on CPU with normal unit tests and coverage tools.
* We inherit C++'s own evolution instead of freezing a dialect.

Fallback costs correctness only, never performance guarantees — documented.

---

## 8. Milestones

Each milestone ends with a runnable demo and a green CI job. Estimates assume 1–2 engineers.

| # | Milestone | Deliverable | Exit criteria |
|---|---|---|---|
| **M0** | Foundations (2w) | Repo layout, pinned LLVM/Clang submodule, patch-series tooling (`git fmt-clang-patch`), CI (build Clang + run lit), Vendored Vulkan headers, `libgpu` skeleton with device enumeration | `ninja clang` + `lit` green; `./bin/gpuinfo` prints devices |
| **M1** | Vertical slice, hardcoded (3w) | Hand-written SPIR‑V module for vector-add loaded by `libgpu`, launched via real Vulkan on Intel/Mesa + NVIDIA; attribute parsed and *validated* by Sema but not yet codegen'd | `examples/vector_add` produces correct results on ≥2 vendors |
| **M2** | End-to-end codegen for the toy subset (6w) | `[[gpu::execute]]` → LLVM IR → SPIR‑V writer for: scalars, pointers, arithmetic, ifs, bounded `for`, `thread_id`, push constants + 1 storage buffer | `saxpy`, `dot product`, `reduction` compile from source and match CPU reference bit-for-bit (with `-ffp-contract=off`) |
| **M3** | Usable compute subset (8w) | Shared memory, barriers, atomics, float math via GLSL.std.450, multidim grids/blocks, `gpu::span` marshalling, pipeline cache, `launch_async`, `-fgpu-dump-spv`, spirv-val in build | `parallel_scan`, `matrix_mul_tiled`, `histogram` pass; perf within 15% of hand-written GLSL compute |
| **M4** | Developer experience (6w) | Restriction-checker diagnostics with fix-it hints, SPIR‑V→source line mapping for validation-layer errors, `-fsanitize=gpu-address`, `gpu::device` allocator, docs/tutorial, CPU-fallback path (§7) working | A newcomer fixes their own rejected kernel using only compiler output |
| **M5** | Scale & robustness (8w) | Cross-TU/LTO kernels, cooperative groups/subgroup ops, fp16/int8, specialization constants for grid sizes, multi-queue submission + capture/replay batching, descriptor-set recycling, memory pool, `vk_ext_scalar_block_layout` independence | Ray-tracing-free benchmark suite (rodinia subset) passes; 10k launches/s overhead < 5 µs |
| **M6** | Hardening & upstream probe (ongoing) | Conformance test suite vs SPIR‑V 1.6 + Vulkan 1.3 compute, fuzzed attribute args, TSan/ASan clean runtime, patch series cleaned for `cfe-dev` RFC, packaging (`pip`/`vcpkg`), spec document | Public repo, reproducible release, upstream review thread opened |

Order rationale: **M1 exists to de-risk the runtime before any compiler work**, because
the highest-probability failure mode of this project is "the compiler half works and
there is no way to observe the result."

---

## 9. Open questions (decide during M0–M2, record answers in `docs/adr/`)

1. **Attribute argument spelling.** Designated initializers in attribute args are a Clang
   extension. Candidates: (a) `[[gpu::execute(grid={1024}, block={32})]]` plain braced
   lists positional; (b) key/value string pairs `[[gpu::execute("grid", 1024)]]`;
   (c) macro-generated `GPU_KERNEL(grid(1024), block(32))`. Decide by testing round-trip
   through all three compilers in §7.
2. **SPIR‑V writer ownership.** Write our own LLVM→SPIR‑V emitter (~8k LOC, full control)
   vs. adopt/vendoring an existing LLVM-SPIRV backend and fixing its gaps. Do a 3-day
   spike in M2 week 1 and decide on measured gap, not reputation.
3. **Memory model default.** Explicit copies (predictable, CUDA-like) vs. coherent
   `VK_EXT_buffer_device_address`/`shaderInt64` unified memory (nicer, less portable).
   Plan: explicit first, unified behind `[[gpu::execute(.unified = true)]]`.
4. **Version pinning.** Track Clang N (last released) or Clang N-1 (stable)? Pin N-1 until
   M5, then move to rolling.
5. **Do we need `spirv-dis`/`spirv-tools` as a build dep?** For `-dump-spv` and CI
   validation: probably yes as a dev-dependency only.
6. **Naming.** `gpu::` collides conceptually with `std::execution` policies and future
   `std::gpu`. Confirm final namespace + header path (`<experimental/gpu>` style?) before
   publishing examples.

---

## 10. Success metrics

| Metric | Target | Measured how |
|---|---|---|
| Language delta | **0** new keywords, **1** public header tree | grep for `Keyword` in our `*.def` patches |
| Unannotated-code regression | **0** behavior changes vs. stock Clang of same version | replay LLVM's `check-clang` suite on our binary |
| Clang patch size | < 25 files, < 6k LOC diff | `git diff --stat` on patch series |
| Rebase cost to next Clang major | ≤ 5 engineer-days | timed exercise in M6 |
| Kernel correctness | 100% of conformance suite vs. CPU reference | CI, per-vendor matrix |
| Performance vs. hand-written GLSL compute | ≥ 85% on 6 benchmark kernels | `bench/` harness |
| Portability | builds+runs on Intel Arc, AMD RDNA, NVIDIA, llvmpipe | CI self-hosted matrix |
| Onboarding | first working kernel in < 30 min | scripted user test on docs |

---

## 11. Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| LLVM→SPIR‑V backend immaturity consumes the schedule | High | Critical | M1 uses hand-written SPIR‑V; M2 week-1 spike decides own-writer vs. adopt; keep scope to the subset in §6.3 |
| Fork rots against moving Clang | Medium | High | Patch series discipline, new-file preference (§5), CI against two Clang majors, upstream early |
| Vendor driver divergence (subgroup size, push-constant limits, fp16) | High | Medium | Capability-driven codegen, per-device limits table, llvmpipe as lowest-common-denominator CI target |
| Silent miscompilation (UB in kernel looks fine on CPU) | Medium | Critical | `-fsanitize=gpu-*`, dual-run differential testing CPU-vs-GPU in CI, `-ffp-contract=off` default for tests |
| Users expect "all of C++ on GPU" | High | Medium | Restriction checker gives actionable errors; §7 CPU fallback makes rejection non-fatal; publish the supported-subset matrix prominently |
| Vulkan complexity leaking into the runtime | Medium | Medium | Keep runtime feature-minimal (non-goals §3.4); validate-layer in debug; no graphics interop ever |

---

## 12. Repository layout (proposed, created in M0)

```
LICENSE                     (Apache-2.0 — already present)
GOALS.md                    this file
README.md                   quick start + status badges
docs/
  spec.md                   normative description of the supported subset
  tutorial.md               "first kernel in 30 minutes"
  adr/                      architecture decision records (answers to §9)
  diagnostics.md            every error/warning ID, cause, fix-it
third_party/
  llvm/                     pinned submodule (our fork lives as a patch series)
  vulkan-headers/           Khronos headers
  spirv-tools/              dev-dependency: validator + disassembler
patches/clang/              ordered patch series applied onto third_party/llvm
include/gpu/                THE public header surface (gpu.hpp + subheaders)
runtime/                    libgpu: loader, device, memory, dispatch, diagnostics
codegen/                    LLVM->SPIR-V pass sources (lives in-tree too, mirrored here)
examples/                   vector_add, saxpy, dot, reduction, scan, tiled_matmul, histogram
tests/
  clang/                    lit tests: attribute parsing, restriction diagnostics
  codegen/                  expected SPIR-V golden files (dis form)
  runtime/                  gtest: loader, marshalling, limits
  conformance/              CPU-reference differential runner, per-vendor matrix
bench/                      rodinia subset, launch-overhead microbench
tools/                      gpuinfo, gpu-dump, gpu-verify, cuda-translator(later)
ci/                         dockerfiles, vendor matrices, check scripts
```

---

## 13. Immediate next actions

1. Agree/adjust this GOALS.md (especially §3 non-goals and §9.1 attribute spelling).
2. M0: scaffold the layout in §12, pin LLVM, stand up CI, write `tools/gpuinfo`.
3. M1 spike: hand-written SPIR‑V vector-add through real Vulkan on the widest hardware
   available — proves the runtime contract before any compiler work begins.
