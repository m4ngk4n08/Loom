# Project Loom

> A lightweight, real-time diagnostic terminal companion for production .NET applications.

Loom attaches directly to running apps and gives developers and SREs instant insight into CPU hotpaths, memory allocations, and thread blockages — delivered through a secure, embedded SSH terminal interface.

---

## At a glance

| | |
|---|---|
| **Runtime** | .NET 10 · Native AOT |
| **Binary size** | < 15 MB (single self-contained executable) |
| **Memory footprint** | < 20 MB background |
| **Access** | SSH on port `2222` — any standard SSH client |
| **Platforms** | Linux · Windows |

---

## How it works

```
[ SSH Client ]
      │  TCP :2222
      ▼
 Embedded SSH Server  ──►  Spectre.Console TUI
      ▲
      │  Core Telemetry Engine
      │
 Diagnostic Host Agent
  ├── SIMD Math Engine (vector search)
  ├── RAG Telemetry Ingestor (JSON → sentences)
  └── Binary Embedding Cache (memory-mapped)
```

Loom runs in a separate network namespace from the target application, querying runtime telemetry via OS diagnostic ports (`/proc` on Linux, EventPipe on Windows). It never opens a shell — only predefined diagnostic command keys are permitted over the SSH channel.

---

## Solution structure

```
Loom.sln
├── Loom.SSH/        ← SSH server · zero-allocation ANSI parser
├── Loom.TUI/        ← Spectre.Console render engine · "Digital Surge" UX
├── Loom.Core/       ← SIMD math engine · telemetry vector search
├── Loom.Storage/    ← Binary embedding cache · RAG ingestor
└── Loom.Host/       ← Entry point · IHostedService bootstrap
```

Each project is independently AOT-compilable and testable. `Loom.Host` is a thin bootstrap that wires them together.

---

## Build phases

The full architectural blueprint and phase-by-phase implementation plan lives in [`loom-build-plan.md`](./loom-build-plan.md). Phases must be completed in order — each is a dependency of the next.

| Phase | Project | Description |
|---|---|---|
| 1 | `Loom.SSH` | Embedded SSH server · zero-allocation ANSI input parser |
| 2 | `Loom.TUI` | Spectre.Console live render loop · "Digital Surge" transitions |
| 3 | `Loom.Core` | SIMD dot product / cosine similarity · 10–20× telemetry search speedup |
| 4 | `Loom.Storage` | Memory-mapped binary embedding cache · millisecond startup |
| 5 | `Loom.Storage` | Utf8JsonReader RAG ingestor · nested JSON → context sentences |

---

## Prerequisites

- [.NET 10 SDK](https://dotnet.microsoft.com/download)
- Any SSH client (`ssh`, PuTTY, Windows OpenSSH)
- Linux or Windows (x64 recommended for AVX2 SIMD acceleration)

---

## Getting started

```bash
# Clone and build
git clone https://github.com/your-org/loom.git
cd loom
dotnet build Loom.sln

# Run (attach to a target process by PID)
dotnet run --project Loom.Host -- --pid <TARGET_PID>

# Connect from any terminal
ssh user@localhost -p 2222
```

For Native AOT builds:

```bash
dotnet publish Loom.Host -r linux-x64 -c Release
```

---

## Security

- Loom runs isolated from the target application — separate network namespace, read-only telemetry access.
- The SSH channel permits only predefined Loom command keys. No interactive shell is exposed.
- Telemetry queries automatically back off when the target application's CPU exceeds **85%**, ensuring Loom never causes service degradation.

---

## Further reading

- [`loom-build-plan.md`](./loom-build-plan.md) — full phase-by-phase architectural plan with key tech and deliverables per phase
