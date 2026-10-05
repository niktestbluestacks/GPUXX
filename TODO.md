# TODO — The Complete Build Plan

Companion to `GOALS.md`. Every checkbox is a concrete, verifiable step. Phases map to
milestones M0–M6 in GOALS.md §8; each phase ends with an **exit gate** that must be
demonstrably true before moving on. Line references point into the vendored tree at
`third_party/llvm-project` (tag `llvmorg-21.1.3`, branch `gpu-execute`).

Legend: `[ ]` open · `[x]` done · `(S/M/L)` rough effort · `(!)` de-risking / decision step

---

## Phase 0 — Foundations (M0)

### 0.1 Toolchain & environment (S)
- [x] Vendor Clang/LLVM 21.1.3 sources (`third_party/llvm-project`, shallow clone)
- [x] Document build procedure (`docs/BUILDING.md`)
- [ ] Install host deps: `cmake ninja-build ccache zlib1g-dev libzstd-dev libedit-dev libncurses-dev pkg-config`
- [ ] Configure LLVM+clang+lld (`-DLLVM_TARGETS_TO_BUILD="X86;SPIRV"`, assertions ON)
- [ ] First full `ninja clang` build completes; `build/bin/clang++ --version` prints 21.1.3
- [ ] Smoke test: `build/bin/clang++ tests/hello.cpp && ./a.out` → "hello from cpu"
- [ ] Add `-fsyntax-only` regression script that replays upstream lit subset later

### 0.2 Patch-series discipline (S)
- [ ] Create orphan branch layout in the vendored clone: pristine tag = baseline, work on topic branches
- [ ] Write `tools/gp-patch export|apply|check` wrappers around `git format-patch` / `git am`
      (numbered patches under `patches/NNN-*.patch`, one logical change per patch)
