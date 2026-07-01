This Software Development Life Cycle (SDLC) is designed for Project Loom, incorporating the constraints of .NET 10, Native AOT compilation, and low-level system engineering identified in the technical review. Due to the low-level memory mapping, SIMD hardware intrinsics, and custom protocol implementation, a **V-Model / Iterative Hybrid SDLC** is used. This approach couples each developmental phase with a matching, highly rigorous verification tier to prevent compilation or runtime failures after trimming.

---

```
                       [ SDLC LIFECYCLE FLOW ]

  1. Requirements & Threat Model ──────> 10. Operational Telemetry & Backoff
            │                                      ▲
            ▼                                      │
  2. AOT-Aware System Design  ─────────>  9. Cross-Platform AOT Deployment
            │                                      ▲
            ▼                                      │
  3. Incremental Implementation ───────>  8. Native-Compiled Profiling & QA
            │                                      ▲
            ▼                                      │
  4. Static & Trim Analysis ───────────>  7. Dynamic Fuzzing & Hardware Tests
            │                                      ▲
            └────────── 5. Continuous Integration ─┘
```

---

## 1. Requirements & Threat Modeling

This phase establishes the operational boundaries, performance targets, and security threat boundaries of the application before writing code.

### A. Technical Constraints & Target Metrics
*   **Memory Footprint:** Under 20 MB of active RAM consumption in idle state; maximum 100 MB under heavy search load.
*   **Startup Latency:** Zero-to-ready cold start under 15 milliseconds, made possible by native code execution and the avoidance of runtime JIT compilation.
*   **Throughput:** SSH parser must process up to 10 Gbps of streaming input on the network socket layer; SIMD engine must filter $10^7$ telemetry records per second.

### B. Threat Modeling (STRIDE)
*   **Spoofing / Tampering:** SSH host key rotation strategy, strictly using Ed25519 host keys to prevent weak RSA configuration exploits.
*   **Information Disclosure:** Uninitialized memory scrubbing. Since Phase 4 uses pointer arithmetic and `unsafe` buffers, memory regions used to parse SSH traffic or read MMF indexes must be cleared explicitly (using `CryptographicOperations.ZeroMemory` or span filling) before disposal to prevent leaking telemetry data or key material.
*   **Denial of Service (DoS):**
    *   Network socket exhaustion: Implement connection-limiting policies on incoming TCP sockets in `Loom.SSH`.
    *   Malformed packet vulnerability: Define strict size limits for SSH channel frames (maximum packet size capped at 32 KB per RFC 4253 guidelines).

---

## 2. Architecture & Native AOT-Aware Design

Every component must be designed with AOT trim-compatibility at its core.

```
┌────────────────────────────────────────────────────────────────────────┐
│                              Loom.Host                                 │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │ (Bootstraps & manages)
         ┌─────────────────────────┼────────────────────────┐
         ▼                         ▼                        ▼
┌──────────────────┐      ┌──────────────────┐    ┌──────────────────┐
│     Loom.SSH     │      │     Loom.TUI     │    │   Loom.Storage   │
│ (Custom Stream)  ├─────>│  (AnsiConsole)   │    │  (MMF & Ingest)  │
└──────────────────┘      └──────────────────┘    └────────┬─────────┘
         │                         │                       │
         │                         │                       ▼
         │                         │              ┌──────────────────┐
         └─────────────────────────┼─────────────>│    Loom.Core     │
                                   │              │  (SIMD Search)   │
                                   └─────────────>└──────────────────┘
```

### A. Modular Isolation & Interface Design
To facilitate independent AOT compilation and testing, assemblies must communicate using interfaces and strictly typed parameter types, avoiding runtime reflection:
*   `Loom.SSH` communicates with `Loom.TUI` via a custom raw duplex `System.IO.Stream` mapping to the decrypted SSH channel.
*   `Loom.TUI` renders widgets on a custom-instantiated `IAnsiConsole` injected with a specialized text writer redirecting console output to the stream.
*   `Loom.Storage` hands data slices to `Loom.Core` as `ReadOnlySpan<float>` to eliminate intermediate memory allocations.

