# NVIDIA Quadro M2200 — Fedora 44 Driver Fix + Ollama GPU Guide

**Hardware:** Lenovo ThinkPad P51 
**GPU:** NVIDIA Quadro M2200 Mobile (GM206GLM, Maxwell architecture) 
**PCI ID:** `10de:1436` 
**OS:** Fedora Server 44 
**Date fixed:** 2026-04-05

---

## The Problem

After installing the stock RPM Fusion `akmod-nvidia` (v595.x) package on Fedora 44, the GPU fails to initialize at boot:

```
NVRM: The NVIDIA GPU 0000:01:00.0 (PCI ID: 10de:1436)
NVRM: installed in this system is not supported by open
NVRM: nvidia.ko because it does not include the required GPU System Processor (GSP).
nvidia 0000:01:00.0: probe with driver nvidia failed with error -1
NVRM: None of the NVIDIA devices were initialized.
```

### Why it happens

NVIDIA driver **595.x and newer** only ships the **open-source kernel module** (`nvidia.ko`). The open module requires a **GPU System Processor (GSP)** which was first introduced in the **Turing architecture (RTX 20xx, 2018)**.

The Quadro M2200 is based on the **Maxwell architecture (GM206, 2015)** — it has no GSP chip. The 595.x open module therefore refuses to bind to it.

---

## GPU Details

| Property | Value |
|---|---|
| Name | NVIDIA Quadro M2200 Mobile |
| Chip | GM206GLM (Maxwell gen 2) |
| PCI ID | `10de:1436` |
| VRAM | 4 GB GDDR5 |
| TDP | 80 W |
| CUDA Compute | 5.2 |
| PCI slot | `0000:01:00.0` |
| Subsystem | Lenovo Device 2251 |

---

## Driver Compatibility Matrix

| Driver Series | Supports Quadro M2200 | Notes |
|---|---|---|
| **595.x (current)** | ❌ No | Open module only; requires GSP (Turing+) |
| **580.x (legacy)** | ✅ Yes | Closed proprietary module; last series supporting Maxwell |
| **470.x (legacy)** | ❌ No | Covers Kepler; Maxwell excluded |
| **390.x (legacy)** | ❌ No | Covers Fermi/Kepler only |

The **580.x series** is the correct and last driver that supports Maxwell GPUs on Fedora 44.

---

## The Fix — Step by Step

### 1. Prerequisites

Ensure RPM Fusion nonfree repo is enabled:

```bash
sudo dnf install -y \
  https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

### 2. Remove the incompatible 595.x driver

```bash
sudo dnf remove -y \
  akmod-nvidia \
  kmod-nvidia \
  xorg-x11-drv-nvidia \
  xorg-x11-drv-nvidia-cuda \
  xorg-x11-drv-nvidia-cuda-libs \
  xorg-x11-drv-nvidia-kmodsrc \
  xorg-x11-drv-nvidia-libs \
  xorg-x11-drv-nvidia-power
```

### 3. Install the 580xx legacy driver

```bash
sudo dnf install -y \
  akmod-nvidia-580xx \
  xorg-x11-drv-nvidia-580xx \
  xorg-x11-drv-nvidia-580xx-cuda \
  xorg-x11-drv-nvidia-580xx-cuda-libs \
  xorg-x11-drv-nvidia-580xx-libs \
  xorg-x11-drv-nvidia-580xx-power \
  nvidia-settings-580xx
```

### 4. Wait for akmods to compile the kernel module

akmods will automatically compile the kernel module for your running kernel. Monitor progress:

```bash
# Watch the build (takes 2–5 minutes depending on CPU)
watch -n5 'pgrep -a rpmbuild || echo "build done"; rpm -qa | grep kmod-nvidia'
```

When you see `kmod-nvidia-580xx-<kernel>-580.142-1.fc44.x86_64` appear, the build is complete.

### 5. Reboot

```bash
sudo reboot
```

### 6. Verify

```bash
# Check driver loaded
lsmod | grep nvidia

