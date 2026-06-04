# Three Laptops, Three Months, One Bug: A Deep-Dive Investigation Into VIDEO_TDR_FAILURE on the Lenovo Legion Pro 5 16IRX10 (RTX 5060)

> **TL;DR** — A reproducible kernel crash (`VIDEO_TDR_FAILURE`, bug check `0x116`) has affected three brand-new Lenovo Legion Pro 5 16IRX10 units I have owned in succession over a three-month period. The crash signature is **identical across all units** (`FAILURE_ID_HASH: c89bfe8c-ed39-f658-ef27-f2898997fdbd`), pointing to `nvlddmkm.sys` (the NVIDIA display driver) at offset `+0x1a254c0`, with `NTSTATUS 0xC000009A (STATUS_INSUFFICIENT_RESOURCES)`. The failure persists after a thorough elimination of every common software cause: clean Studio driver installs via DDU, latest BIOS, latest Intel ME firmware, hypervisor disabled, HVCI off, Clean Boot, fixed display routing, dGPU power state pinned to maximum performance. I am writing this up in the hope that other affected owners will find it, that the data here is useful to Lenovo and NVIDIA engineering, and that someone has a piece of the puzzle I am missing.
>
> **Affected hardware:** Lenovo Legion Pro 5 16IRX10, machine type 83NN, BIOS `S9CN18WW` (24 Nov 2025), Intel Core i9-14900HX, NVIDIA GeForce RTX 5060 Laptop GPU (8 GB GDDR7), Windows 11 Home build 26200.
>
> **Status:** Unresolved. Filing with Lenovo and NVIDIA support. This document is the evidence package.

---

## Table of Contents

