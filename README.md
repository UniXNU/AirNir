# AirNir

Initial, strict host compiler frontend for AIR 2.6 vertex and fragment modules with the
`air64-apple-ios17.7.0` triple. It parses an unchanged LLVM bitcode input,
validates its module and AIR entry metadata, requires the qualified explicit
fast-math/denorm-disabled options, and lowers supported instructions
to real Mesa NIR. An optional host backend invokes actual IR3 for a caller-supplied
generation-7 chip ID from Mesa's catalog. There are no board names, MMIO addresses,
GPU device handles, page-table operations, guest execution or CPU UI renderer.

This is an incomplete compiler, not a Metal implementation. Do not use successful
compilation as evidence that the GPU executed the shader. Pipeline linking,
live resource upload/layout, shader numerical execution on Adreno,
hardware fragment interpolation/textures, compute stages and complete original-library support remain unqualified.

## Supported and refused

The vertex frontend accepts one straight-line function, a vertex ID
(`ushort` or `uint`) or real float2/float4 vertex attributes, read-only constant
buffers and float-vector position/varying outputs. Attributes retain their
actual source location, including holes. Scalar/float2/float4 constant loads
are lowered with source DataLayout offsets; subtraction is also supported. Supported operations include constant-offset GEPs,
aligned scalar float loads, integer equality/unsigned-greater comparisons,
boolean OR, integer AND, ushort zero extension, selects, vector insertion and
shuffle, aggregate insertion, float addition/division (only division allowing reciprocals) and `air.fma.v2f32`.
LLVM DataLayout computes offsets. Explicit `air.buffer_size` limits remain
checked. When that field is absent, element type size is combined with the
actual static load span and dereferenceable minimum to derive a required
binding extent; an element type size alone is not the whole buffer length.
The schema also records alignment required by metadata/IR loads. Runtime
resource binding must independently enforce these requirements. Dynamic,
negative or excessively large inferred spans refuse. Undefined lanes may be assembled and overwritten, but cannot reach
arithmetic or output. Vertex ID's declared 16-bit width is retained explicitly.

Unsupported instructions, stages, types, globals/function constants, metadata,
nonconstant or invalid buffer accesses are refused. This is a bounded compiler
qualification tool; its parser is not qualified as an untrusted-input service.

Strong FMA is implemented by `FusedF32.h` as integer NIR instruction
construction derived from pinned Berkeley SoftFloat 3e algorithms. The whole
48-bit significand product is retained; exponent alignment uses sticky-bit
shifts, cancellation is normalized, and round-to-nearest/ties-to-even happens
once. Subnormals, signed zeros, overflow and infinity/NaN classification are
handled. NaNs are canonicalized; exception flags and payload compatibility
are not claimed. The compiler lowers 64-bit integer intermediates to real
32-bit GPU arithmetic with the existing NIR integer lowering, validates the
result, then runs actual IR3. These are GPU instructions, not CPU arithmetic
or an unfused float MAD substitution. A remaining raw NIR `ffma` still refuses
export rather than reaching the unsupported IR3 switch.

For constant or selected 0/1 coefficients, calls with all three original
`nnan`, `ninf` and `nsz` flags may use a proven identity: `fma(0,x,z)=z` and
`fma(1,x,z)=x+z` within the finite/no-zero-sign contract. The product introduces
no rounding. Generated additions retain NIR exact controls to prohibit further
contraction or reassociation. Missing any flag or a nonbinary coefficient
retains the generic single-rounding integer sequence. This does not permit an
unfused replacement for arbitrary multiplication. `tests/check_coefficients.py`
compares 4096 cases for this path and each of three missing-flag controls;
required flags are read from the actual call, never inferred globally.

