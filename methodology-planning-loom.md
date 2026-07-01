# Methodology Planning Phase: Project Loom

This document defines the production-grade Methodology Planning Phase for Project Loom. It maps the hybrid V-Model/Iterative Software Development Life Cycle (SDLC) into an executable system-engineering plan for deployment under .NET 10 Native AOT.

---

## 1. Phase Decomposition

```
[ V-MODEL LEFT-SIDE DESIGN & EXECUTION MATRIX ]

         Stage A: Requirements & Threat Modeling
                         │
                         ▼
           Stage B: AOT-Compliant Design
                         │
                         ▼
         Stage C: Incremental Implementation
           ├── C1. Loom.SSH (Port 2222 Engine)
           ├── C2. Loom.TUI (IAnsiConsole Streams)
           ├── C3. Loom.Core (AVX2/Neon SIMD)
           ├── C4. Loom.Storage (Memory-Mapped Files)
           └── C5. Loom.Storage (Utf8Json Ingestion)
                         │
                         ▼
            Stage D: Static & Trim Analysis
                         │
                         ▼
          Stage E: CI/CD & Build Infrastructure
```

---

### Stage A: Requirements Engineering & Threat Modeling

#### Entry Criteria
*   Project Loom Build Plan initiated; solution configuration files (`Loom.sln`, individual `.csproj` configurations) drafted.

#### Exit Criteria
*   Security architecture sign-off on the STRIDE Threat Model.
*   Zero-allocation Service Level Agreements (SLAs) defined for Phase 1 (Parser) and Phase 5 (Ingestor).

#### Required Artifacts
*   **STRIDE Threat Matrix Document:** Outlining precise trust boundaries and mitigation strategies for raw TCP inputs.
*   **Interface Contract Stubs:** Zero-allocation API boundaries declared using `ReadOnlySpan<byte>` parameters.

#### Concrete Tasks
1.  Map the trust boundaries between the external SSH client, the target runtime host, and the Loom execution environment.
2.  Define maximum buffer limits for incoming raw TCP payloads within the `Loom.SSH` socket reader to mitigate buffer exhaustion attacks.
3.  Formulate memory consumption constraints and throughput target profiles for the vector processing loops.

#### Roles & Responsibilities
*   **Lead Systems Architect:** Define structural boundary interfaces and zero-allocation target profiles.
*   **Security Architect:** Execute STRIDE modeling and validate raw socket buffer limit specifications.

---

### Stage B: Native AOT-Compliant System Design

#### Entry Criteria
*   Stage A exit criteria fulfilled.

#### Exit Criteria
*   Static code analyzer setup showing zero dynamic compilation pathways.
*   Design sign-off confirming no dynamic reflection or late binding in critical paths.

#### Required Artifacts
*   **Assembly Dependency Graph:** Directing target dependency layout (`Loom.Host` $\to$ `Loom.TUI` $\to$ `Loom.SSH` $\to$ `Loom.Storage` $\to$ `Loom.Core`).
*   **Struct Memory Layout Schema:** Explicit structure definitions detailing byte alignment and packing constraints.

#### Concrete Tasks
1.  Audit third-party dependencies for AOT compatibility; reject libraries containing dynamic IL generation (`System.Reflection.Emit`).
2.  Design code-generated alternatives or compile-time metadata registration schemas to replace dynamic runtime type resolution.
3.  Design structural alignments for memory-mapped file persistence using natural power-of-two boundaries (4-byte alignment for floats, 8-byte for doubles).

#### Roles & Responsibilities
*   **Principal AOT Engineer:** Review third-party libraries for IL trimming readiness and define reflection-free serialization contracts.
*   **Lead Systems Architect:** Layout solution-wide physical code architecture and interface abstractions.

---

### Stage C: Incremental Implementation

```
┌────────────────────────────────────────────────────────────────────────┐
│                      Stage C: Execution Sequence                       │
├────────────────────────────────────────────────────────────────────────┤
│  [C1] Loom.SSH ──> [C2] Loom.TUI ──> [C3] Loom.Core ──> [C4/C5] Storage│
└────────────────────────────────────────────────────────────────────────┘
```

