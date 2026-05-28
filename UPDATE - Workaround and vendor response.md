# UPDATE — 28 May 2026: A Reproducible Workaround, and What It Proves

> This is a follow-up to the original investigation: [Three Laptops, Three Months, One Bug](./README.md). Read that first for the full background, dump analysis, and elimination methodology.
>
> **Two things have happened since the original writeup:**
>
> 1. I have identified a **reproducible workaround** that prevents the crash, which in turn isolates the failure mechanism to a specific GPU code path.
> 2. I posted the investigation to Lenovo's official community forum. A Lenovo Support Specialist engaged constructively. A Lenovo **Community Administrator then overrode that engagement, edited the specialist's reply, and declined to escalate** — on the grounds that I had used NVIDIA's own current driver rather than Lenovo's older repackaged copy.
>
> This update addresses both, because together they make a single point: **the crash is a reproducible defect in the GPU driver/firmware stack, not a consequence of my driver choice — and I can now prove it from both directions.**

---

## Part 1 — The Reproducible Workaround

### The Hypothesis (Restated)

The original investigation concluded the crash mechanism lives in the **dGPU's deep-idle entry/exit path**. The evidence: bug check `0x116 VIDEO_TDR_FAILURE` with `NTSTATUS 0xC000009A (STATUS_INSUFFICIENT_RESOURCES)`, always at idle, always the same failure hash (`c89bfe8c-ed39-f658-ef27-f2898997fdbd`), persisting across every software remediation.

The deep-idle hypothesis makes a falsifiable prediction:

> **If the dGPU is prevented from entering deep idle states, the crash should not occur.**

### The Test

I kept the dGPU continuously engaged by running a single hardware-accelerated video decode workload — a silenced YouTube livestream in a browser tab — with the machine otherwise idle (locked, lid closed, external 4K monitor as the only active display, no other user activity).

Everything else remained in the fully-remediated configuration from the original investigation:

- NVIDIA Studio Driver 596.36 (current, WHQL, DCH)
- BIOS `S9CN18WW` (latest)
- Intel ME firmware `16.1.40.2765` (latest)
- HVCI off, hypervisor disabled
- Clean Boot, X-Rite uninstalled
- Display routed dGPU → external 4K, internal panel disabled
- OS sleep disabled (`PresentationSettings.exe` holding SYSTEM + DISPLAY)

### The Result

| Metric | Value |
|---|---|
| **Last crash** | 23 May 2026, 01:29:44 |
| **Current uptime** | 5 days, 19 hours, 27 minutes (as of 28 May 2026, 20:59) |
| **Crashes during this window** | **0** |
| **Crashes in the 14 days *prior* to the workaround, same configuration but GPU allowed to idle** | **6** (11 May ×2, 14 May, 18 May, 21 May, 23 May) |

For context on why this matters: during the active crash cluster, the **longest gap between crashes was 4 days** (14 May → 18 May). The machine has now been clean for **nearly 6 days** — past that worst-case gap — with the only change being a continuous GPU workload.

```powershell
PS C:\Windows\system32> Get-WinEvent -FilterHashtable @{LogName='System'; Id=41} -MaxEvents 3 | Select-Object TimeCreated
TimeCreated
-----------
5/23/2026 1:29:44 AM     # <-- last crash, before workaround
5/21/2026 4:06:12 AM
5/18/2026 2:51:10 AM

PS C:\Windows\system32> Get-Date
Thursday, May 28, 2026 8:59:25 PM

# Uptime: 5 days, 19 hours, 27 minutes — zero Event 41 since the workaround began
```

### What This Demonstrates

This is a **controlled before/after with a single changed variable**:

- **Without** continuous GPU activity: 6 crashes in 14 days, all at idle, identical signature.
- **With** continuous GPU activity: ~6 days clean and counting.

The variable that changed is whether the dGPU was permitted to enter deep idle. The crash correlates inversely with GPU activity. This is the definition of a **reproduction/prevention recipe**, and it is exactly the kind of artifact a driver or platform engineering team can validate on a test unit within a single day.

### Caveats (Stated Honestly)

I want this document to be credible, so here are the limits of the claim:

1. **6 days is strong but not infinite.** Earlier in this laptop's history there were dormant periods of several weeks between crash *clusters*. I will continue running and will update at the 14-day and 30-day marks. But within an **active** cluster — which is what the preceding 14 days were — 6 clean days under a deliberate intervention is a meaningful contrast.
2. **The minimal sufficient fix is not yet isolated.** The current clean run has both (a) OS-sleep prevention via `PresentationSettings` and (b) continuous GPU workload via YouTube. I have not yet run a controlled test to determine whether (b) alone is necessary, or whether (a) alone would suffice. That controlled testing comes next. Either way, the *combination* works, and either way the fix is a software/configuration workaround for an underlying defect — not a hardware repair.
3. **A workaround is not a fix.** Keeping a GPU artificially busy 24/7 to prevent a kernel crash is not an acceptable end state for a flagship laptop. It increases power draw, heat, and fan noise. It is a diagnostic finding and a stopgap, not a resolution. The resolution has to come from NVIDIA (driver/VBIOS) or Lenovo (BIOS/EC firmware).