- [ ] CI job A: apply patch series onto fresh clone → build → run our lit tests
- [ ] CI job B: unpatched build passes `tests/hello.cpp` + `check-clang-codegen` subset
      (proves we don't break stock behavior — GOALS §10 metric "unannotated-code regression = 0")

### 0.3 Vulkan tooling bootstrap (S)
- [ ] Vendor `KhronosGroup/Vulkan-Headers` → `third_party/vulkan-headers`
- [ ] Vendor/build `spirv-tools` (validator + disassembler) → `third_party/spirv-tools`; also install system copy for CLI use
- [ ] Vendor `SPIRV-Headers` (needed by spirv-tools and by our runtime's hand-written .spv assembly fallback)
- [ ] Obtain a Vulkan device in the dev/CI environment, in priority order:
      discrete GPU driver → Intel ANV → RADV → **llvmpipe/radv via Mesa** (software, always works headless).
      Verify with a vendor sample or `vulkaninfo-sdk`.
- [ ] Env knobs reserved (implemented in Phase 1): `GPU_DEVICE`, `GPU_VALIDATE=1`, `GPU_FORCE_LLVMPIPE=1`

### 0.4 Public header skeleton (S)
- [ ] `include/gpu/gpu.hpp` — umbrella header, feature-detection block:
      `#if defined(__has_feature) ... __has_feature(gpu_execute)` → sets `GPU_COMPILER_SUPPORT`
- [ ] Declare attribute namespace stubs so the header parses on *stock* clang/gcc/msvc
      (attributes unknown-but-harmless; builtins resolve to CPU fallback decls)
- [ ] Decide & freeze the **attribute spelling** (GOALS §9.1). Recommendation to test first:
      `[[gpu::execute(grid=(1024,1,1), block=(32,1,1), shared=4096)]]` — key/value form inside a
      standard attribute-list is parseable as an expression list if we define the args ourselves in Attr.td;
      round-trip all candidates through stock clang++, g++, (msvc if available) and record the winner in
      `docs/adr/0001-attribute-spelling.md`. **( ! )**

**Exit gate 0:** patched-and-unpatched builds green; `ninja check-gpu` target exists (even if empty);
`spirv-val` runs; a Vulkan context can be created from C++ on ≥1 platform.

---

## Phase 1 — Runtime vertical slice with NO compiler changes (M1, part 1)

Goal: prove the entire host-side contract using a **hand-written SPIR-V module**. This deletes
all downstream risk of "compiler works but nothing can execute the result."

### 1.1 Hand-written vector-add shader (S)
- [ ] Author `examples/vector_add/vector_add.spvasm` (or via glslang from GLSL) implementing
      `y[i] = a*x[i] + y[i]`, LocalSize(32,1,1), one push constant (float a), three storage buffers
- [ ] Assemble → `vector_add.spv`; pass `spirv-val --target-env vulkan1.3`
- [ ] Commit both `.spvasm` source and binary; document how to regenerate

### 1.2 libgpu core — device layer (M)
- [ ] `runtime/src/device.cpp`: instance creation, extension gating (`VK_KHR_get_physical_device_properties2` only if pre-1.1),
      physical-device selection heuristic (discrete > integrated > llvmpipe; score by `VkPhysicalDeviceType` + local-memory),
      logical device with exactly one compute-capable queue family (+ transfer family if separate)
- [ ] Minimal `VkAllocationCallbacks` policy (default allocator for now)
- [ ] Capability table capture: max push-constant size, subgroup sizes, buffer alignment, address-model — stored for later codegen/runtime queries
- [ ] `tools/gpuinfo`: enumerate devices, print name/type/api version/limits (this is the M0 demo too)

### 1.3 libgpu core — memory & dispatch (M)
- [ ] Buffer abstraction: `gpu::runtime::buffer` wrapping VkBuffer + dedicated VMA-less allocation
      (host-visible|device-local flags), correct memory-property queries
- [ ] Descriptor set layout (binding 0..N storage buffers), pipeline layout (push-constant range),
      compute pipeline creation from raw SPIR-V `VkShaderModule`
- [ ] One-time submit path: allocate cmd buffer → bind pipeline/descriptors → push constants →
      `vkCmdDispatch(x,y,z)` → submit → fence wait → readback
- [ ] Error handling: `gpu::error_code` mapping from VkResult; validation-layer auto-enable when `GPU_VALIDATE=1`

### 1.4 Kernel metadata container (M)
- [ ] Define the object-file contract NOW (it constrains the compiler ABI): ELF section
      `.gcc.gpu_kernels` holding a fixed-header blob table:
      `{magic, version, entry_name, local_size[3], num_push_constant_bytes, descriptor_count, spv_blob_len, spv_blob[]}`
      plus a `.note.gpu.abi` note recording our ABI version.
- [ ] Writer side: prototype in a tiny host tool `tools/gpu-embed` that injects a .spv into a .o/.a
      (via `objcopy --add-section` initially) so the runtime loader has something to read before CodeGen emits it
- [ ] Reader side: `runtime/src/loader.cpp` scans sections of its own executable/archive inputs
      (link-time registered via linker script or explicit `gpu::register_kernel()` ctor fallback for portability)
- [ ] Unit test: embed vector_add.spv → load by name → launch → compare vs CPU reference

**Exit gate 1:** `examples/vector_add` links against libgpu, loads the embedded SPIR-V, produces
correct results on llvmpipe AND at least one real vendor driver. No compiler was modified to get here.

---

## Phase 2 — Compiler: attributes & semantics (M1 part 2 / M2 front-end)

### 2.1 Attribute declaration (S)
- [ ] `clang/include/clang/Basic/Attr.td`: add
      `def GpuExecute : InheritableAttr { let Spellings = [CXX11<"gpu","execute">]; let Args = [...]; let Subjects = SubjectList<[Function]>; }`
      plus companions `GpuDevice`, `GpuRequire`, `GpuAssumeBounded` (new file `GPURelatedAttrs.inc` if include-splitting helps rebasing)
- [ ] Regenerate; verify `clang -cc1 -dumptu` shows parsed attr; add lit test
      `tests/clang/attr-parse-basic.spt` style: accept/reject matrix (on functions ✓, on variables ✗, on lambdas ✗ v1)
- [ ] Arg parsing: integer literals, braced lists `{x,y,z}`, string literal `name_hint`, identifier keys.
      Unknown keys ⇒ error listing valid keys (GOALS §6.2 "unknown parameter must error").

### 2.2 Sema validation (M)
- [ ] `clang/lib/Sema/SemaGPU.cpp` (**new file**, keeps diff small):
      external linkage required; no varargs; return type void-or-standard-layout; params restricted to
      trivially-copyable/scalar/span types; reject `[[noreturn]]`, virtual overrides, friend-of-class templates for now
- [ ] Duplicate-attribute detection; conflicting grid/block dims (> device limits are runtime-checked, not compile errors)
- [ ] Diagnostic IDs centralized in `DiagnosticGPUKinds.def`; every diagnostic gets a lit test with exact message
- [ ] Implicit registration: emit `__gpu_kernel_ref_<mangled>` global pointing at function + metadata struct
      consumed by our backend (works even without custom sections)

### 2.3 Restriction checker (M) — the quality-of-implementation linchpin
- [ ] AST walker `clang/lib/Sema/GPURestrictions.cpp`: reject (with fix-it text) — recursion beyond inline budget,
      virtual calls, `throw`/`try`, `new`/`delete`, `std::function`, RTTI (`dynamic_cast`/`typeid`), coroutines,
      host pointers escaping, non-standard-layout captures
- [ ] Whitelist mechanism: `[[gpu::device]]` callees and fully-instantiable templates allowed
- [ ] Table-driven "supported subset matrix" generated from the same source as docs/spec.md (single truth)
- [ ] Negative test suite: ~40 mini programs, each rejected for exactly one reason

**Exit gate 2:** `[[gpu::execute(...)]]` parses, validates, and rejects the documented bad cases with
actionable diagnostics; zero regressions in `check-clang-*` subsets.

---

## Phase 3 — CodeGen interception (M2)

### 3.1 Host-side marshalling emission (M)
- [ ] For each kernel, synthesize a host stub `__gpu_host_<mangled>(args...)` that packs scalars into
      push-constant struct, records pointer/array args as descriptors, fills the metadata blob, calls `gpu::rt_launch(blob_id, ...)`
- [ ] `clang/lib/CodeGen/CGGPU.cpp` (**new file**) hooks `CodeGenerator::StartFunction` when kernel attr present
- [ ] Argument classification rules frozen in `docs/adr/0002-kernel-abi.md`: scalar→push const; span/ptr→storage buffer; aggregate-by-value→referral buffer (spill uniform buffer when push-const budget exceeded)

### 3.2 Device-side IR generation (M)
- [ ] When compiling the *kernel body*, switch to a SPIR-V-compatible lowering mode:
      either (a) reuse existing triple `spirv64-unknown-unknown` path (upstream already supports
      `SPIRVTargetInfo` at `clang/lib/Basic/Targets/SPIR.h:357` and builtins in `BuiltinsSPIRV{Base,CL,VK}.td`),
      or (b) stay on host triple + annotate calling convention `spir_kernel` equivalent and lower later.
      Spike both in week 1, decide in `docs/adr/0003-codegen-route.md`. **( ! )**
- [ ] Map `gpu::thread_id()/block_id()/group_dim()/num_groups()` to upstream SPIR-V builtins
      (`__builtin_spv_*` already exist in `clang/lib/Headers/__clang_spirv_builtins.h` — reuse, don't reinvent)
- [ ] Force-inline all `[[gpu::device]]` callees before backend (SPIR-V has no function-pointer graph crossing)
- [ ] Address-space plumbing: default→Private, kernel params→StorageBuffer(4300-style via CL AS map already in SPIR.h),
      `gpu::shared<T>()`→Workgroup; implicit AS conversions = hard error

### 3.3 Backend: LLVM IR → SPIR-V (L) — biggest schedule risk
- [ ] Week-1 spike (GOALS §9.2): evaluate the in-tree `llvm/lib/Target/SPIRV` (present in 21.1.3:
      `SPIRVAsmPrinter.cpp`, `SPIRVGlobalRegistry.cpp`, `SPIRVEmitIntrinsics.cpp`, MC emitter) against our subset:
      does it produce validator-clean .spv for arithmetic/if/for/buffer access? Measure gap.
- [ ] Decision A — adopt & extend in-tree backend (preferred if gap < ~2k LOC): add missing patterns as patches
- [ ] Decision B — write `llvm/lib/SPIRVLower/` module emitter (~8k LOC) consuming LLVM IR directly;
      keep as new-library patch (rebase-safe)
- [ ] Either way: golden-file tests `tests/codegen/*.ll → *.spvasm.expected`, validated with `spirv-val` in CI
- [ ] Entry-point emission: `OpEntryPoint GLCompute`, `LocalSize` decoration from attribute, stable names via `name_hint`
- [ ] Section emission hook: attach final .spv blob + metadata to object as `.gcc.gpu_kernels` (MC layer or post-process pass in driver)

### 3.4 Driver integration (S)
- [ ] Options: `-fgpu-target=vulkan`, `-fgpu-dump-spv=<path>`, `-fgpu=cpu-fallback`, `-Wno-ignored-gpu-attributes`
- [ ] Link step adds `libgpu.a` automatically when kernels present
- [ ] `--help` entries + docs

**Exit gate 3 (the project's turning point):** `saxpy.cpp` written in plain C++ with one attribute
compiles end-to-end with our clang, embeds SPIR-V in the object, launches via libgpu, matches the CPU
reference bit-for-bit under `-ffp-contract=off`. Also: dot product + reduction pass. Stock-clang
regression suite still green.

---

## Phase 4 — Usable compute subset (M3)

- [ ] Shared memory: `gpu::shared<T[]>(n)` + static `shared=N` budget → Workgroup storage class, alignment checks
- [ ] Barriers: `gpu::barrier(mem|workgroup)` → OpControlBarrier; document memory-order guarantees
- [ ] Atomics: `gpu::atomic_add/min/max/exchange/cas` over StorageBuffer & Workgroup (GL.std.450-free, native ops)
- [ ] Float math library → GLSL.std.450 instructions (`fma/sqrt/rsqrt/sin/cos/exp/log/pow/clamp/floor/ceil/trunc/round`)
- [ ] Multi-dimensional grids/blocks; runtime-variable grid override in `gpu::launch(..., gpu::grid{...})`
- [ ] `gpu::span<T>` marshalling complete (bounds carried as specialization info, not in-shader checks for perf)
- [ ] Pipeline cache on disk keyed by hash(spv+device-id); `launch_async` returning `gpu::future`; event-based batching
- [ ] `parallel_scan` (Blelloch), `matrix_mul_tiled`, `histogram` examples pass correctness + within 15% of hand-written GLSL
- [ ] `spirv-val` wired into build as mandatory gate; `-fgpu-dump-spv` ergonomic review

**Exit gate 4:** benchmark harness `bench/` reports numbers for 6 kernels on 2 vendors; no known miscompilations.

---

## Phase 5 — Developer experience (M4)

- [ ] Diagnostics audit: every restriction error carries cause + fix-it ("replace std::cout with gpu::printf (requires SPV_debug_printf)")
- [ ] SPIR-V → source line mapping: keep debug-info through backend; translate validation-layer errors back to file:line
- [ ] `-fsanitize=gpu-address` (address-space misuse trap) and `-fsanitize=gpu-undefined` (overflow traps) via compiler-rt-ish shader instrumentation
- [ ] `gpu::alloc/gpu::free` device allocator (VkMemory requirements honored; leak report at teardown)
- [ ] CPU fallback path (§7 of GOALS): `gpu::launch` executes serial/threadpool over same AST-derived body —
      means headers provide `constexpr`-evaluable builtins (`thread_id()` reads thread-local cursor). Validate with GCC/MSVC CI legs.
- [ ] Tutorial `docs/tutorial.md` "first kernel in 30 minutes"; newcomer usability test recorded
- [ ] IDE story: ensure `compile_commands.json` unaffected; document VSCode/clangd setup

**Exit gate 5:** a person unfamiliar with the project fixes a rejected kernel using only compiler output.

---

## Phase 6 — Scale & robustness (M5)

- [ ] Cross-TU kernels via LTO: kernel registry survives ThinLTO; `-flto` path tested (documented limitation until now)
- [ ] Cooperative groups: `subgroup_size/shuffle/broadcast/all/any/match_*` mapped to `SPV_KHR_subgroup_*` capabilities
- [ ] fp16 (`SPV_FSHINT`/Float16Int8 capability) + int8 paths; specialization constants for grid dims
- [ ] Multi-queue submission, command-buffer capture/replay, descriptor-set recycling, slab memory pool
- [ ] Independent of `VK_EXT_scalar_block_layout` (test packed struct layouts manually)
- [ ] Rodinia-subset conformance suite (≥8 kernels) differential-tested vs CPU references
- [ ] Launch overhead microbench: < 5 µs amortized over 10k launches
- [ ] Rebase drill onto Clang 22.x branch: time-boxed ≤ 5 engineer-days, log pain points in `docs/adr/`

**Exit gate 6:** conformance green on Intel Arc + AMD RDNA + NVIDIA + llvmpipe in CI matrix.

---

## Phase 7 — Hardening & upstream probe (M6+)

- [ ] Conformance test suite vs SPIR-V 1.6 + Vulkan 1.3 compute feature matrix
- [ ] Fuzz attribute arguments (libFuzzer harness on Sema parse path); TSan/ASan clean runtime
- [ ] Spec document `docs/spec.md` normative; `docs/diagnostics.md` auto-generated from TD defs
- [ ] Packaging: CMake superbuild, `vcpkg` port, pip wheel shipping gpuclang + headers + runtime
- [ ] Clean patch series (≤25 files, ≤6k LOC diff) → RFC on cfe-dev/discourse.llvm.org
- [ ] (Separate tool, optional) `cuda-translator` best-effort `.cu` → annotated C++ rewriter

---

## Decisions ledger (fill in as resolved → `docs/adr/`)

| # | Question | Default/proposal | Status |
|---|---|---|---|
| 1 | Attribute arg spelling | `key=(list)` / `key=val` inside `[[gpu::execute(...)]]` | ☐ ADR-0001 |
| 2 | Codegen route: host-triple intrinsics vs offload spirv64 triple | spike both | ☐ ADR-0003 |
| 3 | SPIR-V writer: extend in-tree `llvm/lib/Target/SPIRV` vs own `SPIRVLower` | prefer in-tree if gap < 2k LOC | ☐ ADR-0004 |
| 4 | Memory model default | explicit copies; unified behind opt-in attr | ☐ ADR-0005 |
| 5 | Kernel metadata transport | ELF section + ctor-registration fallback | ☐ ADR-0006 |
| 6 | Namespace/header naming | `<gpu/gpu.hpp>`, namespace `gpu` | ☐ ADR-0007 |

## Risk register (live during execution)

| Risk | Trigger to watch | Pre-agreed response |
|---|---|---|
| In-tree SPIR-V backend insufficient (Phase 3.3) | spike fails on basic loops/buffer access | fall back to own emitter (Decision 3-B); do NOT attempt upstream fixes mid-schedule |
| No GPU in dev/CI env | `vulkaninfo` fails | llvmpipe software driver is mandatory floor; never block on hardware procurement |
| Attribute grammar fights parser (Phase 0.4/2.1) | chosen spelling won't parse cleanly | downgrade to macro-wrapped positional ints (candidate (c) in GOALS §9.1) |
| Scope creep toward "all C++ on GPU" | restriction-checker bug count rising | freeze supported-subset matrix (Phase 2.3); new constructs require ADR |

---

## Current status snapshot (update every session)

- Done: repo skeleton, LLVM 21.1.3 vendored, GOALS.md, BUILDING.md, hello-world smoke test committed.
- Next up: **Phase 0.1 remaining items** (full clang build) → **0.2 patch tooling** → **0.3 Vulkan tooling**.
- Blockers: none known; first build duration is the long pole (hours on small VMs).