### B. AOT Constraint Auditing
*   Disable dynamic reflection, runtime code-generation (`System.Reflection.Emit`), and late binding.
*   Annotate any type-conversion helper methods with `[DynamicallyAccessedMembers]` to explicitly guide the compiler’s trimmer on which metadata must be preserved.

---

## 3. Implementation Methodology & Phase-Gate Review

Development progresses through the five phases sequentially. Each phase is subject to a strict gate-review process.

```
┌─────────┐     Passed QA     ┌─────────┐     Passed QA     ┌─────────┐
│ Phase 1 ├──────────────────>│ Phase 2 ├──────────────────>│ Phase 3 ├─...
└─────────┘                   └─────────┘                   └─────────┘
  (SSH)                         (TUI)                         (SIMD)
```

### Gate-Review Requirements
1.  **Phase 1 (SSH) -> Phase 2 (TUI):**
    *   *Gate:* Socket throughput verified to sustain 1,000 parallel test connections under zero-allocation limits.
    *   *Security validation:* Fuzz-tested ASN.1 parser passes 1 million malformed packets without memory leaks or runtime exceptions.
2.  **Phase 2 (TUI) -> Phase 3 (SIMD):**
    *   *Gate:* Terminal resize packets (`window-change` requests) received via SSH are successfully intercepted and processed, triggering zero-allocation UI redraw loops.
3.  **Phase 3 (SIMD) -> Phase 4 (Cache):**
    *   *Gate:* Microbenchmarks verify SIMD implementation triggers appropriate vector registers (AVX2/Neon) on target platforms without scalar fallback warnings.
4.  **Phase 4 (Cache) -> Phase 5 (Ingestor):**
    *   *Gate:* Validation that memory-mapped file offsets are naturally aligned, and `Unsafe.SizeOf<T>()` is used to map records safely.

---

## 4. Verification, Validation, & Automated Testing

Testing system-level code destined for Native AOT requires verifying both IL behavior (during rapid iteration) and native machine code execution (for correctness and performance).

```
                     ┌─────────────────────────┐
                     │    Test Execution       │
                     └────────────┬────────────┘
                                  │
         ┌────────────────────────┴────────────────────────┐
         ▼                                                 ▼
┌──────────────────┐                              ┌──────────────────┐
│   IL Test Runs   │                              │ Native AOT Runs  │
│  (Fast feedback, │                              │ (Final validation│
│   debug-friendly)│                              │  compiler bugs)  │
└──────────────────┘                              └──────────────────┘
```

### A. Dual-Engine Test Runner Architecture
The CI pipeline executes tests in two distinct environments:
1.  **IL Execution:** Run standard JIT tests (`dotnet test`) for rapid feedback loop validation of logic, parsing structures, and API integration.
2.  **AOT Execution:** Compile the test project itself as Native AOT (supported natively in newer .NET SDK versions) and run the native executable. This validates that trimmer rules did not break serialization, pointer casting, or dynamic reflection references.

### B. Phase-Specific Verification Matrix

| Phase | Testing Strategy | Primary Validation Metric | Tooling |
| :--- | :--- | :--- | :--- |
| **Phase 1: SSH** | Network socket stress-testing and boundary fuzzing. | Socket leaks under heavy concurrent connection cycling; memory leak verification. | `dotnet-counters`, `fuzz-lightyear` / native packet injectors. |
| **Phase 2: TUI** | Virtual terminal assertions, testing ANSI sequence generation. | Verify no allocations occur on the rendering loops during mock screen updates. | Custom headless `IAnsiConsole` + `BenchmarkDotNet`. |
| **Phase 3: Core** | Vector register assertions, cross-platform correctness. | Verifying AVX2/Neon intrinsic branches run on appropriate hardware without scalar fallbacks. | `BenchmarkDotNet`, `ILspy` disassembly analysis. |
| **Phase 4: Storage** | Memory leaks, alignment verification, pointer bounds testing. | Null reference prevention on MMF disposal; ARM64 alignment fault prevention. | `Valgrind`, `dotnet-dump` memory analysis. |
| **Phase 5: Ingest** | Streaming parsing verification, garbage collection monitoring. | Total GC allocations under continuous JSON data streams must equal zero. | `dotnet-trace` (GC allocation tracking). |

---