#### Sub-Stage C1: SSH Infrastructure (`Loom.SSH`)
*   **Entry Criteria:** Stage B design specifications verified.
*   **Exit Criteria:** Secure SSH channel operating on port 2222, verified to handle 1,000 concurrent client connections under strict memory bounds.
*   **Required Artifacts:** Verified cryptographic state machine; non-allocating packet parsing pipeline.
*   **Concrete Tasks:**
    1.  Implement socket reader layer utilizing `ValueTask` async methods with recycled memory blocks.
    2.  Write ASN.1 BER/DER parser loops using `System.Formats.Asn1` targeting `ReadOnlySpan<byte>` spans.
    3.  Implement SSHv2 transport layer handshakes (KEX, host keys, ciphers) backed by safe `.NET Cryptography` APIs.
*   **Roles:** **Lead Network Engineer** (Socket layer, protocol implementation), **Security Engineer** (Cryptographic review, buffer validation).

#### Sub-Stage C2: TUI Render Engine (`Loom.TUI`)
*   **Entry Criteria:** Sub-stage C1 exit criteria fulfilled.
*   **Exit Criteria:** Interactive console output successfully rendered over redirected network stream; resize packets handled dynamically without heap allocations.
*   **Required Artifacts:** Custom stream-backed `IAnsiConsole` driver; window-change event distribution loops.
*   **Concrete Tasks:**
    1.  Create stream translation classes converting Spectre.Console output sequences directly to network write buffers.
    2.  Implement packet event interceptor in `Loom.SSH` to process client `window-change` terminal resize sequences.
    3.  Develop view-render widgets utilizing static type converters, avoiding reflection-based property pathways.
*   **Roles:** **TUI Developer** (Component UI, layout engines), **Lead Network Engineer** (Network-to-TUI stream redirection).

#### Sub-Stage C3: SIMD Math Engine (`Loom.Core`)
*   **Entry Criteria:** Sub-stage C2 exit criteria fulfilled.
*   **Exit Criteria:** Core telemetry search subsystem executing vector comparison loops, achieving a target of $10^7$ records parsed per second.
*   **Required Artifacts:** Platform-portable vector algorithm implementations; CPU hardware intrinsic target branches.
*   **Concrete Tasks:**
    1.  Implement portable vector algorithms using `System.Numerics.Vector<T>`.
    2.  Code optimized platform-specific intrinsic alternatives (such as `Vector256<float>` / `Avx2`) wrapped in explicit validation checks (`Avx2.IsSupported`).
    3.  Implement standard scalar loop fallbacks to handle arbitrary remainder array tail elements.
*   **Roles:** **SIMD Engineer** (Hardware intrinsic design, compiler optimization tracking).

#### Sub-Stage C4: Binary Embedding Cache (`Loom.Storage`)
*   **Entry Criteria:** Sub-stage C3 exit criteria fulfilled.
*   **Exit Criteria:** Safe zero-copy memory-mapped file (MMF) parser operating safely without memory leaks on thread disposal.
*   **Required Artifacts:** Natural alignment structured indices; safe pointer-to-span wrappers.
*   **Concrete Tasks:**
    1.  Define serialized cache structures using natural layout sizing (no `Pack = 1` configurations where misaligned field access degrades performance).
    2.  Develop safe memory-mapped stream loaders wrapping pointer access inside safe `try...finally` resource reclamation paths.
    3.  Replace any runtime `Marshal.SizeOf<T>()` dynamic calls with compile-time evaluated `Unsafe.SizeOf<T>()`.
*   **Roles:** **Storage Engineer** (Memory-mapped cache files, layout structures), **Systems Engineer** (Pointer safety validation).

