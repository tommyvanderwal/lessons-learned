# BeQuiet Dark Power Pro 13 multi-rail OCP can hard-power-off RTX PRO 6000 Blackwell under LLM load (fixed by OCK / single-rail)

**TL;DR:** On a system with a **PNY NVIDIA RTX PRO 6000 Blackwell (600W)** and a **be quiet! Dark Power Pro 13 1600W**, sustained LLM prompt processing repeatedly caused a **full power cut** that required toggling the PSU switch to recover. Switching the PSU to **single-rail mode via the Overclocking Key (OCK)** stopped the shutdowns and the GPU could run near **600W** continuously.

This write-up is meant to be a searchable, reusable “lesson learned” for anyone hitting similar symptoms (especially with very high-power GPUs such as RTX PRO 6000 Blackwell / RTX 5090-class parts).

---

## Symptoms

- System runs normally until an intense GPU workload begins (LLM “prompt processing / prefill” was the most reproducible trigger).
- Then the machine after 20+ seconds 95%+ GPU load **instantly powers off** (no clean shutdown).
- **Recovery requires physically flipping the PSU rocker switch or unplugging AC**.
- During the “dead” state:
  - the **front panel power LED remained lit**
  - the system draws only **standby-level power** at the wall (single-digit watts)
- Happens across OSes (observed on **Linux and Windows**), so it is not an OS-only issue.
- Happend during LM Studio inferencing on GPT-OSS-120B Prompt Processing a 80K+ prompt. 
- Usually around 49K prompt processing in both Windows with LM studio 3.3x and in Linux LM studio 4.2.2.
- Both with 4K and 8K chunk sizes (4096 and 8192) Regardless of parallel sessions settings.

**Why this matters:** If you must toggle AC/PSU to recover, you’re almost certainly dealing with a **PSU protection latch** (OCP/OPP/UVP/etc.), not a typical driver crash.

---

## Hardware

- **GPU:** PNY **NVIDIA RTX PRO 6000 Blackwell** (Workstation Edition, ~600W class)
- **PSU:** **be quiet! Dark Power Pro 13 1600W**
- **Platform:** AM5 / Asus ProArt X670E-CREATOR WIFI mainboard system (Ryzen 7800X3D CPU) - 128GB (2 x 64GB) DDR5
- **Workload trigger:** LLM inference with very heavy prompt processing (prefill), high GPU utilization, large VRAM use.

> Note: Only PNY currently sells the RTX PRO 6000 Blackwell Workstation Edition (as far as I know at the time of writing).

---

## What I tried that did **NOT** fix it

These steps did **not** stop the shutdowns while the PSU remained in default multi-rail mode:

- Forcing **PCIe Gen4** in BIOS
- Disabling PCIe power saving features (ASPM, etc.)
- Trying both **12V-2x6 / 12VHPWR** ports on the PSU
- Using the native be quiet 12V-2x6 cable (no adapter)
- Power-limiting the GPU in software (e.g., `nvidia-smi -pl 300`)
- Adding the second **EPS 8-pin** CPU power connector to the motherboard
- Switching OS / driver stack (Ubuntu, but reproduced on Windows too)

---

## The key observation that changed everything

Because the system required **AC power removal / PSU switch toggle** to recover, and because it fell back to **standby-like power draw**, the behavior matched a **PSU protection latch**.

That pushed the investigation toward PSU rail behavior rather than drivers or kernels.

---

## Fix: Enable OCK / single-rail mode on Dark Power Pro 13

### Result
After enabling **Overclocking Key (OCK)** / **single-rail mode**, the system stopped hard-powering-off and could sustain near-maximum GPU load (close to **600W**) for extended periods.

### Why it works (high-level)
Dark Power Pro 13 defaults to **multi-rail** 12V operation. A single connector/rail can hit an **over-current protection (OCP)** threshold during fast transient spikes, even if average power is “within spec”.

Single-rail mode effectively removes the per-rail limit and allows the PSU to serve short transients from the full 12V capacity without tripping.

### How to enable OCK safely
1. Shut down the system.
2. Flip the PSU switch to **0** and remove AC power.
3. Enable **OCK** either by:
   - installing the **OCK key/jumper** on the PSU (depending on your kit), or
   - using the included **OCK switch bracket** if you installed it
4. Restore AC power, turn PSU back on, boot, and retest.

> Important: only toggle OCK while the PSU is fully off (per the PSU documentation).

---

## Why the printed rail specs can still be misleading

Even if a given rail is labeled for a high current (e.g., “55A”), OCP may still trip because:

- OCP is about **current**, not “GPU-reported watts”
- transient spikes can happen on very short timescales (ms/sub-ms)
- software power limits often control **averages**, not instantaneous spikes
- modern high-power GPUs can have extremely steep load steps, especially during compute bursts (LLM prefill is a perfect example)

---

## How to recognize this issue quickly (checklist)

If you experience **instant power loss** under GPU compute load, ask:

- Do I need to **toggle the PSU switch / unplug AC** before it will power back on?  
  - If yes: strongly suggests **PSU protection latch**.
- Does it happen on both Linux and Windows?  
  - If yes: less likely to be an OS-only driver issue.
- Does enabling **single-rail / OCK** stop it?  
  - If yes: you likely hit **multi-rail OCP**.

---

## Notes / tradeoffs

Running single-rail mode is a reasonable configuration for extreme GPUs, but it’s still wise to:

- ensure the **12V-2x6 connector is fully seated**
- avoid sharp bends right at the connector
- use the PSU’s **native** cable (no adapters)
- keep airflow good (high sustained loads = heat)

Multi-rail can offer better fault containment for some failure modes; single-rail can be more tolerant of high transient GPU loads. Choose based on your use case and comfort level.

---

## Keywords for search

- Dark Power Pro 13 1600W OCP
- be quiet OCK key single rail
- RTX PRO 6000 Blackwell shutdown under load
- 12VHPWR / 12V-2x6 transient power spike
- “PC powers off and requires PSU switch toggle”
- LLM prompt processing prefill power spike
- multi-rail PSU trips with RTX 5090 / 600W GPUs

---

## Suggested next steps for anyone reproducing this

1. Confirm it’s a latch-off (requires PSU toggle).
2. Enable **OCK / single-rail** mode and retest.
3. If still failing in single-rail:
   - reseat/replace the 12V-2x6 cable,
   - test with a different PSU,
   - test the GPU in another system (to rule out a GPU hardware fault).

---

## Appendix: Minimal reproduction description (example)

- Load a very large model in an LLM runtime that heavily utilizes GPU compute and memory.
- Trigger a long prompt/prefill phase that drives the GPU to very high utilization.
- Observe: system powers off in less then a minute and requires PSU toggle to recover in multi-rail mode. Just holding the power button for over 10 seconds has no effect.
- Enable OCK: workload runs stably for minutes at ~600W GPU draw.