## 5. CI/CD Integration & Build Engineering

To guarantee the project is continuously buildable and compilable under Native AOT constraints, the CI/CD pipeline enforces strict build options across both Linux and Windows environments.

```
┌────────────────────────────────────────────────────────┐
│                        Git Push                        │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│               Trimming Static Analysis                 │
│         - Illink Warnings Treated as Errors            │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌───────────────────────────┴────────────────────────────┐
│                  Platform Build Matrix                 │
└─────────────┬────────────────────────────┬─────────────┘
              │                            │
              ▼                            ▼
   ┌────────────────────┐       ┌────────────────────┐
   │ Ubuntu-latest AOT  │       │ Windows-latest AOT │
   │  - clang / gcc     │       │  - MSVC / link.exe │
   └──────────┬─────────┘       └──────────┬─────────┘
              │                            │
              └─────────────┬──────────────┘
                            │
                            ▼
┌───────────────────────────┴────────────────────────────┐
│                    Artifact Packaging                  │
│             - Native Self-Contained Binaries           │
└────────────────────────────────────────────────────────┘
```

### A. MSBuild Configuration Rules
Every project in the solution must declare these settings to block incompatible compilation targets:

```xml
<PropertyGroup>
  <TargetFramework>net10.0</TargetFramework>
  <ImplicitUsings>enable</ImplicitUsings>
  <Nullable>enable</Nullable>
  <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
  
  <!-- Enforce Native AOT constraints -->
  <PublishAot>true</PublishAot>
  <IsTrimmable>true</IsTrimmable>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  <EnableTrimAnalyzer>true</EnableTrimAnalyzer>
  <EnableSingleFileAnalyzer>true</EnableSingleFileAnalyzer>
  <EnableAotAnalyzer>true</EnableAotAnalyzer>
</PropertyGroup>
```

### B. CI Pipeline Steps
The pipeline executes on every pull request and trunk merge:
1.  **Format Check:** Enforce code style limits (`dotnet format --verify-no-changes`).
2.  **Trim Analysis:** Execute compilation. Any compiler warning related to trimming (e.g., `IL2026`, `IL3050`) halts the build.
3.  **Cross-Compilation Execution:**
    *   *Linux Job:* Runs on `ubuntu-latest`. Installs target development tools (`clang`, `zlib1g-dev`) and builds with target optimizations.
    *   *Windows Job:* Runs on `windows-latest`. Installs Desktop Development with C++ workloads and compiles using the MSVC linker.
4.  **AOT Artifact Generation:** Collects self-contained native binaries, strips debug symbols to a separate package, and registers target build checksums (SHA256).

---

## 6. Operations, Monitoring, & Failure Recovery

Deploying a Native AOT telemetry host requires specialized handling, as standard .NET runtime diagnostics (such as the JIT compiler profiler and dynamic assembly instrumentation) are absent.

### A. Diagnostics & Native Telemetry
*   **EventPipe Integration:** Because Native AOT does not compile the standard full-CLR diagnostic loop, integrate lightweight `EventSource` instrumentation.
*   **Symbol Extraction:** Deploy a decoupled debug symbol package (`.dbg` on Linux, `.pdb` on Windows) to an isolated security zone. In production crash states, core dumps must be analyzed with native debuggers (such as `gdb` or `lldb`) mapped with the system symbols to identify precise stack offsets.

### B. Operational Failure Recovery Policies
*   **Dynamic Load Shedding:** If host CPU metrics cross 85%, the telemetry worker threads will dynamically insert delay periods (`Thread.Sleep` or yield intervals) to scale back resource utilization.
*   **Socket Read-Timeout Safeguards:** To protect the server from Slowloris attacks or unresponsive network clients, configure socket structures with strict inactivity timeouts. If an SSH client opens a socket but fails to transmit packets for 30 seconds, the server terminates the underlying socket forcibly.
*   **Segmentation Fault Isolation:** Because Phase 4 uses pointer operations, a memory misalignment or invalid offset attempt could trigger a native access violation (Segmentation Fault). Wrap native pointer read tasks inside localized worker-process execution, or configure OS-level recovery managers (such as `systemd` on Linux or `Windows Recovery Services`) to instantly restart the lightweight service wrapper.