#### Sub-Stage C5: RAG Telemetry Ingestor (`Loom.Storage`)
*   **Entry Criteria:** Sub-stage C4 exit criteria fulfilled.
*   **Exit Criteria:** High-throughput JSON document parsing stream executing with zero heap allocations on the hot path.
*   **Required Artifacts:** Zero-allocation stream parsing engine; path-context tracking buffers.
*   **Concrete Tasks:**
    1.  Develop streaming JSON parser pipelines using `Utf8JsonReader` processing chunked binary blocks.
    2.  Implement path context builders utilizing rented heap structures via `ArrayPool<char>` to construct parent-child context strings.
    3.  Integrate the parsed document stream outputs with Phase 3 vector calculations.
*   **Roles:** **Database/Ingestion Engineer** (JSON state machine, heap allocation tracking).

---

### Stage D: Static & Trim Analysis (Compilation Verification)

#### Entry Criteria
*   Compilation of all assemblies completes with default configuration settings.

#### Exit Criteria
*   Build pipeline generates 100% warning-free outputs under strict compiler analyzers (`EnableTrimAnalyzer`, `EnableAotAnalyzer`).

#### Required Artifacts
*   **Trim Remediation Record:** Documenting all analyzer warnings and their applied code mitigations.
*   **Configuration Files:** Build-time custom trimming directive XML files (`rd.xml`).

#### Concrete Tasks
1.  Enable aggressive trimming options within the project configurations.
2.  Audit every analyzer warning; remediate virtual generics or dynamic properties with static descriptors.
3.  Inject source-generator compilation instructions for all JSON serialization requirements.

#### Roles & Responsibilities
*   **Static Analysis Lead:** Trace compiler warnings to concrete source locations and design resolution models.
*   **Build Engineer:** Manage target workspace parameters, global tool execution, and artifact compilation outputs.

---

### Stage E: CI/CD Pipeline & Build Infrastructure Integration

#### Entry Criteria
*   Codebase compiles cleanly in developer local test environments.

#### Exit Criteria
*   Automated CI execution triggers on code submit, performing dual-mode test runs, compilation on both Linux and Windows agents, and artifact code-signing.

#### Required Artifacts
*   **CI Configuration Pipelines:** Declarative target configurations (.yml files) mapping target test executions and build actions.
*   **Production Deployment Packages:** Cryptographically signed, self-contained native binary targets.

#### Concrete Tasks
1.  Configure CI container environments targeting development tool requirements (Clang 19, MSVC v143 build toolchains).
2.  Set up dual-engine testing tasks (running standard JIT test runs and native AOT compiled test executables).
3.  Integrate code-signing pipeline routines using target code authority certificate sets.

#### Roles & Responsibilities
*   **DevOps/Build Engineer:** Build automation scripts, toolchain environment verification, code-signing integration.

---

## 2. Schedule and Sequencing

This schedule enforces sequentially checked quality gates. AOT validation occurs inside the design and implementation cycles, rather than as a post-facto verification step.

```
[ DEVELOPMENT SPRINT TIMELINE ]

Weeks:      1-2     3-4     5-6     7-8     9-10    11-12
Stage A:    [██]
Stage B:            [██]
Stage C1-C2:                [██]
Stage C3-C5:                        [██]
Stage D:                                    [██]
Stage E:                                            [██]
```

### Sprint 1 (Weeks 1-2): Stage A (Requirements & Threat Modeling)
*   **Deliverables:** Completed STRIDE Threat matrix; baseline zero-allocation interfaces defined.
*   **Verification:** Architecture review of communication protocols; verification of raw socket buffer allocations.

### Sprint 2 (Weeks 3-4): Stage B (AOT Design & Architecture)
*   **Deliverables:** Assembly architecture diagrams; explicit structure definitions; target system library reviews.
*   **Verification:** Dependency validation scan; check for dynamic IL generation tools in dependency lists.

### Sprint 3 (Weeks 5-6): Stages C1-C2 (SSH & TUI Foundation)
*   **Deliverables:** Socket engine executing on port 2222; ANSI terminal stream parser; UI layout engines.
*   **Verification:** Execution of custom terminal connection tests; check of allocation behavior during network events.

