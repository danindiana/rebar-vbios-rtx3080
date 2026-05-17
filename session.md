# RTX 3080 ReBAR / Multi-GPU Audit — Session
**Date:** 2026-05-14T144311  
**Host:** worlock (192.168.1.151)  
**Original goal:** Enable Resizable BAR on RTX 3080 via VBIOS update.  
**Outcome:** VBIOS flash deferred (no official ASUS Linux path). Full multi-GPU topology audit completed; optimization opportunities identified.

---

## System Context

| Component | Detail |
|-----------|--------|
| OS | Xubuntu 22.04.5 LTS (kernel 6.8.12) |
| Motherboard | ASRock X570 Taichi |
| CPU | AMD Ryzen 9 5950X (32 threads) |
| RAM | 128 GB |
| GPU 0 | RTX 5080 — `00000000:0E:00.0` — 16 GB — ReBAR **ENABLED** |
| GPU 1 | RTX 3080 — `00000000:0F:00.0` — 10 GB — ReBAR **DISABLED** |
| Display | Connected to GPU 1 (RTX 3080) |
| Driver | 580.142 / CUDA 13.0 |
| Display Manager | LightDM |

---

## GPU Topology Audit Results

### BAR1 State (confirmed real, kernel-enforced)

| GPU | BAR1 | Status |
|-----|------|--------|
| RTX 5080 | 16384 MiB | ReBAR active ✓ |
| RTX 3080 | 256 MiB | ReBAR inactive ✗ |

The kernel allocated exactly 256 MiB for the 3080's BAR1 at boot. No resize was attempted.
The VBIOS's PCIe ReBAR capability register only advertises `64MB 128MB 256MB` as supported sizes —
10240 MiB is not in the list, so neither the kernel nor the NVIDIA driver can negotiate up to it.

```
lspci -vv -s 0f:00.0 — Physical Resizable BAR:
    BAR 1: current size: 256MB, supported: 64MB 128MB 256MB
```

dmesg BAR1 allocations at boot:
```
0e:00.0  RTX 5080: [mem 0x7800000000-0x7bffffffff 64bit pref]  = 16 GiB ✓
0f:00.0  RTX 3080: [mem 0x7c10000000-0x7c1fffffff 64bit pref]  = 256 MiB ✗
```

### PCIe Link State

| GPU | Slot max | Current | Gen | Note |
|-----|----------|---------|-----|------|
| RTX 5080 | x16 | x4 (idle downscale) | 4 | Primary slot, full bandwidth when active |
| RTX 3080 | x16 max reported | x4 **physical** | 4 | Electrically x4 — confirmed by dmesg warning |

dmesg for 3080:
```
63.012 Gb/s available PCIe bandwidth, limited by 16.0 GT/s PCIe x4 link at 0000:00:03.3
(capable of 252.048 Gb/s with 16.0 GT/s PCIe x16 link)
```

The 3080 is connected through `00:03.3` (ASRock X570 Taichi's secondary PCIe controller), which
is electrically wired at x4. This is a motherboard topology constraint — no slot change will fix it
without a different board.

### P2P (Peer-to-Peer) Capability

```
nvidia-smi topo -p2p r:
    GPU0   GPU1
GPU0   X     NS
GPU1   NS    X
NS = Not Supported
```

No direct GPU-to-GPU transfers. All inter-GPU communication routes through host CPU RAM:
`GPU0 → PCIe → CPU → PCIe → GPU1`. Combined with the 3080's x4 physical slot (~63 Gbps max,
shared with host traffic), this is the primary multi-GPU inference bottleneck.

### VBIOS Versions

| GPU | VBIOS |
|-----|-------|
| RTX 5080 | 98.03.6C.00.6C |
| RTX 3080 | 94.02.42.40.66 (pre-ReBAR) |

RTX 3080 subsystem: `1043:87b0` → **ASUS TUF Gaming RTX 3080 10G**

### Clock State (under qwen3-coder:30b load)