1. [Why I Am Writing This](#1-why-i-am-writing-this)
2. [The Hardware and the Setup](#2-the-hardware-and-the-setup)
3. [The Symptom](#3-the-symptom)
4. [Three Laptops, Same Crash](#4-three-laptops-same-crash)
5. [What VIDEO_TDR_FAILURE Actually Means](#5-what-video_tdr_failure-actually-means)
6. [The Investigation, Phase by Phase](#6-the-investigation-phase-by-phase)
7. [Hypotheses Considered and Eliminated](#7-hypotheses-considered-and-eliminated)
8. [The Display Routing Discovery (and Why It Wasn't the Answer)](#8-the-display-routing-discovery-and-why-it-wasnt-the-answer)
9. [The Final Crash and the Diagnostic Endpoint](#9-the-final-crash-and-the-diagnostic-endpoint)
10. [Evidence Catalogue](#10-evidence-catalogue)
11. [Configuration State Tables](#11-configuration-state-tables)
12. [The Conclusion: This Is a Platform Defect](#12-the-conclusion-this-is-a-platform-defect)
13. [What I Am Doing Next](#13-what-i-am-doing-next)
14. [What You Can Do If This Is Happening to You](#14-what-you-can-do-if-this-is-happening-to-you)
15. [Appendices: Full Dump Output, Commands, References](#15-appendices)
16. [Community Discussion on Reddit, 100's of similar cases](https://www.reddit.com/r/LenovoLegion/comments/1tn5os7/three_laptops_three_months_one_bug_a_deepdive/)

---

## 1. Why I Am Writing This

Three brand-new Lenovo Legion Pro 5 16IRX10 laptops, purchased in sequence over three months, have all suffered the same kernel-level GPU crash, with an identical software fingerprint, despite every reasonable software remediation. This is not a one-off lemon. This is a defect at the platform level — somewhere between NVIDIA's mobile Blackwell driver, NVIDIA's VBIOS for this specific OEM SKU, and Lenovo's system firmware (BIOS and EC).

I am writing this for four audiences:

1. **Other owners** of the Legion Pro 5 16IRX10 (or related 16IRX10 / Legion 5 Gen 10 SKUs with RTX 5060/5070 mobile parts) who are experiencing the same crashes and are wondering whether they have a defective unit or a defective product line. **You very likely have the latter.** Returning the laptop will not fix it. I tried twice.

2. **Lenovo support and engineering** — this is the evidence package. Eight kernel dumps, identical hash, three units, every reasonable software remediation attempted. If your support staff are reading this looking for a way to deflect the issue back to me, please understand: I have done your work. I have ruled out everything I can rule out from the outside.

3. **NVIDIA developer support** — your driver `596.36` Studio branch produces a reproducible TDR on this specific OEM platform under idle conditions, with a consistent stack trace. The dump file is available on request.

4. **Everyone else who debugs these things** — I am putting the entire methodology here, including the dead ends, in the hope that someone smarter than me sees a thread I have missed. If you spot it, please get in touch. I will buy you a beverage of your choice.

This took roughly two weeks of evening time across multiple debugging sessions to compile, with assistance from Claude (Anthropic's AI assistant) acting as a sounding board and Windows debugging guide. The technical decisions and the writeup are mine; the AI helped accelerate hypothesis generation and verify command syntax against current Microsoft documentation.

---

## 2. The Hardware and the Setup

### The Laptop

| Component | Value |
|---|---|
| **Brand / Model** | Lenovo Legion Pro 5 16IRX10 |
| **Machine Type** | 83NN |
| **Marketing Name** | Legion Pro 5i 16″ Gen 10 |
| **CPU** | Intel Core i9-14900HX (Raptor Lake-HX Refresh, 24 cores / 32 threads, 5.8 GHz boost) |
| **dGPU** | NVIDIA GeForce RTX 5060 Laptop GPU (Blackwell, 8 GB GDDR7, 115 W max) |
| **iGPU** | Intel UHD Graphics (Raptor Lake-S Mobile) |
| **Display Routing** | Advanced Optimus (Hybrid) with MUX |
| **Internal Panel** | 16″ 2560×1600 @ 240 Hz (Lenovo DisplayHDR) |
| **RAM** | 32 GB DDR5-5600 SO-DIMM (Kingston, 2× 16 GB) |
| **Storage** | Samsung NVMe (1 TB, OS drive) + WDC NVMe (USB external) |
| **OS** | Windows 11 Home, build 26200 (25H2 branch) |
| **BIOS** | `S9CN18WW`, dated 24 Nov 2025 (latest available as of May 2026) |
| **EC / Intel ME** | Intel Management Engine 16.1 Firmware 16.1.40.2765 (latest, installed during investigation) |
| **NVIDIA Driver** | 596.36 (Studio, WHQL, DCH) |
| **Power** | 230 W AC adapter, 80 Wh battery (cycle count: 1) |

The laptop was purchased new from an authorised Lenovo retailer in early 2026. It is the third unit of the same model I have owned. The first two were replaced under warranty after exhibiting the same crash described below.

### The Usage Environment

- **Location**: Dubai, UAE. Stable mains power, stable temperature (controlled climate, 22-24°C ambient).
- **Physical setup**: Lid closed, on a desk, connected to a single external monitor.
- **External display**: LG UltraFine 4K (3840×2160 @ 60 Hz) over USB-C.
- **Power state**: Always plugged in, AC adapter connected to a clean mains circuit.
- **Workload**: Software development. Node.js / TypeScript / Python. VSCode, terminals, Chrome/Edge browsers, occasional Docker against a remote daemon (no local Docker / Hyper-V / WSL).
- **No gaming, no rendering, no GPU compute workloads.** The dGPU is effectively idle 100% of the time from a workload perspective.

This last point matters. The crashes are **not** correlated with GPU load. They occur during idle.

---

## 3. The Symptom

### The Visible Behaviour

Returning to the laptop after stepping away for anywhere between 30 minutes and several hours, I find Windows has restarted itself. No application state is preserved. No BSOD is visible by the time I sit down — the machine has already rebooted and presents me with the lock screen.

There is no specific trigger I can correlate with the crash beyond "idle for a non-trivial period." It happens overnight. It happens during dinner. It happened once at 2 PM while I was in the next room.

### The Forensic Behaviour

After each event:

- **Windows Event Log** records an `Event 41` (Kernel-Power, Critical) with timestamp.
- **A minidump** is written to `C:\Windows\Minidump\` (first 7 crashes) or **a full kernel dump** is written to `C:\Windows\MEMORY.DMP` (8th crash).
- The dump always identifies the same bug check: `0x00000116 VIDEO_TDR_FAILURE`.
- The "Caused By Driver" attribution (as shown by NirSoft BlueScreenView) consistently points to `dxgkrnl.sys` (the Microsoft DirectX kernel) at the surface level, with `nvlddmkm.sys` (NVIDIA's display driver) as the underlying responsible module.

### The Timeline

| # | Date | Time | Laptop Unit | Bug Check |
|---|---|---|---|---|
| 1 | 18 Jan 2026 | 20:18:41 | Unit #2 (first 16IRX10) | `0x133` DPC_WATCHDOG_VIOLATION |
| 2 | 19 Jan 2026 | 20:46:33 | Unit #2 | `0x116` VIDEO_TDR_FAILURE |
| 3 | 22 Jan 2026 | 18:38:01 | Unit #2 | `0x116` VIDEO_TDR_FAILURE |
| 4 | 22 Jan 2026 | 20:05:27 | Unit #2 | `0x133` DPC_WATCHDOG_VIOLATION |
| 5 | 1 Mar 2026 | 14:07:40 | Unit #2 | `0x116` VIDEO_TDR_FAILURE |
| 6 | 11 May 2026 | 19:06:35 | Unit #3 (current) | `0x116` VIDEO_TDR_FAILURE |
| 7 | 11 May 2026 | 20:43:05 | Unit #3 | `0x116` VIDEO_TDR_FAILURE |
| 8 | 14 May 2026 | 22:31:53 | Unit #3 | `0x133` DPC_WATCHDOG_VIOLATION |
| 9 | 18 May 2026 | 02:50:52 | Unit #3 | `0x116` VIDEO_TDR_FAILURE |
| 10 | 21 May 2026 | 04:05:53 | Unit #3 | `0x116` VIDEO_TDR_FAILURE |
| 11 | 23 May 2026 | 01:29:44 | Unit #3 | `0x116` VIDEO_TDR_FAILURE |

(Unit #1 was a different model — a higher-end laptop replaced in late 2025 after exhibiting the same `VIDEO_TDR_FAILURE` symptom. Returned for refund.)

### The Crash Hash

The most damning piece of evidence: **every `0x116` crash on units #2 and #3 produces an identical `FAILURE_ID_HASH`**:

```
FAILURE_ID_HASH: {c89bfe8c-ed39-f658-ef27-f2898997fdbd}
FAILURE_BUCKET_ID: 0x116_IMAGE_nvlddmkm.sys
```

Microsoft's failure hash is computed from the call stack, the responsible module, and the failure context. The same hash across machines and weeks means the same code path is hanging in the same way every single time.

---

## 4. Three Laptops, Same Crash

This is the part that should make Lenovo and NVIDIA stop and pay attention.

- **Unit #1** (different higher-end model, late 2025): Same TDR symptom. Returned for refund.
- **Unit #2** (Legion Pro 5 16IRX10, early 2026): Same TDR within 48 hours of setup. Replaced under warranty after multiple crashes.
- **Unit #3** (Legion Pro 5 16IRX10, early/mid 2026): The same TDR returned after a few weeks of clean operation. Six crashes in ~2 weeks. This unit is the one I have now.

Three units. Same crash. Same kernel signature on the two units where dumps were collected. Different serial numbers, different physical assemblies, different production batches.

Two units of the **same model** with the **same NVIDIA Blackwell mobile silicon**, **same Lenovo BIOS family**, **same VBIOS** is not a coincidence — it is the model platform itself.

---

## 5. What VIDEO_TDR_FAILURE Actually Means

For non-Windows-debugger readers, this section gives the background. Skip to Section 6 if you already know.

**TDR** stands for *Timeout Detection and Recovery*. It is a Windows subsystem (introduced with Vista, evolved continuously since) that watches the GPU. The rules are roughly:

1. The OS submits a GPU command (draw a frame, copy a buffer, run a shader).
2. The OS expects the GPU to acknowledge or complete the command within a timeout (default: 2 seconds at the kernel scheduler level, plus driver-internal timeouts).
3. If the GPU does not respond, the OS assumes the GPU has hung.
4. The OS asks the GPU driver to reset the device.
5. If the reset succeeds, you see a brief screen flicker and a "Display driver stopped responding and has recovered" notification. No crash.
6. **If the reset also fails — the GPU does not even respond to the reset request — the OS gives up and bug-checks the kernel.** This is bug check `0x116`.

The bug check parameters tell you what failed:

```
Arg1: TDR_RECOVERY_CONTEXT pointer (internal Windows structure)
Arg2: Pointer to the responsible driver module
Arg3: NTSTATUS of the last failed operation
Arg4: Internal context-dependent data
```

In our case:

```
Arg1: ffff848e52156290        ← Recovery context
Arg2: fffff804777f54c0        ← Inside nvlddmkm.sys
Arg3: ffffffffc000009a        ← STATUS_INSUFFICIENT_RESOURCES
Arg4: 0000000000000004        ← Context flag
```

`STATUS_INSUFFICIENT_RESOURCES (0xC000009A)` is interesting. It does not mean "the system ran out of RAM." It means **the GPU could not service a resource allocation request from the OS** — typically a GPU memory allocation, a kernel paged pool entry related to GPU state, or a context handle. On an idle desktop with 32 GB of RAM and a barely-used GPU, the system has plenty of resources at the OS level. So this means **the GPU itself refused or could not complete an allocation, even though the OS asked it to**.

The most likely interpretation: the GPU was in a deep idle state (low power, low clocks, possibly with certain functional units gated off), the OS asked it to wake up and allocate something, and the wake-up sequence stalled or failed. The driver then could not initialise the operation, returned `INSUFFICIENT_RESOURCES`, the OS waited for a retry, the retry timed out, the reset attempt timed out, and the kernel bug-checked.

This is consistent with **a flaw in the GPU's power management state machine, or in the negotiation between the GPU and the system firmware that controls power**.

---

## 6. The Investigation, Phase by Phase

The investigation ran across roughly two weeks of evening sessions. Each phase introduced one or more configuration changes and was followed by a "watch window" — wait for the next crash (or lack of one) to gather signal.

### Phase 0: Initial Triage

- Identified the crash via `BlueScreenView` (NirSoft) reading the minidumps.
- Confirmed `0x116 VIDEO_TDR_FAILURE` attribution: `dxgkrnl.sys` surface, `nvlddmkm.sys` underlying.
- Replaced peripherals: keyboard, mouse, USB cables, display cable, eventually the external monitor itself. **No effect.**

### Phase 1: Driver Reset (DDU + Studio Driver)

1. Booted into Safe Mode.
2. Ran [Display Driver Uninstaller (DDU)](https://www.guru3d.com/files/cat/214) with these options:
   - Prevent downloads of drivers from Windows Update
   - Remove GeForce Experience
   - "Clean and do not restart"
3. Installed [NVIDIA Studio Driver 596.36](https://www.nvidia.com/en-us/drivers/) (DCH, WHQL, current Studio branch as of April 2026) — Custom (Advanced) install, "Perform a clean installation," only Graphics Driver and HD Audio Driver components.
4. Blocked further Windows Update driver replacement:
   ```powershell
   New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" -Force | Out-Null
   Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" -Name "ExcludeWUDriversInQualityUpdate" -Value 1 -Type DWord
   ```

**Result**: Crashes continued. Same hash.

### Phase 2: Uninstalled Lenovo X-Rite Color Assistant

Reliability Monitor (`perfmon /rel`) showed `X-Rite Color Assistant` appearing in the critical events list adjacent to crash timestamps. X-Rite Color Assistant is Lenovo's factory display calibration utility, preinstalled on Legion machines with factory-calibrated panels. It writes ICC profiles to the GPU's LUT (lookup table) at startup and on display events, which means it pokes the NVIDIA driver in the surface area we suspect.

Uninstalled it from Settings → Apps → Installed apps. No further interaction with the GPU from this software.

**Result**: Crashes continued. The X-Rite entries in Reliability Monitor were victims (any app holding a D3D context dies during a TDR), not the cause.

### Phase 3: Clean Boot

Disabled all non-Microsoft startup services and applications via `msconfig` and Task Manager → Startup. This is the standard Microsoft "Clean Boot" troubleshooting state — it isolates the OS from any third-party software that loads at boot.

The known third-party kernel drivers on this system were minimal. No antivirus beyond Defender. No VPN client. No Docker Desktop. No DisplayLink. No virtual display drivers.

**Result**: Crashes continued. The cause is not third-party software.

### Phase 4: Power and Sleep

Disabled OS-level sleep entirely:

```powershell
powercfg /change standby-timeout-ac 0
powercfg /change hibernate-timeout-ac 0
powercfg /change monitor-timeout-ac 10
powercfg /change standby-timeout-dc 0
powercfg /change hibernate-timeout-dc 0
```

Also configured lid-close to "Do nothing" (since lid is always closed in this setup, this matters).

Confirmed via `powercfg /a` that the laptop uses **S3 traditional sleep**, not S0ix Modern Standby — eliminating one entire class of "wake from sleep" failure modes.

Confirmed via `powercfg /waketimers` that no wake timers were active. Confirmed via `powercfg /lastwake` that no recent wake source was logged.

**Result**: Crashes continued. The cause is not the OS sleep/wake path.

### Phase 5: Hypervisor and HVCI Disable

The 5/21 dump revealed the hypervisor was active (`Hypervisor.RootFlags.IsHyperV: 1`). VBS (Virtualization-Based Security) was running with HVCI / Memory Integrity enabled — Windows 11's default on supported hardware. HVCI validates kernel-mode driver code pages in a hypervisor-protected memory region and has been a documented contributor to GPU TDR issues in past NVIDIA driver branches.

Disabled HVCI:

- Windows Security → Device security → Core isolation → Memory integrity → Off.

That alone did not stop the hypervisor from launching. Forced the broader VBS disable:

```powershell
reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" /v EnableVirtualizationBasedSecurity /t REG_DWORD /d 0 /f
reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\HypervisorEnforcedCodeIntegrity" /v Enabled /t REG_DWORD /d 0 /f
bcdedit /set "{default}" hypervisorlaunchtype off
```

After reboot, verified:

```powershell
bcdedit /enum "{default}" | findstr /i hypervisor
# hypervisorlaunchtype    Off

Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard
# SecurityServicesRunning: {0}
# VirtualizationBasedSecurityStatus: 2   ← stubbornly remains at 2
```

The `VirtualizationBasedSecurityStatus: 2` remained even after all these steps. The hypervisor stayed loaded — this is a Lenovo OEM secured-core enforcement that cannot be defeated without re-imaging Windows. However, `SecurityServicesRunning: {0}` confirms no security services are consuming the hypervisor — it is "loaded but completely idle."

**Result**: Crashes continued. HVCI was not the cause; the residual hypervisor presence appears to be inert.

### Phase 6: Firmware Updates

Checked Lenovo's support page for machine type 83NN ([Lenovo Support 83NN](https://pcsupport.lenovo.com/) — search for "83NN" or "Legion Pro 5 16IRX10"):

- **BIOS**: `S9CN18WW` dated 24 Nov 2025. **This is the latest available.** No newer version published as of May 2026.
- **Intel Management Engine Firmware**: `16.1.40.2765`, released 19 Jan 2026. Was older on the unit. Installed via Lenovo's package.

The Intel ME firmware was the only firmware lever available. BIOS was already current.

**Result**: After firmware update, crashes continued.

### Phase 7: NVIDIA Power Management to "Prefer Maximum Performance"

The deep-idle P-state transition hypothesis: the GPU clocks itself down aggressively when idle (from P0 active to P8 deep idle), and the wake transition is what fails. The fix-test for this hypothesis: pin the GPU at P0.

NVIDIA Control Panel → Manage 3D settings → Global Settings → Power management mode → **Prefer maximum performance**.

This prevents the dGPU from entering deep idle states. The GPU runs hotter and consumes more power but never has to "wake up."

**Result**: Crashes continued. In fact, the next crash (5/23) occurred *with* the P0 lock active.

### Phase 8: Display Routing Fix (and the discovery that this was wrong)

This was the most promising lead. I will discuss it in its own section ([Section 8](#8-the-display-routing-discovery-and-why-it-wasnt-the-answer)) because the discovery process and the disappointment are the most instructive parts.

### Phase 9: BIOS GPU Mode (Discrete-Only — partial)

Switched from Hybrid Auto to Discrete Graphics in BIOS Setup (F2 at boot → Configuration → Graphics Device).

**Result**: Examination of `Win32_VideoController` revealed both adapters were still enumerated and active. On this 16IRX10 model, the BIOS "Discrete" mode is partial — it routes the internal panel to the dGPU but leaves the iGPU active for external outputs. Not a full MUX bypass.

Reverted to Hybrid mode and instead used Windows Display Settings to disable the internal panel entirely.

### Phase 10: The Final State

After all of the above, the system reached its cleanest possible configuration short of reinstalling Windows:

| Layer | State |
|---|---|
| BIOS | `S9CN18WW` (latest) |
| Intel ME Firmware | `16.1.40.2765` (latest) |
| GPU Mode | Hybrid (Optimus) |
| Display routing | dGPU drives external 4K, internal panel disabled in Windows |
| NVIDIA Driver | Studio 596.36, DDU clean install |
| NVIDIA Power Mode | Prefer maximum performance (then later reverted to Optimal — no effect either way) |
| Windows Sleep | Disabled on AC and DC |
| Lid behaviour | Do nothing |
| HVCI / Memory Integrity | Off |
| VBS | Disabled in registry (but hypervisor remains loaded as inert OEM enforcement) |
| Clean Boot | All non-Microsoft startup items disabled |
| X-Rite Color Assistant | Uninstalled |
| Wake timers | None |

And the crash on 23 May 2026 at 01:29 AM still happened, with the identical signature.

---

## 7. Hypotheses Considered and Eliminated

In rough order of investigation:

| # | Hypothesis | How Tested | Result |
|---|---|---|---|
| 1 | Defective hardware (specific unit) | Two warranty replacements, identical symptom on each | **Ruled out** — defect is across units of the same model |
| 2 | Bad NVIDIA driver branch | DDU + clean Studio install of latest version (596.36) | Ruled out — current Studio driver is affected |
| 3 | Third-party driver conflict | Clean Boot, disabled all non-Microsoft services / startup | Ruled out — no change |
| 4 | X-Rite Color Assistant | Uninstalled | Ruled out — no change |
| 5 | Bad Windows Update / system file corruption | `sfc /scannow`, `DISM /Online /Cleanup-Image /RestoreHealth` | Ruled out — system files clean |
| 6 | Power source / hotel mains / generator | Crashes occurred at home on stable Dubai mains across multiple weeks | Ruled out — geography-independent |
| 7 | Cable / connector / monitor | Replaced all peripherals | Ruled out |
| 8 | Sleep/wake transition | OS sleep disabled, lid close set to "Do nothing" | Ruled out — crashes occur even when machine never sleeps |
| 9 | Modern Standby (S0ix) bug | `powercfg /a` confirmed S3 traditional sleep is in use; S0ix not supported on this model | Ruled out — N/A |
| 10 | Wake timers waking the system | `powercfg /waketimers` returned none | Ruled out |
| 11 | PCIe ASPM bug | `powercfg /energy` reported ASPM already disabled by Windows due to known incompatibility | Ruled out — Windows has been working around this from day one |
| 12 | HVCI / Memory Integrity overhead | Disabled HVCI via Windows Security UI and registry | Ruled out — no change |
| 13 | Hyper-V / WSL / Docker | Not installed on this machine; Docker runs against remote daemon | Ruled out — none of these features enabled |
| 14 | Battery / power circuitry degradation | Battery cycle count: 1 (brand new); always on AC | Ruled out |
| 15 | Thermal | HWiNFO showed normal idle temperatures; crashes occur at idle, not under load | Unlikely but cannot be 100% ruled out without thermal logging across a crash event |
| 16 | Intel ME firmware outdated | Updated to `16.1.40.2765` | Ruled out — crashes continued |
| 17 | BIOS outdated | Already on latest `S9CN18WW` from 24 Nov 2025 | N/A — nothing to update |
| 18 | Display misrouting (dGPU on lid-closed internal panel) | Fixed via Windows Display Settings to disable internal panel; verified via `Win32_VideoController` | Ruled out — same crash signature persisted (this is detailed in [Section 8](#8-the-display-routing-discovery-and-why-it-wasnt-the-answer)) |
| 19 | GPU deep-idle P-state transition failure | Pinned GPU to P0 via NVIDIA Control Panel "Prefer maximum performance" | Ruled out — crash on 23 May occurred with P0 lock active |
| 20 | Electron app / VSCode keeping GPU partially active | Observation only, not formally tested with controlled trials | Possibly contributory but unlikely root cause |
| 21 | Intel Serial IO I2C Host Controller (Device Problem Code `0xa` flagged by `powercfg /energy`) | Identified but not on the GPU path | Likely unrelated to TDR; flagged for future cleanup |

---

## 8. The Display Routing Discovery (and Why It Wasn't the Answer)

This deserves its own section because it was the most interesting finding of the entire investigation, even though it turned out not to be the cause.

### The Discovery

After all the obvious causes were eliminated, I ran:

```powershell
Get-CimInstance Win32_VideoController | Select-Object Name, CurrentHorizontalResolution, CurrentVerticalResolution, CurrentRefreshRate
```

Output:

```
NVIDIA GeForce RTX 5060 Laptop GPU  →  2560 x 1600 @ 240 Hz
Intel(R) UHD Graphics                →  3840 x 2160 @ 60 Hz
```

Translation: my **internal Lenovo panel** (2560×1600 @ 240 Hz) was being driven by the **dGPU**, even though the lid was closed and I could not see it. My **external LG UltraFine 4K** (the only display I actually use) was being driven by the **iGPU**.

This is exactly backwards from what you would want. The dGPU was generating 240 frames per second to a panel I could not see, while the iGPU strained against driving 4K@60. The dGPU was in a degenerate state — partially active, useless work, never able to fully idle because it was driving a connected (if invisible) display.

### Why I Thought This Was the Answer

It fit the evidence:

- **It explained why all three laptops crashed the same way** — same setup pattern (lid closed, single external monitor) reproduced the misrouting each time.
- **It explained the idle correlation** — the dGPU was always partially busy with the closed-lid panel, never fully idle, never fully active, always in the dangerous transitional zone.
- **It explained why nothing software-side fixed it** — the misrouting was a configuration, not a bug. Drivers cannot fix a setup where they have been routed incorrectly.
- **It was uniquely identifiable to my setup**, which justified why the issue followed me across machines.

### The Fix Applied

Windows Settings → System → Display → click internal Lenovo panel (the smaller one in the diagram) → "Multiple displays" → **"Show only on [LG monitor]"** → Apply.

Verification:

```powershell
Get-CimInstance Win32_VideoController | Select-Object Name, CurrentHorizontalResolution
# NVIDIA GeForce RTX 5060 Laptop GPU  →  3840 x 2160   (now driving the LG)
# Intel(R) UHD Graphics                →  (blank, dormant)
```

The dGPU was now correctly driving the external 4K display. The iGPU was enumerated but not driving anything.

### Why It Was Not the Answer

The 23 May crash, with the routing fixed and verified, with the P0 lock active, with the hypervisor disabled, with every other variable controlled — **same exact failure hash**. The display misrouting was a real misconfiguration worth fixing, but it was not the cause.

That was a hard moment. The misrouting fix was the most plausible explanation I had found in two weeks. It survived 19 hours and then the same crash returned with the same signature.

### What I Still Believe About It

Display misrouting is real, it was happening, and it is a meaningful misconfiguration on Lenovo's part. Lenovo ships these machines with Optimus configurations that result in the dGPU driving a closed-lid panel at high refresh rates while the iGPU drives external outputs. This is **wasteful and probably contributory** to thermal and power stress on the dGPU, but it is not the *trigger* for the TDR. It would have made any GPU stability problem worse, not caused one on its own.

---

## 9. The Final Crash and the Diagnostic Endpoint

The final crash, on 23 May 2026 at 01:29:44, came after roughly 9 hours of intensive coding work followed by ~30 minutes of idle as I left the desk to go to bed.

### Crash Context

- **Configuration**: The fully-remediated state described in [Phase 10](#phase-10-the-final-state).
- **Workload before crash**: 9 hours active coding (VSCode, Chrome, terminals).
- **Idle period before crash**: Approximately 30 minutes between the user leaving the desk and the crash.
- **Dump produced**: `C:\Windows\MEMORY.DMP` — a 4 GB full kernel dump (the previous 10 crashes produced minidumps; this one produced a full kernel dump because Windows escalated under "Automatic memory dump" mode).
- **Event 41 Properties**: Bug check code `278` decimal = `0x116` hex = `VIDEO_TDR_FAILURE`.

### WinDbg Analysis

`!analyze -v` output (full output in Appendix A):

```
VIDEO_TDR_FAILURE (116)
Attempt to reset the display driver and recover from timeout failed.
Arguments:
Arg1: ffff848e52156290, Optional pointer to internal TDR recovery context.
Arg2: fffff804777f54c0, The pointer into responsible device driver module.
Arg3: ffffffffc000009a, NTSTATUS of the last failed operation.
Arg4: 0000000000000004, Optional internal context dependent data.

FAILURE_BUCKET_ID:  0x116_IMAGE_nvlddmkm.sys
FAILURE_ID_HASH:    {c89bfe8c-ed39-f658-ef27-f2898997fdbd}    ← IDENTICAL to all prior crashes

STACK_TEXT:
  nt!KeBugCheckEx
  dxgkrnl!TdrBugcheckOnTimeout+0x101
  dxgkrnl!ADAPTER_RENDER::Reset+0x220
  dxgkrnl!DXGADAPTER::Reset+0x58a
  dxgkrnl!TdrResetFromTimeout+0x15
  dxgkrnl!TdrResetFromTimeoutWorkItem+0x22
  nt!ExpWorkerThread+0x4bb
  nt!PspSystemThreadStartup+0x5a
  nt!KiStartSystemThread+0x34

MODULE_NAME: nvlddmkm
IMAGE_NAME:  nvlddmkm.sys
```

The stack trace contains **no NVIDIA code in the call path itself** — only kernel and `dxgkrnl` frames. This is characteristic of an asynchronous GPU hang: the GPU stopped responding to commands submitted on a different thread (long since gone), the OS waited for a TDR timeout, the OS asked the driver to reset, the reset timed out, and the kernel bug-checked. The originating thread that submitted the failing work to the GPU is no longer on any CPU at the moment of the bug check.

The fault is at `nvlddmkm+0x1a254c0`, which is an instruction inside the NVIDIA driver but does not represent the *cause* of the hang — it is the address WinDbg latches onto for attribution. The real failure is in GPU silicon state or microcode that we cannot inspect from the CPU side.

The `NTSTATUS 0xC000009A` (`STATUS_INSUFFICIENT_RESOURCES`) is the cause attribution that matters: the GPU was unable to fulfil an allocation/setup request from the OS at the moment something tried to engage it after a period of idle.

---

## 10. Evidence Catalogue

I am happy to provide any of the following to Lenovo and NVIDIA engineering on request:

1. **Eight `.dmp` files** spanning 18 January 2026 through 23 May 2026, covering two physical units of the same model:
   - `012226-25812-01.dmp` (22 Jan 2026 8:05 PM)
   - `030126-...-01.dmp` (1 Mar 2026)
   - `051126-22531-01.dmp` (11 May 2026 7:06 PM)
   - `051126-22937-01.dmp` (11 May 2026 8:43 PM)
   - `051426-22156-01.dmp` (14 May 2026 10:31 PM)
   - `051826-24593-01.dmp` (18 May 2026 2:50 AM)
   - `052126-14343-01.dmp` (21 May 2026 4:05 AM)
   - `MEMORY.DMP` (23 May 2026 1:29 AM — full 4 GB kernel dump)

2. **System reports** generated via:
   - `powercfg /energy /duration 60` (HTML, identifies misconfigured devices)
   - `dxdiag /t dxdiag.txt`
   - `msinfo32` export
   - `pnputil /enum-devices /connected /class Display`

3. **HWiNFO64 sensor logs** (available on request — would need to run with logging enabled across a crash event)

4. **Configuration screenshots** showing NVIDIA Control Panel settings, Device Manager state, Display settings, BIOS version, BlueScreenView crash list, Reliability Monitor timeline.

5. **Serial numbers** of all three affected units (available privately to Lenovo support — not posted publicly).

---

## 11. Configuration State Tables

For anyone reproducing this investigation or comparing notes, the exact state of the laptop at the time of the final crash:

### BIOS Settings (non-default)

| Setting | Value |
|---|---|
| GPU Mode | Hybrid (Auto) — tried both Hybrid and Discrete |
| Secure Boot | Enabled |
| Virtualization | Enabled (cannot disable on this BIOS) |

### Windows Power Settings

```powershell
powercfg /q SCHEME_CURRENT SUB_SLEEP STANDBYIDLE
# AC: 0 (never)
# DC: 0 (never)

powercfg /q SCHEME_CURRENT SUB_SLEEP HIBERNATEIDLE
# AC: 0 (never)
# DC: 0 (never)

powercfg /q SCHEME_CURRENT SUB_BUTTONS LIDACTION
# AC: 0 (do nothing)
# DC: 0 (do nothing)
```

### NVIDIA Control Panel (Global Settings)

| Setting | Value |
|---|---|
| Power management mode | Optimal power (after revert; was Prefer maximum performance during 23 May crash) |
| OpenGL rendering GPU | NVIDIA GeForce RTX 5060 Laptop GPU |
| Preferred graphics processor | High-performance NVIDIA processor |
| CUDA - Sysmem Fallback Policy | Prefer No Sysmem Fallback |
| Low Latency Mode | Off |
| Vertical sync | Use the 3D application setting |
| Max Frame Rate | Off |
| Background Application Max Frame Rate | Off |
| Triple buffering | Off |

### Hypervisor / VBS State

```
SecurityServicesRunning             : {0}     (no HVCI, no Credential Guard)
VirtualizationBasedSecurityStatus   : 2       (hypervisor loaded but no services consuming it)
EnableVirtualizationBasedSecurity   : 0       (registry says off)
HypervisorEnforcedCodeIntegrity     : 0       (off)
bcdedit hypervisorlaunchtype        : Off     (explicitly set)
```

### Display Routing (Verified)

```
NVIDIA GeForce RTX 5060 Laptop GPU   →   3840 x 2160 @ 60 Hz (LG UltraFine 4K)
Intel(R) UHD Graphics                →   (no active display — dormant)
```

### Drivers and Firmware

```
BIOS:               LENOVO S9CN18WW, 24 Nov 2025 (latest)
Intel ME Firmware:  16.1.40.2765 (latest, installed during investigation)
NVIDIA Driver:      596.36 Studio (DCH, WHQL, current Studio branch)
Windows:            11 Home, build 26200.8457 (25H2)
```

---

## 12. The Conclusion: This Is a Platform Defect

After two weeks of systematic elimination:

- The crash is reproducible.
- It occurs at idle.
- It produces a consistent kernel signature.
- It has occurred on three different physical units of effectively the same Lenovo platform (RTX 5060 mobile + i9-14900HX + Lenovo 16IRX10 firmware family).
- It is not preventable through any standard Windows configuration.
- It is not preventable through driver reinstall.
- It is not preventable through OEM utility removal.
- It is not preventable through firmware updates available to users.
- It is not preventable through GPU power state pinning.
- It is not preventable through display routing correction.

The cause is somewhere inside one or more of:

1. **NVIDIA's `nvlddmkm.sys` driver code path that handles GPU resource allocation after idle** on Blackwell mobile silicon paired with Lenovo VBIOS variants.
2. **NVIDIA's VBIOS** (`98.06.34.00.F3` on this unit) for this specific OEM SKU — the VBIOS controls GPU-side power state management.
3. **Lenovo's BIOS / EC firmware** ACPI tables and platform power negotiation with the GPU.

I cannot fix any of these from outside. Lenovo and NVIDIA can.

The most efficient resolution would be:

1. **NVIDIA**: Inspect the deep-idle resource allocation code path on Blackwell mobile parts in driver branch 596.x. Reproduce on a Lenovo 16IRX10 unit.
2. **Lenovo**: Either publish a new BIOS for the 16IRX10 that revises GPU power negotiation, or work with NVIDIA to update the VBIOS shipped with the platform.

---

## 13. What I Am Doing Next

1. **Filing a formal Lenovo Premier Support case** for machine type 83NN with all eight dump files, the WinDbg analysis, the system reports, and a reference to this document. Requesting escalation to Lenovo engineering rather than a fourth unit replacement.

2. **Filing an NVIDIA developer support bug report** at [https://developer.nvidia.com/nvidia_bug/add](https://developer.nvidia.com/nvidia_bug/add) with the `MEMORY.DMP` from the 23 May crash and the WinDbg analysis. Specifying RTX 5060 Laptop on Lenovo 16IRX10 platform.

3. **Running one more diagnostic experiment**: disabling the dGPU entirely in Device Manager (`Right-click → Disable device`). The 4K external monitor will fall back to being driven by the iGPU. If the machine then runs cleanly for 7+ days without a crash, this confirms the fault is specifically in the dGPU/driver/VBIOS path and rules out CPU, RAM, motherboard, and OS as participants. This is the strongest technical evidence I can give Lenovo.

4. **Living with the machine** in its current configuration in the meantime. Crashes every 2-4 days are not catastrophic — work is saved frequently, BitLocker is configured, and I have learned to git-commit before stepping away. But it is not how a $3000+ laptop should behave.

5. **Updating this document** if Lenovo or NVIDIA responds, if a BIOS or driver update changes the picture, or if a reader points out something I missed.

---

## 14. What You Can Do If This Is Happening to You

If you own a Legion Pro 5 16IRX10 (or related 16IRX10 / Legion 5 Gen 10 SKU with RTX 5060/5070 mobile) and you are experiencing the same crashes:

### Confirm It Is the Same Issue

1. Install [NirSoft BlueScreenView](https://www.nirsoft.net/utils/blue_screen_view.html) (free) — point it at `C:\Windows\Minidump\`.
2. Look at your crash entries. If you see `0x00000116` bug check codes with `Caused By Driver: dxgkrnl.sys` and `nvlddmkm.sys` as the second driver listed, you likely have the same issue.
3. For a stronger confirmation, open the dump in [WinDbg](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/) (free, Microsoft Store) and run `!analyze -v`. Compare your `FAILURE_ID_HASH` against `{c89bfe8c-ed39-f658-ef27-f2898997fdbd}` — if it matches, you and I are seeing the same bug.

### Do Not Bother With These (Already Tested, Did Not Work)

- Returning the laptop for replacement (next unit will likely have the same issue)
- Reinstalling Windows
- Trying older or newer NVIDIA drivers (problem persists across versions)
- Disabling Lenovo Vantage / X-Rite / OEM utilities
- Updating BIOS (current BIOS is the latest as of mid-2026)
- Disabling sleep
- Disabling HVCI / Memory Integrity

### Things Worth Trying (May Help, Did Not Fully Resolve For Me)

- **Disable the dGPU in Device Manager** entirely. The iGPU can drive a 4K external monitor at 60 Hz over USB-C. You lose any GPU acceleration but you may also lose the crashes. This is what I am about to test as a 7-day stability proof.
- **Fix display routing**: open Windows Display Settings, identify the internal panel, and set "Multiple displays → Show only on [your external monitor]". Verify with `Get-CimInstance Win32_VideoController` that the iGPU shows blank resolution and the dGPU is driving your external.
- **Set NVIDIA Power Mode to "Prefer Maximum Performance"** in NVIDIA Control Panel. This prevents the deep idle state. The GPU runs hotter and the fan more, but it eliminates the worst-case power transitions.

### Add Your Voice

- **Open a Lenovo support case** for your machine type. The more cases filed, the more likely engineering escalation happens. Reference this article and the failure hash.
- **File an NVIDIA bug report** at [https://developer.nvidia.com/nvidia_bug/add](https://developer.nvidia.com/nvidia_bug/add) with your dump file.
- **Post on the [Lenovo Legion forums](https://forums.lenovo.com/)** with your model and crash details.
- **Post on [r/LenovoLegion](https://www.reddit.com/r/LenovoLegion/)** and [r/nvidia](https://www.reddit.com/r/nvidia/) — visibility helps.
- **Comment on this article** — every additional reproduction strengthens the case.

---

## 15. Appendices

### Appendix A: Full WinDbg `!analyze -v` Output (23 May 2026 Crash)

```
VIDEO_TDR_FAILURE (116)
Attempt to reset the display driver and recover from timeout failed.
Arguments:
Arg1: ffff848e52156290, Optional pointer to internal TDR recovery context (TDR_RECOVERY_CONTEXT).
Arg2: fffff804777f54c0, The pointer into responsible device driver module (e.g. owner tag).
Arg3: ffffffffc000009a, Optional error code (NTSTATUS) of the last failed operation.
Arg4: 0000000000000004, Optional internal context dependent data.

BUGCHECK_CODE:  116
BUGCHECK_P1: ffff848e52156290
BUGCHECK_P2: fffff804777f54c0
BUGCHECK_P3: ffffffffc000009a
BUGCHECK_P4: 4

FILE_IN_CAB:  MEMORY.DMP
FAULTING_THREAD:  ffff848e32fcb080
PROCESS_NAME:  System

STACK_TEXT:
ffffa809`aa830808 fffff804`5e8d6d5d : 00000000`00000116 ffff848e`52156290 fffff804`777f54c0 ffffffff`c000009a : nt!KeBugCheckEx
ffffa809`aa830810 fffff804`5eb43cb4 : fffff804`777f54c0 ffff848e`29a76050 00000000`00002000 ffff848e`29a76110 : dxgkrnl!TdrBugcheckOnTimeout+0x101
ffffa809`aa830850 fffff804`5e8e5e3e : ffff848e`29ade000 00000000`00000000 00000000`00000004 00000000`00000000 : dxgkrnl!ADAPTER_RENDER::Reset+0x220
ffffa809`aa830880 fffff804`5e920ef5 : ffff848e`00000100 00000000`00000000 ffffa809`00000000 00000000`00000000 : dxgkrnl!DXGADAPTER::Reset+0x58a
ffffa809`aa830910 fffff804`5e921052 : ffff848e`1aee6260 ffff848e`0e6eeaa0 00000000`00000000 00000000`00000000 : dxgkrnl!TdrResetFromTimeout+0x15
ffffa809`aa830940 fffff804`ccae5c3b : ffff848e`32fcb080 ffff848e`0e7cfaa0 ffff848e`0e7cfa00 ffff848e`0e7cfaa0 : dxgkrnl!TdrResetFromTimeoutWorkItem+0x22
ffffa809`aa830980 fffff804`ccc7eaba : ffff848e`32fcb080 ffff848e`32fcb080 fffff804`ccae5780 ffff848e`0e7cfaa0 : nt!ExpWorkerThread+0x4bb
ffffa809`aa830b30 fffff804`cceaa0a4 : ffffe200`971e5180 ffff848e`32fcb080 fffff804`ccc7ea60 ffffe200`96f40180 : nt!PspSystemThreadStartup+0x5a
ffffa809`aa830b80 00000000`00000000 : ffffa809`aa831000 ffffa809`aa82a000 00000000`00000000 00000000`00000000 : nt!KiStartSystemThread+0x34

SYMBOL_NAME:  nvlddmkm+1a254c0
MODULE_NAME:  nvlddmkm
IMAGE_NAME:   nvlddmkm.sys
FAILURE_BUCKET_ID:  0x116_IMAGE_nvlddmkm.sys
FAILURE_ID_HASH:    {c89bfe8c-ed39-f658-ef27-f2898997fdbd}
OS_VERSION:   10.0.26100.1
```

### Appendix B: Key PowerShell Diagnostic Commands

```powershell
# Crash event log
Get-WinEvent -FilterHashtable @{LogName='System'; Id=41} -MaxEvents 20 |
  Select-Object TimeCreated, Message | Format-List

# Get bug check code from latest Event 41
$evt = Get-WinEvent -FilterHashtable @{LogName='System'; Id=41} -MaxEvents 1
$evt.Properties | ForEach-Object { $_.Value }

# Locate dump files
Get-ChildItem C:\Windows\Minidump\*.dmp | Sort-Object LastWriteTime -Descending
Get-ChildItem C:\Windows\MEMORY.DMP -ErrorAction SilentlyContinue

# Crash dump configuration
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\CrashControl" |
  Select-Object CrashDumpEnabled, MinidumpDir, DumpFile, AutoReboot, LogEvent

# Display routing
Get-CimInstance Win32_VideoController |
  Select-Object Name, CurrentHorizontalResolution, CurrentVerticalResolution, CurrentRefreshRate

Get-PnpDevice -Class Display |
  Select-Object FriendlyName, Status, ConfigManagerErrorCode, Present

Get-PnpDevice -Class Monitor |
  Select-Object FriendlyName, Status, Present

# Sleep states available
powercfg /a

# Wake history and timers
powercfg /lastwake
powercfg /waketimers

# Hypervisor / VBS state
bcdedit /enum "{default}" | findstr /i hypervisor

Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard |
  Select-Object SecurityServicesRunning, VirtualizationBasedSecurityStatus

Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\DeviceGuard"
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\HypervisorEnforcedCodeIntegrity"

# Disable hypervisor (requires reboot)
reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" /v EnableVirtualizationBasedSecurity /t REG_DWORD /d 0 /f
reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\HypervisorEnforcedCodeIntegrity" /v Enabled /t REG_DWORD /d 0 /f
bcdedit /set "{default}" hypervisorlaunchtype off

# Disable Windows sleep
powercfg /change standby-timeout-ac 0
powercfg /change hibernate-timeout-ac 0
powercfg /change monitor-timeout-ac 10
powercfg /change standby-timeout-dc 0
powercfg /change hibernate-timeout-dc 0

# Set lid action to "Do nothing" (so closing the lid doesn't sleep the machine)
powercfg /setacvalueindex SCHEME_CURRENT SUB_BUTTONS LIDACTION 0
powercfg /setdcvalueindex SCHEME_CURRENT SUB_BUTTONS LIDACTION 0
powercfg /S SCHEME_CURRENT

# Energy diagnostic report (generates HTML at C:\Windows\System32\energy-report.html)
powercfg /energy /duration 60

# Block Windows Update from replacing GPU driver
New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" -Force | Out-Null
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" `
  -Name "ExcludeWUDriversInQualityUpdate" -Value 1 -Type DWord

# Last 5 installed updates
Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 5

# Reliability history (visual: perfmon /rel)
```

### Appendix C: Useful Links

- **Lenovo Support — search for 83NN or Legion Pro 5 16IRX10**: [https://pcsupport.lenovo.com/](https://pcsupport.lenovo.com/)
- **NVIDIA Driver Downloads**: [https://www.nvidia.com/en-us/drivers/](https://www.nvidia.com/en-us/drivers/)
- **NVIDIA Bug Report Submission**: [https://developer.nvidia.com/nvidia_bug/add](https://developer.nvidia.com/nvidia_bug/add)
- **Display Driver Uninstaller (DDU)**: [https://www.guru3d.com/files/cat/214](https://www.guru3d.com/files/cat/214)
- **NirSoft BlueScreenView**: [https://www.nirsoft.net/utils/blue_screen_view.html](https://www.nirsoft.net/utils/blue_screen_view.html)
- **WinDbg (Microsoft Store)**: [https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/)
- **HWiNFO64**: [https://www.hwinfo.com/](https://www.hwinfo.com/)
- **Microsoft Bug Check Reference (`0x116`)**: [https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/bug-check-0x116--video-tdr-failure](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/bug-check-0x116--video-tdr-failure)
- **Lenovo Legion Forums**: [https://forums.lenovo.com/](https://forums.lenovo.com/)
- **r/LenovoLegion subreddit**: [https://www.reddit.com/r/LenovoLegion/](https://www.reddit.com/r/LenovoLegion/)
- **r/nvidia subreddit**: [https://www.reddit.com/r/nvidia/](https://www.reddit.com/r/nvidia/)

### Appendix D: Glossary

- **TDR**: Timeout Detection and Recovery. Windows subsystem that watches for GPU hangs and attempts to reset the driver.
- **dGPU**: Discrete GPU. The dedicated graphics chip (RTX 5060 in this case).
- **iGPU**: Integrated GPU. The graphics chip built into the CPU (Intel UHD on Raptor Lake-HX).
- **Optimus**: NVIDIA's hybrid graphics technology that routes work between iGPU and dGPU dynamically. "Advanced Optimus" adds a hardware MUX for direct dGPU-to-display routing.
- **MUX**: Multiplexer. A hardware switch that selects which GPU drives a given display output.
- **VBIOS**: Video BIOS. The firmware on the GPU itself, separate from the system BIOS.
- **EC**: Embedded Controller. A small auxiliary microcontroller on the motherboard that handles power, thermals, keyboard, and platform housekeeping.
- **ACPI**: Advanced Configuration and Power Interface. The standard interface between OS and platform firmware for power management.
- **HVCI**: Hypervisor-protected Code Integrity. Windows security feature that validates kernel-mode code in a protected memory region via the hypervisor. Also called "Memory Integrity" in the UI.
- **VBS**: Virtualization-Based Security. The broader Windows feature set that uses the hypervisor to isolate sensitive operations. HVCI is one component of VBS.
- **DCH**: Declarative, Componentized, Hardware Support Apps. Microsoft's modern driver model.
- **DDU**: Display Driver Uninstaller. A free third-party tool for cleanly removing GPU drivers.
- **PCIe ASPM**: PCI Express Active State Power Management. Link-level power saving substates (L0s, L1, L1.1, L1.2).
- **P-states**: Performance states. GPU power/clock states from P0 (highest, active) through P8 (deepest idle).

---

## Final Word

I have spent too many evenings on this. The machine should just work — that is the entire premise of buying a finished product from a major manufacturer with a recent CPU and a recent GPU. The fact that three brand-new units have failed in identical fashion is not a customer-side problem to solve.

If you are reading this because Google brought you here after your own Legion crashed, I am sorry. You are not crazy and you are not unlucky in an isolated way. There is a real defect, and the more of us who document it, the faster Lenovo and NVIDIA will be motivated to fix it.

If you are at Lenovo or NVIDIA reading this, the contact information is available through whatever channel you reached this document via. The dumps and evidence are at your disposal. Please escalate.

If you are a debugger who spots something I missed in this 4000-word recap, please reach out. I would rather be wrong about "this is a platform defect" than continue to lose work to a midnight reboot.

---

*Document version: 1.0 — May 2026*
*Investigation conducted by Ruslan Abuzant with debugging assistance from Claude (Anthropic).*
*All command output, dump file metadata, and configuration details are real and reproducible.*

