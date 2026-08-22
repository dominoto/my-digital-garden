---
{"dg-publish":true,"dg-path":"Resources/Fixing Intel N100 hardware transcoding in Jellyfin.md","permalink":"/resources/fixing-intel-n100-hardware-transcoding-in-jellyfin/","title":"Fixing Intel N100 hardware transcoding in Jellyfin (GuC/HuC firmware)","created":"2026-08-21","updated":"2026-08-22","dg-note-properties":{"type":["[[Notes/Article]]"],"topics":["[[Notes/Homelab]]"],"title":"Fixing Intel N100 hardware transcoding in Jellyfin (GuC/HuC firmware)","created":"2026-08-21","modified":"2026-08-22","author":["Domagoj Zubac"]}}
---


A debugging session that started as "can I squeeze a bit more out of QuickSync?" and turned out to be "my iGPU has been completely dead this whole time."

**TL;DR:** On Debian 13 (Trixie) the i915 firmware blobs moved out of `firmware-misc-nonfree` into a new package called **`firmware-intel-graphics`**. Without them, i915 fails to load GuC firmware and *wedges the GPU entirely* on Alder Lake-N. No hardware decode, no hardware encode, silent fallback to software.

---

## The symptom

```console
$ sudo cat /sys/kernel/debug/dri/0/gt*/uc/guc_info
GuC firmware: i915/tgl_guc_70.bin
        status: MISSING
        version: wanted 70.12.1, found 0.0.0
        uCode: 0 bytes

$ sudo cat /sys/kernel/debug/dri/0/gt*/uc/huc_info
HuC firmware: i915/tgl_huc.bin
        status: ERROR
```

`MISSING` with `0 bytes` means the firmware file isn't on disk. HuC shows `ERROR` as a knock-on effect: it can't be authenticated without GuC.

The kernel log tells the fuller story:

```
i915 0000:00:02.0: firmware: failed to load i915/tgl_guc_70.bin (-2)
i915 0000:00:02.0: [drm] *ERROR* GT0: GuC initialization failed -ENOENT
i915 0000:00:02.0: [drm] *ERROR* GT0: Enabling uc failed (-5)
i915 0000:00:02.0: [drm] *ERROR* GT0: Failed to initialize GPU, declaring it wedged!
```

That last line is the important one. On modern kernels GuC submission isn't optional for Alder Lake-N. If the firmware fetch fails, i915 gives up on the GPU completely.

### Why `tgl_` on an N100?

Alder Lake-N reuses the Tiger Lake (Gen12) GuC/HuC blobs, so the filenames look wrong but aren't. The kernel correctly identifies the hardware:

```
i915 0000:00:02.0: [drm] Found alderlake_p/alderlake_n (device ID 46d1)
```

---

## The fix

### 1. Check whether the firmware exists at all

```bash
ls -l /usr/lib/firmware/i915/ | grep -E 'tgl_(guc|huc)|adlp_dmc'
journalctl -k | grep -iE 'i915|guc|huc'
```

If the directory doesn't exist, the package isn't installed.

### 2. Install the right package

This is the part that trips people up. Guides written before 2025 tell you to install `firmware-misc-nonfree`. As of the 20250410 release, Debian split the Intel GPU blobs into their own package:

```bash
sudo apt install firmware-intel-graphics
```

Make sure `non-free-firmware` is in your sources:

```bash
grep -r non-free-firmware /etc/apt/sources.list /etc/apt/sources.list.d/
```

You want a line like:

```
deb http://deb.debian.org/debian trixie main contrib non-free non-free-firmware
```

### 3. Rebuild initramfs and reboot

```bash
sudo update-initramfs -u
sudo reboot
```

### 4. Verify

```bash
sudo cat /sys/kernel/debug/dri/0/gt0/uc/guc_info | head -5
sudo cat /sys/kernel/debug/dri/0/gt0/uc/huc_info | head -5
journalctl -k | grep -iE 'guc|huc|wedged'
```

A healthy boot looks like:

```
[drm] GT0: GuC firmware i915/tgl_guc_70.bin version 70.49.4
[drm] GT0: HuC firmware i915/tgl_huc.bin version 7.9.3
[drm] GT0: HuC: authenticated for all workloads
[drm] GT0: GUC: submission enabled
[drm] GT0: GUC: SLPC enabled
[drm] GT0: GUC: RC enabled
```

No `wedged` line. `SLPC` and `RC` are GuC-managed power and render-clock control, nice side benefits.