Regular float addition and multiplication conservatively retain each source
rounding step with NIR exact controls, even when the source permits more
optimization. NaN, infinity and zero-sign permissions are copied from the
individual LLVM instruction; module-wide fast-math never supplies them.
Source-permitted `arcp` division remains exact in the pre-backend NIR used
for numerical tests, then legalizes explicitly to reciprocal/multiply before
IR3. Both `.nir` and `.backend.nir` are exported, and the actual legalized
scalar count is reported. Strict division remains refused. This does not
weaken the strong fused-FMA implementation. `tests/check_math_flags.py`
checks separate versus fused rounding, strict/finite/fast operation controls
and permitted versus refused division.

The remaining generic sequence is large; correctness tests do not qualify real-time
UI performance. Hardware execution, cache behavior and scheduling still need
qualification. Varying slots follow metadata order and are provisional;
matching cross-stage interfaces is required before a real pipeline can link.

## Build integration and offline checks

The owning umbrella pins Mesa commit
`85fda3b36405bcaea8bffc64aa4f2651dc90770d` and LLVM 23.1.3. It supplies a small
opt-in Meson integration and the existing host stream/build-ID corrections.
`Frontend.cpp` uses LLVM; `MesaBackend.c` isolates the NIR/IR3 C API. The rendering
drivers and Mesa's LLVM backend remain disabled; LLVM here only parses input IR
and emits the explicit host test oracle.

In the umbrella, run `python3 scripts/build-air-nir-host.py` after its target AIR
audit. The script verifies original input hashes, builds the compiler, compares
original-IR host oracles with NIR constant evaluation, checks actual IR3 assembly
roundtrip and classifies every original module with explicit refusal outcomes.
No original modules, firmware or generated code belong in this repository.

Every successful lowering also writes `.interface.json` with source argument
indices, attribute locations, constant buffer bindings, minimum extents/
alignment, unused arguments and symbolic output identities. An unused marker
contradicting actual SSA use refuses. This is a source/compiler schema, not
qualified Metal descriptor formats, GPU constant upload state, pipeline linkage
or actual runtime bindings.

The compiled `.ir3.json` separately exports the actual IR3 variant state,
including constant allocations/ranges, original NIR read spans, padded immediate
values, raw allocated I/O registers/component masks, shader lengths/alignment,
register/private-memory counts, constant limits and device-specific preamble
policy. Fields are bytes unless explicitly suffixed with another unit. This
export reads the live variant before it is destroyed; its source chip identity
is owned by the compiler context rather than a temporary stack value.
The file omits host pointers and does not claim qualified runtime bindings.

## Native constant preparation

`ConstantUploads.c/.h` is a portable C subroutine for the source/IR3 constant
contract. It has no LLVM, NIR, allocation, device or OS dependency. Callers supply
trusted readable metadata, source bindings with **logical** lengths/minimums,
and a destination image. It copies only source-accessible bytes into declared
promoted UBO intervals and zeros compiler-alignment tails beyond the logical
length. Every recorded source read must remain within actual supplied data and
be covered by a promoted range; padding never substitutes a required source
byte. Required CPU immediate words are copied in little-endian order.

All bounds/coverage/overlap/alias checks precede writes. Refusals return zero
counts and preserve the destination. Other constant allocations, including
GPU-written preamble values and primitive/driver state, are not modified.
A later GPU uploader must submit **only** the declared range/immediate writes;
the entire partially filled image is not a ready draw state. Runtime code still
has to establish resource ownership/lifetimes, actual native-4K mappings,
coherency, command authorization, primitive/pipeline state, fences and display.
This subroutine prepares parameters; it computes no pixels or shader results.

The actual pinned A740 catalog entry does not enable
`load_shader_consts_via_preamble`, even though it is generation 7. Some immediate
pools must therefore be uploaded separately from `.bin`. Source constant accesses
also promote to 64-byte ranges on this device, including original buffers with
only 8/16 logical bytes; copying that rounded range directly from the source
would overread. Both properties are measured in actual compiler state and
checked against retained disassembly, rather than inferred from generation.