# Check GPU recognized
nvidia-smi
```

Expected output from `nvidia-smi`:
```
+-----------------------------------------------------------------------+
| NVIDIA-SMI 580.142    Driver Version: 580.142    CUDA Version: 13.0  |
+-------------------------------+---------------------+----------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A| Volatile Uncorr. ECC |
|   0  Quadro M2200         Off| 00000000:01:00.0 On|              N/A |
|  4096MiB |      0%      Default |
+-----------------------------------------------------------------------+
```

---

## Installed Packages (working state)

```
akmod-nvidia-580xx-580.142-1.fc44.x86_64
kmod-nvidia-580xx-6.19.11-300.fc44.x86_64-580.142-1.fc44.x86_64
nvidia-gpu-firmware-20260309-1.fc44.noarch
nvidia-settings-580xx-580.142-1.fc44.x86_64
xorg-x11-drv-nvidia-580xx-580.142-1.fc44.x86_64
xorg-x11-drv-nvidia-580xx-cuda-580.142-1.fc44.x86_64
xorg-x11-drv-nvidia-580xx-cuda-libs-580.142-1.fc44.x86_64
xorg-x11-drv-nvidia-580xx-kmodsrc-580.142-1.fc44.x86_64
xorg-x11-drv-nvidia-580xx-libs-580.142-1.fc44.x86_64
xorg-x11-drv-nvidia-580xx-power-580.142-1.fc44.x86_64
```

---

## Ollama GPU Support

With the 580xx driver working, [Ollama](https://ollama.com) (v0.19.0 tested) automatically detects and uses the GPU via CUDA.

Ollama journal log confirms GPU detection on successful boot:
```
INFO source=types.go msg="inference compute"
  id=GPU-56bc9262-4f75-fdde-df4d-2c8be1ec2e39
  library=CUDA
  compute=5.2
  name=CUDA0
  description="Quadro M2200"
  driver=13.0
  pci_id=0000:01:00.0
  type=discrete
  total="4.0 GiB"
  available="3.9 GiB"

INFO source=routes.go msg="vram-based default context"
  total_vram="4.0 GiB"
  default_num_ctx=4096
```

### Verify Ollama is using the GPU

```bash
# Check service status
systemctl status ollama

# Check journal for GPU detection line
journalctl -u ollama -n 100 | grep 'inference compute'

# Pull a small model and watch GPU memory rise
ollama pull tinyllama
watch -n1 nvidia-smi
```

### Notes on Maxwell + CUDA compute 5.2

- CUDA compute capability **5.2** is supported by CUDA 12 (required ≥ 3.5)
- Ollama uses `cuda_v12` libraries on this GPU
- 4 GB VRAM fits models up to approximately **3B–4B parameters** at full precision, or **7B** at 4-bit quantization
- For larger models, use `OLLAMA_NUM_GPU=999` and let Ollama automatically offload layers

---

## Troubleshooting

### Driver not loading after reboot

Check dmesg:
```bash
sudo dmesg | grep -i nvidia
```

If you see `GSP` errors, you still have the open module. Run:
```bash
rpm -qa | grep kmod-nvidia
```
Make sure only `kmod-nvidia-580xx-*` appears, not `kmod-nvidia-6.*` (the 595 closed built from old packages).

### akmods fails to build

Check the build log:
```bash
cat /var/cache/akmods/nvidia-580xx/$(uname -r)-x86_64-build_error.log 2>/dev/null
```

Make sure kernel headers are installed:
```bash
sudo dnf install -y kernel-devel-$(uname -r)
```

### Ollama not using GPU (shows CPU only)

Ollama falls back to CPU if CUDA is not available. Check:
```bash
journalctl -u ollama -n 50 | grep 'inference compute'
```

If it says `library=cpu`, the NVIDIA driver is not loaded. Verify with `lsmod | grep nvidia` and `nvidia-smi`.

---

## System Info

| Component | Detail |
|---|---|
| Machine | Lenovo ThinkPad P51 |
| OS | Fedora Server 44 |
| Kernel | 6.19.11-300.fc44.x86_64 |
| Driver | NVIDIA 580.142 (legacy 580xx series) |
| CUDA Version | 13.0 |
| Secure Boot | Disabled |
| Ollama | 0.19.0 |

---

## References

- [RPM Fusion NVIDIA Howto](https://rpmfusion.org/Howto/NVIDIA)
- [NVIDIA Legacy GPU List](https://www.nvidia.com/en-us/drivers/unix/legacy-gpu/)
- [Ollama GitHub](https://github.com/ollama/ollama)
- [NVIDIA open kernel module requirements](https://github.com/NVIDIA/open-gpu-kernel-modules) (requires Turing+)
