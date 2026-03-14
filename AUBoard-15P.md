# AUBoard-15P — Overview & Technical Reference

## What Is It?

The **AUBoard-15P** is an FPGA development kit by **Avnet**, designed by **TRIA Technologies**, built around the **AMD Artix UltraScale+ XCAU15P** — a pure FPGA chip fabricated on a 16nm process.

> **Important distinction:** The AU15P is a **pure FPGA** — it has **no hard ARM processor cores** integrated on-chip. This sets it apart from hybrid SoC-FPGAs like the Zynq UltraScale+ (which pairs FPGA fabric with quad-core Cortex-A53). All processing, including soft CPU cores and any OS, must be instantiated inside the FPGA fabric itself.

---

## Key Specifications

| Component | Details |
|---|---|
| **FPGA** | AMD Artix UltraScale+ XCAU15P |
| **Process node** | 16nm FinFET |
| **Logic cells** | 170K programmable logic cells |
| **DSP slices** | 576 |
| **On-chip BRAM** | 7.6 Mb |
| **GTH transceivers** | 12 × GTH at up to 16 Gbps |
| **DDR4 RAM** | 2 GB (ISSI) |
| **Flash** | 512 Mb QSPI (ISSI) — for boot/config |
| **Power supply** | 60W external PSU |
| **Power devices** | Renesas + TDK |

### Interfaces & Connectivity

| Interface | Details |
|---|---|
| **10GbE SFP+** | 1 port (via GTH transceiver) |
| **HDMI 2.0** | Rx + Tx (3 GTH transceivers) |
| **PCIe Gen4** | x4 endpoint (4 GTH transceivers) |
| **FMC LPC** | 4 GTH transceivers, 80 FPGA I/Os via Samtec connectors |
| **10/100 Ethernet** | Microchip PHY |
| **JTAG / UART** | microUSB (on-board debugger) |
| **Expansion** | 1× Mikroe Click Board site |
| **User I/O** | Slide switches, push buttons, red + RGB LEDs |
| **Sensor** | STMicroelectronics temperature sensor |

### Software & Tools

| Tool | Purpose |
|---|---|
| **AMD Vivado** | FPGA synthesis, implementation, board definition file (BDF) included |
| **PetaLinux BSP** | Pre-built BSP for Linux on MicroBlaze soft core |
| **Bare metal** | Supported |
| **RTOS** | Supported |
| **Linux** | Supported via MicroBlaze soft processor |

---

## Can I Run Linux on It?

**Yes — but with important caveats.**

Since the AU15P has no hard ARM cores, Linux does **not** run natively the way it does on a Raspberry Pi or RK3588 board. Instead:

1. You instantiate a **MicroBlaze** soft processor core inside the FPGA fabric using Vivado.
2. You boot a Linux kernel built with **PetaLinux** (AMD's embedded Linux framework), targeting MicroBlaze.
3. Avnet/TRIA provide a **PetaLinux BSP** to speed up this process.

**What this means in practice:**
- Linux works, but MicroBlaze is a **soft 32-bit RISC core** — significantly slower than a hardware Cortex-A53 or Cortex-A72.
- It is primarily useful for control-plane tasks, not for running general-purpose Linux workloads.
- The real value of the board is running **custom hardware accelerators** in the FPGA fabric alongside the soft CPU.
- Setup requires Vivado + PetaLinux toolchain knowledge; the barrier is higher than booting Linux on an ARM SBC.

---

## Main Advantages Over ARM Boards (e.g., RK3588, iMX8)

ARM SoC boards are general-purpose processors. The AUBoard-15P offers fundamentally different capabilities:

### 1. Massively Parallel Hardware Execution
FPGA logic executes all 170K logic cells **simultaneously**. Unlike a CPU that executes instructions sequentially, the FPGA implements custom data pipelines in hardware — no OS scheduling overhead, no context switching.

### 2. Custom Hardware Accelerators
You can implement custom IP cores for video processing, signal processing, network packet inspection, ML inference, encryption, or any domain-specific algorithm — achieving throughput that a CPU cannot match at similar power.

### 3. Ultra-Low, Deterministic Latency
Every processing stage in the FPGA is a physical pipeline. Input-to-output delay is **fixed and cycle-accurate**, unaffected by OS load or interrupt jitter. Critical for real-time and industrial control applications.

### 4. High-Speed Multi-Protocol Transceivers
12 GTH transceivers at **16 Gbps** each support PCIe Gen4, 10GbE SFP+, HDMI 2.0, and FMC expansion simultaneously. ARM SoCs typically offer far fewer high-speed serial lanes at lower speeds.

### 5. PCIe Gen4 x4 Endpoint
Offers high-bandwidth host connectivity, enabling the board to act as an accelerator card in a PC/server — uncommon in ARM SBC platforms.

### 6. Custom I/O Protocol Implementation
Any interface standard (MIPI CSI, LVDS, custom industrial protocol, etc.) can be implemented in the FPGA fabric without waiting for silicon vendors to add it.

### 7. Reprogrammability
The FPGA bitstream can be updated to implement entirely different hardware logic, allowing field upgrades without hardware changes.

### 8. DSP Performance
576 DSP48E2 slices provide hard-wired multiply-accumulate blocks ideal for signal processing, filtering, FFTs, and neural network inference at rates that would require a dedicated NPU in an ARM SoC.

---

## Closest Comparable Boards

### Within the AMD/Xilinx Ecosystem (FPGA + ARM — Hybrid)

The most natural step-up is a **Zynq UltraScale+ MPSoC** board, which adds hard ARM cores alongside FPGA fabric — giving you both native Linux and FPGA acceleration:

| Board | FPGA Device | ARM Cores | Logic Cells | Key Differentiator |
|---|---|---|---|---|
| **Avnet Ultra96-V2** | XCZU3EG | 4× Cortex-A53 + 2× Cortex-R5 | 154K | Compact, consumer-friendly |
| **AMD ZCU104** | XCZU7EV | 4× Cortex-A53 + 2× Cortex-R5 | 504K | H.264/H.265 video codec, similar price tier |
| **AMD ZCU102** | XCZU9EG | 4× Cortex-A53 + 2× Cortex-R5 | 599K | More GTH lanes, larger form factor |

> The **ZCU104** is arguably the closest spiritual equivalent — similar price range to the AUBoard-15P, similar FPGA capability tier, but with hard ARM cores for running native Linux.

### From the RockChip Family (Pure ARM SoC)

These boards excel at running Linux workloads natively but have no FPGA fabric:

| Board | SoC | CPU | Key Feature |
|---|---|---|---|
| **Rock 5B / Orange Pi 5 Plus** | RK3588 | 4× Cortex-A76 + 4× Cortex-A53 | 6 TOPS NPU, PCIe 3.0, USB 3.0 |
| **ROCK 3A / Rock Pi 4** | RK3399 | 2× Cortex-A72 + 4× Cortex-A53 | Mature ecosystem, good Linux support |

> The **RK3588** is the most powerful RockChip SoC available. It dominates in general-purpose Linux performance and has an integrated NPU, but cannot perform FPGA-style parallel logic or custom hardware acceleration.

### From the NXP i.MX Family (Pure ARM SoC)

| Board | SoC | CPU | Key Feature |
|---|---|---|---|
| **Toradex Apalis iMX8QM** | i.MX 8QuadMax | 2× Cortex-A72 + 4× Cortex-A53 + 2× Cortex-M4F | Heterogeneous compute, industrial grade |
| **Variscite DART-MX8M-PLUS** | i.MX 8M Plus | 4× Cortex-A53 + M7 + NPU | 2.3 TOPS NPU, strong multimedia |

> The **i.MX 8QuadMax** is NXP's most powerful heterogeneous SoC, with real-time M4F cores alongside application cores — but still no FPGA fabric.

### Summary: Which Is "Closest"?

| Goal | Recommended Board |
|---|---|
| FPGA + native ARM Linux (closest equivalent) | **AMD ZCU104** (Zynq UltraScale+ MPSoC) |
| Powerful ARM Linux SBC (RockChip) | **RK3588-based board** (Rock 5B, Orange Pi 5 Plus) |
| Industrial ARM SoC (NXP) | **i.MX 8QuadMax MEK** |
| Pure FPGA, same chip, different form factor | **ALINX AXAU15** or **Opal Kelly XEM8305** |

---

## References

- [AUBoard-15P Product Page — Avnet Americas](https://www.avnet.com/americas/products/avnet-boards/avnet-board-families/auboard-15p-fpga-development-kit/)
- [AUBoard-15P Getting Started Guide (PDF) — Avnet](https://www.avnet.com/wcm/connect/a2ac8ebf-17af-4d07-993c-8a865f0c410a/AUB-15P-DK-GSG-V1P3.pdf?MOD=AJPERES&CACHEID=ROOTWORKSPACE-a2ac8ebf-17af-4d07-993c-8a865f0c410a-ptrsEVS)
- [Avnet Press Release — AUBoard 15P](https://news.avnet.com/press-releases/press-release-details/2024/Avnet-Releases-AUBoard-15P-Development-Kit/default.aspx)
- [MicroZed Chronicles — TRIA AUBoard 15P Review](https://www.adiuvoengineering.com/post/microzed-chronicles-tria-auboard-15p-fpga-development-kit)
- [AMD Artix UltraScale+ FPGAs — Product Page](https://www.amd.com/en/products/adaptive-socs-and-fpgas/fpga/artix-ultrascale-plus.html)
- [TRIA Technologies — AUBoard-15P](https://www.tria-technologies.com/product/auboard-15p/)
- [Hackster.io — Avnet Releases AUBoard 15P](https://www.hackster.io/news/avnet-releases-auboard-15p-development-kit-6da7cd94de6d)
- [Electronic Specifier — Avnet AUBoard 15P](https://www.electronicspecifier.com/products/fpgas/avnet-releases-auboard-15p-development-kit)