The umbrella's `scripts/check-adreno-constant-uploads.py` checks all fourteen
compiled original layouts using inaccessible pages after actual source spans,
verifies every required read/immediate and untouched internal allocation, runs
33 sanitized guard/refusal controls, and cross-builds the C module/static archive
and C++ call boundary for ordinary ARM64/iOS 17. The C++ consumer imports the
same unmangled C symbol. These are host/cross-build observations, not XNU or
GPU execution. Public API metadata must be copied into trusted memory by a
future kernel driver before invoking the helper.

The CLI takes `INPUT.bc CHIP_ID OUTPUT_PREFIX --compile`. It requires
`IR3_SHADER_DEBUG=asmroundtrip` and never opens a GPU. Actual `.bin`, retained
`.asm` and pre-backend `.nir` outputs are host build products. Exit 4 indicates
frontend/target/check refusal; exit 5 indicates backend/export refusal.

`--test-evaluate CASES.bin`, appended to `--compile`, is host verification only.
`--test-evaluate-int32 CASES.bin` runs the same comparisons after integer
lowering, independently of later reciprocal/float optimizations.
Each case contains little-endian uint32 vertex ID followed by the declared
binding-0 buffer bytes (or no bytes for a vertex-only shader), up to 65,536 cases.
It replaces inputs in a NIR clone, uses NIR constant folding and writes float
outputs in four-channel slots to `.eval`; unused channels are zero. The test
mode accepts at most one buffer at argument 0 / binding 0.

`INPUT.bc CHIP_ID OUTPUT.bc --test-oracle` emits a separate, retargeted host test
copy of the original IR, with a stable C test wrapper and explicitly fused LLVM
FMA replacing the named AIR intrinsic. The input file stays unchanged. Its FMA
interpretation must still be checked against the AIR implementation; the oracle
is not evidence of Apple GPU or Adreno instruction execution. It is never a
runtime renderer or deployment fallback.


`--test-resources CASES.bin` uses the actual source interface for a multi-resource
host test. A case contains little-endian uint32 vertex ID, 16 attribute slots
of four uint32 float words each, then each declared constant-buffer block in
source argument order (sizes from the interface). It evaluates after real
integer lowering. Missing bytes refuse; omitted resources are never replaced
with zeros. This format is a test input, not a device command or Metal ABI.

`--test-resource-oracle` writes a separate LLVM host wrapper accepting an
attribute array, a binding-indexed pointer array, vertex ID and float output.
The copy clears fast rewrite permissions and corresponding unsafe FP attributes
to select the same conservative separate-rounding evaluation as the frontend;
otherwise the host could legally fuse the original fast matrix operations.
Mandatory FMA calls remain fused. The input bitcode is unchanged. The umbrella
checks original input hashes and this chosen policy against generated NIR,
using correctly aligned host buffers; generated A740 ISA is not executed.

The authored `tests/fma_vertex.ll` and `tests/check_fma.py` qualify the fused
operation separately from AIR library inputs. The host test checks 131,072
results both before and after integer lowering against native `fmaf`. It
covers the full Cartesian product of 32 chosen special values, random raw
bits, product cancellation, halfway rounding and its neighbors. Two directed
controls distinguish fused arithmetic from separately rounded multiply/add,
including finite cancellation after a product that alone would overflow.
NaN classification is compared; payloads/flags are excluded. This is bounded
sampling/structured coverage, not exhaustive proof over all 2^96 inputs or
execution of the generated GPU ISA.

## License

MIT except `FusedF32.h`, which derives from BSD-3-Clause Berkeley SoftFloat 3e.
Retain and distribute the full notice in `licenses/SoftFloat-3e.txt` with source
and binary derivatives. The owning umbrella records the exact upstream Git
revision and source/license hashes. LLVM and Mesa remain separately licensed
dependencies.

