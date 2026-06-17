# Update: Lenovo "Premium" Support took 23+ days to respond, sent me a troubleshooting script, I did all of it, crashed again in 25 minutes

Follow-up to my earlier deep-dive on the VIDEO_TDR_FAILURE crashes on the Legion Pro 5 16IRX10 (RTX 5060). For anyone just finding this: three brand-new units, same kernel crash, same failure hash, at idle. Full investigation is linked in my profile/previous post.

Here's where things stand.

## The 23-day silence

I opened a support ticket with full documentation — eight crash dumps, WinDbg analysis, the whole thing. I have Premium Care. It sat untouched for **23+ days**. No response, no acknowledgment, nothing.

What finally moved it? I picked up the phone and complained that a Premium Care ticket had been ignored for nearly a month. Got a reply within **5 minutes**. Funny how that works.

## The response

To their credit, the engineer who replied actually acknowledged the issue in writing — recognized the crash signature, agreed it "points toward a software/firmware conflict rather than isolated hardware," and escalated to Tier 3 Engineering. That part was genuinely better than the forum brush-off I got earlier (where a community admin literally edited a support specialist's reply and refused to escalate — but that's a different story).

Then they sent the standard troubleshooting script:

1. DDU + reinstall the **Lenovo OEM driver** (not NVIDIA's current one)
2. Disable Hardware-Accelerated GPU Scheduling (HAGS)
3. Set TdrDelay = 10
4. Update BIOS
5. Vantage thermal mode + confirm Hybrid GPU mode

## So I did all of it. Exactly as asked.

- DDU'd in safe mode, installed their exact validated OEM driver (32.0.15.8205)
- Disabled HAGS
- TdrDelay was already at 10 (and crashes were happening anyway — so the GPU is hard-hanging, not just slow)
- BIOS already latest, ME firmware already latest
- Thermal → Balanced, GPU → Hybrid confirmed

Everything verified. Bare config, GPU left to idle, no workarounds running.

## Result: crashed 25 minutes after I locked it and went to take a shower.

Same bug check (0x116), same `dxgkrnl+186d5d` offset, same `STATUS_INSUFFICIENT_RESOURCES`, same everything — now on **Lenovo's own validated driver** with **their own recommended settings**.

I have honestly never been so happy to come back to a rebooted machine. Because this kills their last excuse. There is no longer any "you used the wrong driver" argument — it crashes on theirs too. Every driver tested now fails: factory, their OEM (twice), and NVIDIA Studio.

## Where it goes from here

- Sent Tier 3 a full report: every step documented, the crash on their own driver, and a request to either commit to an engineering fix with a timeline OR replace/buy back the unit.
- NVIDIA, meanwhile, **accepted my bug report** and is actively reviewing the full kernel dump. The chip maker is taking it more seriously than the laptop maker did for 23 days.
- An on-site technician is being dispatched. I'll let them swap parts — if it still crashes after a board swap (which I expect), that's just more evidence it's systemic.

**For anyone else with this laptop:** if you're getting 0x116 crashes at idle, you're not crazy and your unit isn't a one-off lemon. Check your minidumps in BlueScreenView — if you see the same hash/offset, comment here. The more of us who document it, the harder it is for them to keep calling it user error.

Will update when Tier 3 responds or when the tech visit happens.