### Sprint 4 (Weeks 7-8): Stages C3-C5 (SIMD, Storage, & Ingestor)
*   **Deliverables:** Memory-mapped data loader; SIMD query execution vectors; JSON stream parsers.
*   **Verification:** Run target vector tests on both x64 and ARM64 platforms; capture memory allocations under database ingestion loads.

### Sprint 5 (Weeks 9-10): Stage D (Static Code & Trim Verification)
*   **Deliverables:** Trim-warning-free codebase; custom trim XML declarations (`rd.xml`).
*   **Verification:** Run build analyzer checks with compilation flags set to treat trim warnings as compilation errors.

### Sprint 6 (Weeks 11-12): Stage E (CI/CD Pipeline Setup & System Launch)
*   **Deliverables:** Build workflow pipelines; binary code-signing configurations; final system release packages.
*   **Verification:** Verify dual-mode test runs execute without errors on both Linux and Windows runner targets.

---

## 3. Risk Register

| Risk ID | Risk Description | Affected Stages | Mitigation Action |
| :--- | :--- | :--- | :--- |
| **R-01** | **AOT Trimmer breaks reflection dependency**<br>The native compiler strips out metadata used internally by third-party libraries (such as `Spectre.Console` CLI), leading to target exceptions. | Stage B, C2, D | Apply compiler annotation attributes (`[DynamicallyAccessedMembers]`) or include custom explicit compilation rules in `rd.xml` to keep required metadata. |
| **R-02** | **Hardware vector execution crashes**<br>Using direct SIMD calls (e.g., `Vector256<float>`) on unsupported systems triggers runtime failures. | Stage C3 | Wrap all direct platform intrinsics inside checks (like `Avx2.IsSupported`). Provide portable `Vector<T>` or scalar fallbacks for unsupported systems. |
| **R-03** | **Memory alignment faults on ARM64**<br>Reading unaligned structs (using `Pack = 1`) on ARM64 platforms causes slow reads or program crashes. | Stage C4 | Structure cache database files with natural field boundaries (align fields to their natural size offsets) and avoid `Pack = 1` rules. |
| **R-04** | **Unmanaged memory leaks in pointer routines**<br>Failures during pointer access (inside `AcquirePointer`) leak memory-mapped file handles over long execution periods. | Stage C4 | Enforce strict usage of `try...finally` structures to release pointers via `ReleasePointer` immediately after execution finishes. |
| **R-05** | **Path context allocation storms**<br>Recursive parsing during JSON data ingestion creates string allocations on the heap, breaking the zero-allocation target. | Stage C5 | Use rented scratch buffers via `ArrayPool<char>` and manipulate data using `Span<char>` slices rather than instantiating new string objects. |

---

## 4. Toolchain and Environment Specification