The frontend also accepts strict float32 scalar/vector fragment render-target
outputs and readonly constant buffers in the existing straight-line profile.
Sparse target numbers and MRT return identities are preserved separately from
dense return slots; duplicate/out-of-range targets, nonzero secondary indices,
depth/half outputs, mixed stages and undefined lanes fail explicitly. The
ordinary create API remains vertex; `air_nir_create_stage` selects vertex or
fragment, and ABI/interface exports identify the actual stage. Struct constants
are flattened without inventing values for undefined data. LLVM 23 vector splat
constants must take the vector path even when `dyn_cast<ConstantFP/Int>` matches;
scalar casts are now guarded by scalar type before lane expansion.

Three unchanged original QuartzCore clears compile to A740 fragment code with
actual internal encoding roundtrip, original-IR/NIR bitwise output equality,
correct sparse color(2) and dual MRT identities. Authored nonzero controls cover
all eight color targets and distinct MRT channels so zero clears cannot hide
lane/target swaps. Ten independent refusal controls pass. This is compiler/output
ABI qualification; no fragment rasterization, interpolator/texture state,
render-pass/blend pipeline, live GPU execution or original UI is claimed.

IR3 ABI exports also include actual hardware-rounded branch stack, merged
register mode, input-word count, full-quad and pixel-LOD requirements. These are
retained while the live variant exists, not guessed from assembly names or
binary allocation size. Instruction groups remain separate from padded binary
bytes: the actual code may require an extra aligned allocation group for safe
trailing NOPs without increasing the SP instruction-size register value.

## Bounded original-library input

`Library.h` / `Library.c` provide an allocation-free C reader for the observed
AIR-bearing MTLB profile, including a uniquely selected slice in a big-endian
fat container. `air_library_open` validates all sections, tagged function
records, ordered complete module extents, reflection trailer, LLVM wrappers
and every complete module HASH through a trusted SHA256 callback before
publishing a view. Identical adjacent module aliases reuse the already
verified digest, so cumulative hashing stays within the 64-MiB input bound.
There are at most 4096 functions and 16 tags per record. Pointer arithmetic,
unaligned input, output aliasing and failure atomicity are checked.

`air_library_get` returns original name/stage and exact bitcode/wrapper views
by function index. The caller must keep the opened view and its backing
immutable and alive. No contents are rewritten or copied. These hashes are
integrity checks, not AMFI or signature authority. Unsupported container
profiles refuse; accepting a container does not mean all its AIR compiles.

The CLI accepts `INPUT.metallib CHIP_ID OUTPUT_PREFIX --compile-library INDEX`.
It hashes the complete modules with the same LLVM Support implementation,
parses the original selected bitcode and requires its actual entry name/stage
to agree with the library record. The usual frontend refusals remain active.
The two original UI libraries' 240 function views match the independent audit;
the 17 currently supported entries produce identical A740 machine code,
complete resource ABI and interface files via this direct library path.
The sanitizer test exercises immutable original inputs next to a real final
guard page, unaligned inputs, malformed containers and transactional failures.
`tests/test_library.c` uses Darwin CommonCrypto only as the test hash callback;
the reader itself has no crypto-library or OS dependency.

## Float32 fragment varyings

The frontend now accepts scalar/float-vector fragment varyings with explicit
center sampling and perspective interpolation. It retains exact generated AIR
identities, rejects duplicate/missing identities and assigns dense local input
slots separately from those identities. Actual NIR uses smooth pixel
barycentrics and load_interpolated_input; IR3 retains its real padded input
locations and barycentric system value. Pipeline linking still has to match
these identities to actual vertex exports. An unused float4 position with
center/no-perspective metadata is accepted without inventing a position value;
a used position remains a refusal.

Scalar air.fast_saturate.f32 lowers only with that instruction's explicit
nnan/ninf/nsz permissions. The host numerical oracle uses LLVM min/max on
finite values; NIR tests consume supplied post-interpolation values. They do
not compute barycentrics, rasterize primitives or qualify hardware interpolation.
Five original QuartzCore path/stroke fragments each pass 4096 original-IR/NIR
numerical cases and real IR3 assembler roundtrip, with exact original varying
identities and component masks. Linear/flat/centroid/sample modes, half inputs,
used positions and weaker math contracts remain unqualified/refused.