> **Note:** `firmware-intel-graphics` also carries `adlp_dmc.bin` (display microcontroller). Missing DMC disables runtime power management, which matters for idle wattage on an always-on box.

---

## What GuC and HuC actually do

| | Role | Why you care |
|---|---|---|
| **GuC** | Graphics micro-controller: command submission scheduling, power management (SLPC) | On ADL-N it's *mandatory*; without it the GPU won't initialise |
| **HuC** | HEVC/H.264 micro-controller: hardware bitrate control for the low-power encode path | Enables `-low_power 1` and `-mbbrc 1`; without it, VBR silently degrades toward constant-QP |
| **DMC** | Display micro-controller | Runtime power management for the display engine |

HuC is the one that affects encode *quality*. GuC is the one that affects whether anything works at all.

---

## Container setup (Docker / Cosmos)

Host-side `vainfo` is only a diagnostic. The container ships its own ffmpeg and VA-API driver, so what matters is:

```bash
ls -l /dev/dri/
docker exec <container> vainfo
```

You need:

- `/dev/dri` passed into the container
- The container user in the `render` group. **The GID often differs between host and image**, which is the classic reason hardware transcoding stays broken when the host looks perfect

### OpenCL for tone mapping

The LinuxServer Jellyfin image doesn't include an OpenCL runtime by default. Without it, ffmpeg fails *before processing a single frame* if Jellyfin requests an OpenCL device:

```
Failed to get number of OpenCL platforms: -1001.
Failed to set value 'opencl=ocl@va' for option 'init_hw_device': No such device
Error parsing global options: No such device
```

Fix by adding the Docker mod:

```
DOCKER_MODS=linuxserver/mods:jellyfin-opencl-intel
```

Then recreate the container.

---

## Jellyfin settings that matter

**Dashboard → Playback → Transcoding**

| Setting | Value | Notes |
|---|---|---|
| Hardware acceleration | Intel QuickSync (QSV) | |
| Enable hardware encoding | ✅ | |
| Intel Low-Power H.264 encoder | ✅ | Safe once HuC is authenticated |
| Intel Low-Power HEVC encoder | ✅ | Gen12 VDEnc HEVC supports B-frames |
| Enable Tone mapping | ✅ | OpenCL path; needs the Docker mod above |
| Enable VPP Tone mapping | ❌ | Fixed-function fallback; takes precedence if both are on |
| Tone mapping algorithm | BT.2390 | Recommended default |
| Tone mapping peak | 100 | Means 1000 nits. Describes the **source**, not your display |
| Encoding preset | `slower` | Fixed-function encoder; costs almost nothing on QSV |
| Segment length | 3 s | See below |

### Don't set segment length to 1

HLS needs a keyframe at every segment boundary, so Jellyfin derives the GOP from segment length. At 24 fps:

- `-hls_time 3` → `-g 72`, sensible
- `-hls_time 1` → `-g 24`, a keyframe **every second**, which eats a large fraction of a constrained bitrate budget and shows up as softness and blocking in motion

---

## Reading a healthy transcode log

```
-init_hw_device vaapi=va:,vendor_id=0x8086,driver=iHD
-init_hw_device qsv=qs@va
-init_hw_device opencl=ocl@va
-hwaccel vaapi -hwaccel_output_format vaapi
-codec:v:0 hevc_qsv -low_power 1 -preset slower -mbbrc 1
-vf "scale_vaapi=w=1280:h=720,
     hwmap=derive_device=opencl:mode=read,
     tonemap_opencl=format=nv12:p=bt709:tonemap=bt2390:peak=100:desat=0,
     hwmap=derive_device=qsv:mode=write:reverse=1,
     format=qsv"
```

What to look for:

- **`-low_power 1`**: the VDEnc path is active, HuC is doing rate control
- **`hwaccel_output_format vaapi`**: decoded frames stay in GPU memory
- **`hwmap`** chains: zero-copy handoffs between VAAPI, OpenCL and QSV contexts. Not memory copies
- **`libx264` / `libx265` anywhere**: hardware failed, you're on the CPU
- **`q=-0.0`** in the progress lines is normal for QSV; the driver doesn't report a quantiser back

### Watching it live

```bash
sudo apt install intel-gpu-tools
sudo intel_gpu_top
```

The **Video** engine (decode/encode) and **VideoEnhance** (scaling) should be busy; Render/3D stays near idle unless OpenCL tone mapping is in the chain.

---

## Results