The toolchain versions listed below are required across development environments and automated build pipelines to prevent compiler errors.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Required Software Stack                         │
├────────────────────────────────────────────────────────────────────────┤
│  .NET 10.0 SDK  ──>  LLVM / Clang 19  ──>  MSVC Build Tools v143       │
│  (Runtime Core)      (Linux Compiler)      (Windows Compiler)          │
└────────────────────────────────────────────────────────────────────────┘
```

*   **Runtime Software Development Kit:** .NET 10.0 SDK (Version 10.0.100 or higher).
*   **Linux Native Compiler Toolchain:** LLVM / Clang 19 with dependent developer tooling libraries (`zlib1g-dev`).
*   **Windows Native Compiler Toolchain:** Microsoft Visual C++ Build Tools (v143 toolset, included in Visual Studio 2022 v17.12+).
*   **Unit Testing Framework:** xUnit v2.9.x with matching target support libraries.
*   **Microbenchmarking Harness Engine:** BenchmarkDotNet v0.14.0.
*   **Performance Diagnostics & Profiling Tooling:**
    *   `dotnet-trace` (Version 10.0.x) for checking GC allocations.
    *   `dotnet-dump` (Version 10.0.x) for auditing target memory structures.
    *   Valgrind (Version 3.23.0+) on Linux targets to monitor unmanaged memory allocations.
    *   GDB (Version 15.1+) for verifying crash dump logs.
*   **Security Validation Fuzzers:** SharpFuzz (Version 2.2.0+) for structural buffer validation checks.

---

## 5. Verification Integration

This project uses a dual-engine testing strategy to catch bugs early in development and verify target compatibility before deployment.

```
                                  [ CODE COMMIT ]
                                         │
                                         ▼
                        ┌─────────────────────────────────┐
                        │      IL Execution Engine        │
                        │  - Fast feedback dynamic runs  │
                        └────────────────┬────────────────┘
                                         │
                                         ▼
                        ┌─────────────────────────────────┐
                        │    Trimming Static Analysis     │
                        │  - Warnings treated as errors   │
                        └────────────────┬────────────────┘
                                         │
                                         ▼
                        ┌─────────────────────────────────┐
                        │    Native AOT Target Build      │
                        │  - Cross-platform compilation   │
                        └────────────────┬────────────────┘
                                         │
                                         ▼
                        ┌─────────────────────────────────┐
                        │    Native Execution Engine      │
                        │  - Memory, SIMD, Fuzz tests     │
                        └────────────────┬────────────────┘
                                         │
                                         ▼
                               [ SIGNED ARTIFACTS ]
```

### A. Dual-Engine Execution Workflow
The automated CI environment runs tests across two verification lanes:

#### 1. IL Execution Engine
*   **Goal:** Provide rapid code feedback during development iterations.
*   **Execution Command:** `dotnet test --configuration Debug`
*   **Operational Scope:** Validates logical execution paths, boundary exceptions, and object-oriented designs.

#### 2. Native AOT Target Engine
*   **Goal:** Verify trimmer compliance and ensure native code behaves exactly like IL code.
*   **Execution Command:** `dotnet publish --configuration Release /p:PublishAot=true`
*   **Operational Scope:** Compiles the unit test projects into separate native executables to verify pointer operations, SIMD behaviors, and memory-mapped layouts.

---

### B. CI Integration & Gated Triggers

The compilation pipeline defines automated check rules on target events:

```yaml
# Loom Build Pipeline Config Snippet
on:
  push:
    branches: [ main, release/* ]
  pull_request:
    branches: [ main ]

jobs:
  validate_pr:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
    steps:
      - uses: actions/checkout@v4
      - name: Setup .NET SDK
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'

      - name: Run IL Fast Verification Test Suites
        run: dotnet test Loom.sln --configuration Debug

      - name: Compile and Verify Trimming Diagnostics
        run: dotnet build Loom.sln --configuration Release /p:TreatWarningsAsErrors=true /p:EnableTrimAnalyzer=true

      - name: Run Native AOT Compilation Packages
        run: dotnet publish Loom.Host/Loom.Host.csproj --configuration Release -r ${{ matrix.os == 'ubuntu-latest' && 'linux-x64' || 'win-x64' }}
```

---

### C. Release Artifact Verification & Signing
When builds on the target release branch complete successfully, the pipeline generates deployable binaries:

1.  **Binary Stripping:**
    *   **Linux:** Runs `strip --strip-debug loom` to remove debugging symbols, reducing the binary size below the 15 MB limit.
    *   **Windows:** Extracts target symbols into an isolated `.pdb` archive file, leaving the native executable clean.
2.  **Cryptographic Checksum Registry:**
    *   Generate secure verification checksums: `sha256sum loom > loom.sha256`.
3.  **Code-Signing Procedures:**
    *   **Windows Deployments:** Sign executables using `signtool.exe` with a trusted Authenticode certificate.
    *   **Linux Deployments:** Sign binaries using GnuPG signatures (`gpg --detach-sign --armor loom`). This guarantees file integrity and prevents tampering before deployment.