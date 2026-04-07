# Intel AI Pod Data Flow Architecture

_An Intel-based equivalent of the NVIDIA Vera Rubin Pod Data Flow with LPX & Groq 3 Integration_

---

## Architecture Overview

```
INTEL AI POD DATA FLOW WITH CXL & GAUDI 3 INTEGRATION
═══════════════════════════════════════════════════════════════════════════════════

┌──────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│  1. External       2. Infrastructure   3. Control Plane   4. Core Compute        │
│  Data Ingress      & Offload           ┌─────────────┐    (Prefill)              │
│                                        │             │    ┌──────────────┐       │
│  INTEL TOFINO 2 ──► INTEL IPU ────────►│  INTEL      │───►│ GAUDI 3 AI   │       │
│  ETHERNET          (Mount Evans)       │  XEON W9    │    │ ACCELERATOR  │       │
│                    · Security          │  (Granite   │    │  HBM2e       │       │
│  INTEL TOFINO 2    · Protocol offload  │   Rapids)   │    ├──────────────┤       │
│  ETHERNET          · NVMe-oF storage   │             │    │ GAUDI 3      │       │
│        ↑           · PCIe offload      │  AI Frmwks  │    ├──────────────┤       │
│   EXTERNAL         Tokens ↓           │  (PyTorch,  │    │ GAUDI 3      │       │
│   DATA             User Prompt ↓      │   OpenVINO) │    ├──────────────┤       │
│                                        └──────┬──────┘    │ GAUDI 3      │       │
│                                               │           └──────┬───────┘       │
│                                        ┌──────▼──────┐          │ KV Cache       │
│                                        │  CXL 3.0    │          │ PREFILL STAGE  │
│                                        │  (Storage/  │          │ (Prompt Proc.) │
│                                        │   Expansion)│          │                │
│                                        │  SSD        │          │                │
│                                        │  Intel E810 │          │                │
│                                        │  NIC        │          │                │
│                                        │  HIGH-BW    │          │                │
│                                        │  STORAGE/IO │          │                │
│                                        └─────────────┘          │                │
│                                                                  │                │
│  6. Interconnect (Intra-POD)           5. Specialized Inference  │                │
│  ┌──────────────────────────┐          (Decode)                  │                │
│  │  GAUDI 3 SCALE-UP FABRIC │◄─────────────────────────────────┘                │
│  │  UALink / 2400 Gb/s      │          ┌──────────────────────┐                  │
│  │                          │  Tokens ►│ GAUDI 3 DECODE       │                  │
│  │  Unified HBM2e Pool      │◄─────────│  (Token Generation)  │                  │
│  │  All-to-All GPU Memory   │  GPU Data│                      │                  │
│  └───────────┬──────────────┘          │  HBM2e / SRAM        │                  │
│              │ GPU Data                │  900 GB/s BW         │                  │
│              ▼                         └──────────────────────┘                  │
│  8. Interconnect (Inter-POD)                                                      │
│  ┌──────────────────────────┐                                                    │
│  │  INTEL OMNI-PATH         │──► IB Traffic ──► AI SUPERCOMPUTER CLUSTERS        │
│  │  ARCHITECTURE (OPA) /    │                                                    │
│  │  Intel E810 Ethernet     │                                                    │
│  └──────────────────────────┘                                                    │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘

Legend:  [ HBM2e ]   [ SRAM ]   [ CXL Storage ]
```

---

## Component Mapping: NVIDIA → Intel

| Stage | NVIDIA Component | Intel Equivalent |
|-------|-----------------|-----------------|
| 1. External Data Ingress | Spectrum-X800 Ethernet | **Intel Tofino 2 / Intel Ethernet 800 Series** |
| 2. Infrastructure & Offload | BlueField-4 DPU | **Intel IPU (Mount Evans / Arrow Creek)** |
| 3. Control Plane — CPU | Vera CPU | **Intel Xeon W9 (Granite Rapids)** |
| 3. Control Plane — Storage/IO | LPX (Liquid-cooled PCIe Expansion) | **Intel CXL 3.0 Expansion / Intel Optane PMem** |
| 3. Storage NIC | ConnectX-8 NIC | **Intel E810 100GbE NIC** |
| 4. Core Compute (Prefill) | Rubin GPU + HBM4 | **Intel Gaudi 3 AI Accelerator + HBM2e** |
| 5. Specialized Inference (Decode) | Groq 3 LPU + SRAM | **Intel Gaudi 3 (Decode) + HBM2e/SRAM** |
| 6. Intra-POD Interconnect | NVLink 6 Switch 200Gb/s PAM4 | **Intel UALink / Gaudi 3 Scale-up 2400 Gb/s** |
| 8. Inter-POD Interconnect | Quantum-X800 InfiniBand | **Intel Omni-Path Architecture (OPA) / Intel E810** |
| AI Frameworks | PyTorch | **PyTorch + Intel OpenVINO / IPEX** |

---

## Data Flow Description

### 1. External Data Ingress
External data enters via **Intel Tofino 2** programmable Ethernet switches (top and bottom), providing line-rate programmable packet processing.

### 2. Infrastructure & Offload
**Intel IPU (Infrastructure Processing Unit)** handles:
- Security and isolation
- Protocol offload (RDMA, NVMe-oF)
- Storage access over fabric
- Passes **User Prompts** and **Tokens** downstream

### 3. Control Plane
- **Intel Xeon W9 (Granite Rapids)** CPU runs AI frameworks (**PyTorch + OpenVINO/IPEX**) and dispatches accelerator instructions.
- **Intel CXL 3.0 Storage Expansion** provides high-bandwidth, low-latency storage/IO via NVMe SSDs and Intel E810 NICs.

### 4. Core Compute — Prefill Stage
Multiple **Intel Gaudi 3 AI Accelerators** with **HBM2e** memory perform:
- Prompt processing
- **KV Cache generation**
- Connected via Intel's high-bandwidth scale-up fabric

### 5. Specialized Inference — Decode Stage
**Intel Gaudi 3** accelerators in decode configuration handle **token generation** using HBM2e bandwidth (~900 GB/s per card) and on-chip SRAM.

### 6. Interconnect — Intra-POD
**Intel UALink / Gaudi 3 Scale-up Fabric** (up to 2400 Gb/s aggregate) unifies all HBM2e into a single memory pool and passes KV Cache + GPU data between accelerators.

### 7. Token Flow
Tokens flow from the Scale-up fabric to Gaudi 3 decode accelerators for autoregressive generation.

### 8. Interconnect — Inter-POD
**Intel Omni-Path Architecture (OPA)** or **Intel E810 Ethernet** carries GPU data and fabric traffic out to **AI Supercomputer Clusters**.

---

## Key Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Prefill / Decode Split** | Gaudi 3 prefill nodes generate KV Cache; Gaudi 3 decode nodes generate tokens |
| **Memory Bandwidth** | HBM2e @ ~900 GB/s per Gaudi 3 card |
| **Scale-up Fabric** | Intel UALink / Gaudi 3 native 2400 Gb/s NIC-less interconnect |
| **Scale-out Fabric** | Intel OPA / E810 for multi-pod clusters |
| **Storage** | CXL 3.0 for coherent memory expansion; NVMe-oF for distributed storage |
| **Software Stack** | PyTorch + Intel Extension for PyTorch (IPEX) + OpenVINO |