| GPU | SM current | SM max | Mem current | Mem max | Power | P-state |
|-----|-----------|--------|-------------|---------|-------|---------|
| RTX 5080 | 2610 MHz | 3090 MHz | 14801 MHz | 15001 MHz | 39W / 275W cap | P1 |
| RTX 3080 | 1785 MHz | 2100 MHz | 9251 MHz | 9501 MHz | 109W / 275W cap | P2 |

Both have persistence mode enabled. Both are well below power caps — not thermally constrained.

### Current Ollama Config (`/etc/systemd/system/ollama.service.d/override.conf`)

```ini
OLLAMA_HOST=127.0.0.1:11434
CUDA_VISIBLE_DEVICES=0,1
OLLAMA_GPU_OVERHEAD=0
OLLAMA_NUM_PARALLEL=1
OLLAMA_MAX_LOADED_MODELS=2
OLLAMA_KEEP_ALIVE=60m
```

**Not set:** `OLLAMA_MAIN_GPU`, `OLLAMA_FLASH_ATTENTION`, `OLLAMA_KV_CACHE_TYPE`

### Live Model: qwen3-coder:30b (Q4_K_M, 22.1 GB)

| GPU | VRAM used | Share |
|-----|-----------|-------|
| RTX 5080 | ~13.9 GB / 16 GB | ~63% |
| RTX 3080 | ~7.6 GB / 10 GB | ~37% |

---

## Why the VBIOS Flash Was Deferred

ASUS provides VBIOS updates as Windows `.exe` installers only. There is no official ASUS-supported
Linux VBIOS update path for the TUF-RTX3080-10G-GAMING. The community approach (extract `.rom`
with `7z`, flash with `nvflash -6`) is unsupported and the `-6` flag bypasses nvflash's own
board-ID safety check. Given the risk of an unbootable card with no vendor recovery path on Linux,
the flash was deferred until a Windows boot environment is available.

**Safe path when ready:**
- Boot Windows (USB PE or installed) and run `RTX3080_V6.exe` from ASUS support directly.
- That is the only officially supported method.

---

## Optimization Opportunities (no VBIOS change needed)

### 1. Set `OLLAMA_MAIN_GPU=0` (low risk, immediate)
Forces the 5080 to handle embedding and output layers. Default may already be 0 but making it
explicit ensures the fast GPU handles the high-bandwidth head/tail layers.

### 2. Enable Flash Attention — `OLLAMA_FLASH_ATTENTION=1` (low risk)
Reduces KV cache memory footprint. Frees VRAM on the 3080, allowing more model weight layers to
stay on the faster 5080 instead of spilling to the 3080.

### 3. Compress KV cache — `OLLAMA_KV_CACHE_TYPE=q8_0` (low risk)
Halves KV cache VRAM usage at minimal quality cost. Same effect as above: shifts layer balance
toward the 5080. Use `q4_0` for more aggressive reduction.

### 4. Single-GPU mode for models ≤ ~14 GB
For models that fit in the 5080's 16 GB alone, consider running with `CUDA_VISIBLE_DEVICES=0` to
avoid the 3080's x4 + no-P2P overhead entirely. The inter-GPU copy cost on these cards is real.
Switch back to `0,1` only for models that genuinely need both (≥ 16 GB).

### 5. Consider `OLLAMA_SCHED_SPREAD=1` (experimental)
Distributes a single model's layers more evenly across GPUs. May reduce per-GPU latency spikes at
the cost of more PCIe transfers. Worth benchmarking against default.

### Constraints that cannot be changed without hardware swap
| Constraint | Root cause | Workaround |
|------------|-----------|------------|
| 3080 BAR1 = 256 MiB | VBIOS (pre-ReBAR) | VBIOS flash in Windows |
| 3080 PCIe x4 | X570 Taichi secondary controller, electrically x4 | Different motherboard |
| No GPU P2P | No NVLink; different PCIe root complexes | NVLink bridge (not supported on these SKUs) |