Intel N100, 1080p10 HDR10 HEVC → 720p SDR HEVC with tone mapping, audio downmixed 5.1 → stereo:

| Pipeline | Throughput |
|---|---|
| GPU wedged (software fallback) | unusable |
| VPP tone mapping | ~246 fps / **10.2x** realtime |
| OpenCL BT.2390 tone mapping | ~252 fps / **10.3x** realtime |

The OpenCL path is *not* slower here, because tone mapping runs **after** the downscale, so it's shading 1280×720, not 1920×1080. You get the better curve essentially for free, and Jellyfin drops the `procamp_vaapi=b=16` brightness fudge it uses to compensate for VPP's cruder curve.

---

## Bonus: when does the GPU actually get used?

| Mode | What happens | GPU |
|---|---|---|
| **Direct play** | Client handles the file as-is; server streams bytes | None |
| **Direct stream** | Container remuxed (MKV → MP4), streams untouched | None |
| **Transcode** | Decode → filter → encode | Yes |

Things that quietly force a transcode:

- Image-based subtitles (PGS/VOBSUB) needing burn-in
- HDR → SDR tone mapping for an SDR client
- A client bitrate cap below the source bitrate
- Unsupported audio codec (transcodes audio only; CPU work, not GPU)

Also GPU-accelerated outside playback: **trickplay/BIF thumbnail generation** and chapter image extraction. Both hammer the GPU during library scans.

---

## Audio: the downmix rabbit hole

Separate from video, but it came up alongside.

**Audio boost when downmixing** and **Stereo Downmix Algorithm** only apply when ffmpeg is actually downmixing multichannel to stereo. On a stereo source that's direct played or `-codec:a copy`'d, they do nothing.

Check the source first. If it's already `stereo`, no setting on that page will help and the quietness is just how the file was mastered.

When a downmix *is* happening, the algorithm choice matters:

| Algorithm | Behaviour |
|---|---|
| **None** | ffmpeg defaults; centre ≈ -3 dB |
| **Dave750** | `c0 = 0.5*FC + 0.707*FL + 0.707*BL + 0.5*LFE`, centre weighted **below** fronts/surrounds. Preserves bass and space, does *not* help dialogue |
| **NightmodeDialogue** | Centre pushed forward, surrounds attenuated. Best speech intelligibility, flatter mix |
| **RFC7845** | Ogg Opus spec matrix; close to None |
| **AC-4** | Dolby's coefficients |

If dialogue is getting lost, reach for **NightmodeDialogue** before raising the boost, because boost amplifies the loud peaks too.

### Why film audio feels quiet vs. PC content

Film and TV are mastered with wide dynamic range around a reference level. Almost everything on a PC (YouTube, web players, music) is loudness-normalised to roughly -14 LUFS and heavily compressed. Same volume slider, very different result.

Jellyfin's LUFS normalisation covers music libraries only, not video. Real fixes live downstream: your TV's Auto Volume / DRC, or forcing the downmix to the server (set client max audio channels to 2) so ffmpeg does it properly instead of the TV.

> If your TV feeds an external stereo DAC over optical, note that optical is 2ch PCM or bitstream DD only. A stereo PCM DAC can't decode bitstream, so the TV must be set to **PCM**, meaning *the TV* is doing the 5.1 downmix unless you force it server-side.

---

## Gotchas checklist

- [ ] `firmware-intel-graphics`, **not** `firmware-misc-nonfree` (Debian 13+)
- [ ] `non-free-firmware` component enabled in apt sources
- [ ] `update-initramfs -u` then **reboot**, since firmware is fetched at i915 init
- [ ] `/dev/dri` passed into the container, render GID matched
- [ ] OpenCL Docker mod installed if using OpenCL tone mapping
- [ ] Don't enable both VPP and OpenCL tone mapping; VPP wins and OpenCL sits idle
- [ ] Segment length ≥ 3 s
- [ ] Check whether audio is even being downmixed before touching downmix settings

---

If this guide saved you some time, you can say thanks with a coffee:

<a href="https://ko-fi.com/dominoto"><img src="https://storage.ko-fi.com/cdn/kofi2.png?v=3" width="200" alt="Buy me a coffee at ko-fi.com"></a>

## References

- Kernel firmware source: <https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/tree/i915>
- Debian firmware wiki: <https://wiki.debian.org/Firmware>
- Jellyfin hardware acceleration docs: <https://jellyfin.org/docs/general/administration/hardware-acceleration/>