---

## Part 2 — The Vendor Response, and Why It Is Wrong

### What Happened on the Lenovo Forum

I posted the full investigation to Lenovo's official community forum (Gaming Laptops section). The sequence:

1. **A Lenovo Support Specialist** ("Hassan_Lenovo", 20,000+ posts, registered 2018) replied substantively. He acknowledged:
   - The consistent crash signature "suggests a systemic driver/firmware interaction problem rather than isolated hardware defects."
   - "Seeing the same failure across three systems points to a broader compatibility issue with the RTX 5060 stack."
   - "Filing with Lenovo and NVIDIA support is the right step, as this requires engineering-level review."

2. **A Lenovo Community Administrator** ("Andy") then **edited the Support Specialist's reply** and posted that the Community Team **will not be opening a case nor escalating this issue.** His stated reasoning:

   > "We have frequently found that the symptoms described, graphics driver crashes, have been related to a driver version having been installed other than the one offered on the Lenovo support site for the specific system type."

   He then quoted the part of my investigation noting that NVIDIA Studio Driver 596.36 "doesn't appear to be from the Lenovo support site," and concluded:

   > "The fact that the symptom was reproducible on 3 systems would come as no surprise if the same non-Lenovo driver was used on all 3 systems."

### Why This Reasoning Does Not Hold

There are four independent problems with this conclusion.

#### Problem 1 — The crashes began on Lenovo's own factory drivers, before any clean install.

This is the central factual error. The DDU + NVIDIA Studio driver install was a **remediation attempt** performed in **May 2026 on the current unit only**. The crash history predates it:

| Crash | Date | Driver in use at the time |
|---|---|---|
| Unit #1 | Late 2025 | **Lenovo factory stock** |
| Unit #2 | 18–22 Jan, 1 Mar 2026 | **Lenovo factory stock** |
| Unit #3 | 11 May 2026 (first crashes) | **Lenovo factory stock** |
| Unit #3 | 18–23 May 2026 | NVIDIA Studio 596.36 (installed *after* the 11 May crashes as a fix attempt) |

**At least six crashes occurred on Lenovo's own shipped drivers, before any non-Lenovo driver was ever installed.** The minidumps from January through 11 May are all on factory configuration. The administrator's reasoning — "the crash reproduces because you used a non-Lenovo driver" — is contradicted by the document he was replying to, which lays out this exact timeline.

If anything, the persistence of the **identical crash signature across both Lenovo's factory driver and NVIDIA's current Studio driver** is *itself* the evidence that this is not a driver-version issue. The same `FAILURE_ID_HASH` appeared under both. A driver-version problem would not survive a complete driver replacement.

#### Problem 2 — The "non-Lenovo driver" is the GPU manufacturer's own current, WHQL-certified driver.

NVIDIA Studio Driver 596.36 is:

- Published by **NVIDIA**, the manufacturer of the GPU in this laptop.
- **WHQL-certified** (signed by Microsoft's Windows Hardware Quality Labs).
- The **DCH** variant (Microsoft's current driver model).
- Listed on [NVIDIA's official Studio Driver page](https://www.nvidia.com/en-us/drivers/) as supporting the RTX 5060 Laptop GPU.

Lenovo's "support site driver" is, in the overwhelming majority of cases, an **older snapshot of the same NVIDIA driver**, repackaged by Lenovo and updated on a slower cadence. Recommending that a customer downgrade to an older repackaged driver — and asserting that using the manufacturer's current driver *caused* the defect — is not a technically defensible position. It is also inconsistent with how Lenovo markets this laptop as a developer/creator workstation, since Studio drivers are precisely what that audience is told to install.

#### Problem 3 — The workaround result rebuts the driver-version theory directly.

If this were a bad-driver problem, then keeping the GPU busy would not prevent it — a bad driver would crash regardless of GPU load. Instead, the **same driver** (596.36) produces:

- **Crashes** when the GPU is allowed to idle.
- **No crashes** when the GPU is kept busy.

The variable is not the driver. The driver is constant across both outcomes. The variable is the GPU's idle state. That points to a **power-management defect in the driver/VBIOS/firmware stack**, triggered by deep idle — not to the choice of driver version.

#### Problem 4 — A Community Administrator overrode a Support Specialist's engineering recommendation.

The Support Specialist recommended engineering-level review. A community moderator — whose role is moderation, not engineering triage — edited that reply and unilaterally closed the path to escalation. Whatever the merits of the technical argument, the **process** here means a reproducible kernel-level defect, with eight dumps and a reproduction recipe, will not reach Lenovo engineering because a forum administrator decided it should not.

### The Net Effect

The stated Lenovo community position is now, in effect:

- The current WHQL-signed driver from the GPU's own manufacturer is unacceptable on a Lenovo laptop.
- Crashes that occurred on Lenovo's factory drivers, months before any driver change, are attributable to a driver change that happened later.
- The fix is to install an older repackaged driver.
- Engineering will not see the evidence.

I do not accept this, and I have laid out above why it does not survive contact with the timeline or the reproduction data.

---

## Part 3 — Current Status and Asks

### Filed

- **NVIDIA developer bug report**: submitted and acknowledged (ID on file).
- **Lenovo community forum thread**: posted; escalation declined by community administrator (this section documents that).
- **Lenovo formal warranty channel**: in progress (separate from the forum).

### Open Asks to NVIDIA and Lenovo Engineering

1. **Validate the reproduction recipe.** On a Legion Pro 5 16IRX10 (machine type 83NN) with RTX 5060 Laptop GPU: leave the machine idle with the dGPU permitted to enter deep idle, lid closed, external display connected. Observe for `0x116` over several days. Then repeat with a continuous GPU workload and observe the absence of crashes.
2. **Inspect the deep-idle resource-allocation path** in `nvlddmkm.sys` branch 596.x on Blackwell mobile, around the `STATUS_INSUFFICIENT_RESOURCES` return during wake-from-deep-idle.
3. **Examine the VBIOS / platform power negotiation** between the GPU and Lenovo's EC/BIOS on this SKU.

### Evidence Available on Request

- 8 kernel dumps (Jan–May 2026, two physical units), including a 4 GB full kernel `MEMORY.DMP`.
- Full WinDbg `!analyze -v` output for the two most recent crashes.
- `powercfg /energy` report, `dxdiag`, `msinfo32`, PnP device enumeration.
- Complete configuration documentation and the workaround test data above.

---

## Part 4 — For Other Affected Owners

If you own a Legion Pro 5 16IRX10 (or a related 16IRX10 / Legion Gen 10 SKU with RTX 5060/5070 mobile) and you are seeing `0x116 VIDEO_TDR_FAILURE` crashes at idle:

**Immediate stopgap** (not a fix, but it has kept my machine up for ~6 days and counting):

1. Keep the dGPU continuously busy. The simplest method: open a hardware-accelerated video (e.g. a silenced livestream) in a browser tab and leave it playing. Make sure your browser's tab-sleeping / memory-saver feature is **not** suspending that tab — verify the video is still actually playing.
2. Optionally also prevent OS-level idle: run `presentationsettings /start` (built into Windows).

**Confirm you have the same bug**: open your minidumps in [WinDbg](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/), run `!analyze -v`, and compare your `FAILURE_ID_HASH` to `c89bfe8c-ed39-f658-ef27-f2898997fdbd`. If it matches, you are seeing the same defect.

**Add your voice**: the more reproductions are documented publicly, the harder this is to dismiss as individual user error.

- NVIDIA bug report: [https://developer.nvidia.com/nvidia_bug/add](https://developer.nvidia.com/nvidia_bug/add)
- Lenovo forums: [https://forums.lenovo.com/](https://forums.lenovo.com/)
- r/LenovoLegion: [https://www.reddit.com/r/LenovoLegion/](https://www.reddit.com/r/LenovoLegion/)
- r/nvidia: [https://www.reddit.com/r/nvidia/](https://www.reddit.com/r/nvidia/)

---

## Closing

The original investigation eliminated every reasonable software cause and concluded this is a platform defect. This update adds the other half of the proof: **I can reproduce the crash by allowing the GPU to idle, and I can prevent it by keeping the GPU busy — using the exact same driver throughout.**

That is not consistent with "you used the wrong driver." It is consistent with a power-management bug in the GPU driver/firmware stack that triggers on deep idle. The recommendation to install a months-old repackaged driver does not address it, and declining to escalate it does not make it go away — it just means the next owner rediscovers it from scratch.

I remain available to NVIDIA and Lenovo engineering with the full evidence package, through any channel they prefer.

---

*Update version: 1.0 — 28 May 2026*
*Part of the ongoing investigation at [this repository]. Will be updated at the 14-day and 30-day uptime marks, and if either vendor engages.*