---

## ReBAR Speedup — Host CPU Scaling Properties

The remap serialization cost is fixed per operation regardless of core count. What changes
is what fraction of your total CPU budget that fixed cost consumes.

```
CPU overhead fraction ≈ remap_cost / (remap_cost + useful_work_per_core × core_count)
```

As core count drops, the denominator shrinks and the fraction grows. On a 32-core 5950X
that fraction is noise. On a 4-core system it's material. On a 2-core embedded system it
can dominate.

The total speedup from ReBAR has two components that scale differently:

```
S_total = S_pcie_serialization + S_cpu_overhead

S_pcie_serialization  →  roughly constant across core counts
                          (protocol-level, can't be parallelized away)

S_cpu_overhead        →  grows as core count decreases
                          (remap lock, TLB shootdowns, IPI cost as fraction of budget)
```

On worlock, S was almost entirely `S_pcie_serialization`. On a 4-core system running the
same workload, `S_cpu_overhead` would add meaningfully on top of that. The card benefits
more, not less, as the host gets weaker.

### Generalization Caveat

The measured S = 2.59× on the 5950X is likely a **floor** for weaker host CPUs running
the same workload, not a ceiling. Someone running the same 3080 on a budget 6-core Ryzen 5
would probably measure a higher S, because they were losing more to CPU overhead in the
256 MiB regime than worlock was.

```
⊢ Generalization: S is not portable. On weaker host CPUs, S may be
                  higher than measured on worlock, not lower, because
                  CPU-side remap overhead consumes a larger fraction
                  of available compute.
```

---

## Files

| File | Purpose |
|------|---------|
| `REBAR_VBIOS_GUIDE.md` | Full VBIOS flash guide (phases 1–3) — deferred |
| `vbios-preflight.sh` | Pre-flash safety script (TTY only) — deferred |
| `vbios-verify.sh` | Post-flash ReBAR verification — deferred |
| `~/vbios-work/preflight-check.sh` | Prerequisites checker for when flash is revisited |
| `session.md` | This document |

---

## VBIOS Flash — When Revisiting

If a Windows boot environment becomes available:
1. Boot Windows
2. Download `RTX3080_V6.exe` from ASUS support → TUF-RTX3080-10G-GAMING → BIOS & Firmware
3. Run the installer — it handles everything natively
4. Reboot to Linux and run `bash ~/vbios-work/vbios-verify.sh`

Expected result after successful flash:
```
RTX 3080 BAR1: 10240 MiB   (up from 256 MiB)
PCIe ReBAR BAR1 register: supported: 64MB 128MB ... 10240MB
```

---

## Applied Changes

### 2026-05-14 — `OLLAMA_MAIN_GPU=0`

Added to `/etc/systemd/system/ollama.service.d/override.conf`. Forces the RTX 5080
(CUDA device 0) to handle embedding and output layers — the highest-bandwidth operations
in transformer inference. Previously unset; default behaviour was undefined.

Ollama restarted and confirmed active with setting live:
```
OLLAMA_MAIN_GPU=0
```

### 2026-05-14 15:15 — Live workload observation (gemma4:31b-it-q4_K_M)

Spot-checked with `peek.sh` and `nvidia-smi` while inference was running.

| | RTX 5080 (GPU 0) | RTX 3080 (GPU 1) |
|--|--|--|
| Compute util | 31% | 25% |
| VRAM used | 14.6 GB / 16.3 GB | 8.9 GB / 10.2 GB |
| Temperature | 61°C | 60°C |
| Power draw | 100W / 275W cap | 156W / 275W cap |
| SM clock | 2655 MHz | 1785 MHz |

Both GPUs active and contributing. Temps well within range. Neither near power cap.
Runner PID up 1:47, no stuck/pending requests.

Note: 3080 draws more power relative to its compute share (156W vs 100W on the 5080)
— consistent with higher per-operation cost from the x4 PCIe + no-P2P constraint, but
within expected bounds. No anomalies.
