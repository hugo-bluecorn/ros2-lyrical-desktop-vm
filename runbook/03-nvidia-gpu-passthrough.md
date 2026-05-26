# 03 — NVIDIA GPU passthrough

Pass the host's NVIDIA RTX 2000 Ada through to the guest via VFIO.
This is the only phase that requires a host reboot.

## Goal

After this phase:

- `prime-select` is set to `intel` on the host, releasing the dGPU.
- The VM boots with both virtio-gpu (display) and the NVIDIA RTX
  2000 Ada (CUDA/compute) via VFIO passthrough.
- `nvidia-smi` works inside the guest.
- The NVIDIA GPU is confirmed available for CUDA and compute
  workloads. GL rendering on the NVIDIA GPU works but is **not
  visible on screen** due to a virtio-gpu DMA-BUF import limitation
  (see Step 9).

## Why VFIO passthrough?

VFIO assigns a physical PCI device directly to a guest VM. The guest
gets bare-metal GPU performance — CUDA, NVENC, and headless
rendering — because it talks to real hardware through its own driver,
not through a paravirtualized shim.

On this laptop (Dell Precision 3581, MUXless hybrid graphics), the
Intel iGPU drives all physical displays. The NVIDIA RTX 2000 Ada has
zero display connectors. Passing the dGPU to a VM has zero impact on
the host's display.

The guest uses **both** GPUs:

- **virtio-gpu** — display surface (SPICE console) and all visible
  GL rendering. Required because the MUXless dGPU has no video
  outputs.
- **NVIDIA RTX 2000 Ada via VFIO** — CUDA compute, NVENC encoding,
  headless rendering. PRIME render offload works (the GPU renders
  correctly) but the results are **not visible on screen** because
  virtio-gpu cannot import DMA-BUFs from the real GPU (see Step 9).

## Why no IOMMU kernel parameters?

IOMMU is auto-detected and functional on this host (kernel 7.0,
Intel Raptor Lake-P, VT-d enabled in firmware). `journalctl -k`
shows `DMAR: IOMMU enabled` and `Enabled IRQ remapping in x2apic
mode` without any boot parameters. Adding `intel_iommu=on iommu=pt`
is a minor host I/O optimization, not a VFIO requirement — see
Extras at the end.

## Sources

- [Ubuntu GPU virtualisation with QEMU/KVM][ubuntu-gpu] — VFIO
  passthrough, `managed='yes'` for automatic driver handling.
- [Linux kernel VFIO driver API][kernel-vfio] — IOMMU group
  isolation model.
- [libvirt domain format — host device assignment][libvirt-hostdev]
  — `<hostdev type='pci' managed='yes'>`, `<driver name='vfio'/>`,
  `<rom>` element.

[ubuntu-gpu]: https://ubuntu.com/server/docs/how-to/graphics/gpu-virtualization-with-qemu-kvm/
[kernel-vfio]: https://docs.kernel.org/driver-api/vfio.html
[libvirt-hostdev]: https://libvirt.org/formatdomain.html

## Prerequisites

- Phase 2 complete: VM has ROS 2 Lyrical desktop installed, glmark2
  baseline recorded.
- Guest VM is **shut down** (not running or suspended).

## Adapt

| Aspect | This runbook | If yours differs |
|--------|-------------|-----------------|
| GPU | NVIDIA RTX 2000 Ada at `01:00.0`, IOMMU group 17 | Substitute your GPU's PCI address and group |
| GPU switching | `prime-select` (Dell/Ubuntu) | `optimus-manager`, `envycontrol`, or manual module blacklisting |
| CPU vendor | Intel (VT-d) | AMD: IOMMU also auto-detected on recent kernels |

---

## Step 1 — Verify IOMMU and group isolation

These checks run on the host before any changes, while still in
`on-demand` GPU mode.

```sh
$ journalctl -k --no-pager | grep -i -E 'dmar|iommu' | head -15
```

**Verify:** lines include `DMAR: IOMMU enabled` and
`DMAR-IR: Enabled IRQ remapping`.

```sh
$ ls /sys/kernel/iommu_groups/17/devices/
$ lspci -nns 01:00.0
```

**Verify:** group 17 contains only `0000:01:00.0`, which is the
`NVIDIA Corporation AD107GLM [RTX 2000 Ada] [10de:28b8]`.

```sh
$ modinfo vfio-pci | head -3
```

**Verify:** module exists at
`/lib/modules/7.0.0-15-generic/kernel/drivers/vfio/pci/`.

## Step 2 — Optional: dump vBIOS

The MUXless dGPU loads its vBIOS via ACPI `_ROM`, not a standard PCI
ROM BAR. If the guest NVIDIA driver fails to initialize in Step 8,
you may need to provide the vBIOS manually. It is easier to dump it
now while the nvidia driver is still loaded.

```sh
$ sudo bash -c 'echo 1 > /sys/bus/pci/devices/0000:01:00.0/rom'
$ sudo cat /sys/bus/pci/devices/0000:01:00.0/rom > /tmp/vbios-rtx2000ada.rom
$ sudo bash -c 'echo 0 > /sys/bus/pci/devices/0000:01:00.0/rom'
$ ls -lh /tmp/vbios-rtx2000ada.rom
```

**Watch out:** if the `echo 1` fails with "Input/output error," the
ROM is not exposed via this interface on your card. In that case, skip
this step — the guest may work without it, and if not, the vBIOS can
be extracted using `nvflash` or from the NVIDIA driver package. Try
Step 8 without the vBIOS first.

If successful, copy to the repo:

```sh
$ cp /tmp/vbios-rtx2000ada.rom vm/vbios-rtx2000ada.rom
```

## Step 3 — Switch to intel-only GPU mode

Release the dGPU from the host NVIDIA driver.

```sh
$ prime-select query
```

**Verify:** currently shows `on-demand`.

```sh
$ sudo prime-select intel
```

**Why:** `prime-select intel` prevents the nvidia kernel modules from
loading on the next boot. The `nouveau` driver is already blacklisted
by the NVIDIA driver package (in
`/lib/modprobe.d/nvidia-graphics-drivers.conf`). The dGPU will be
left unbound — no driver claims it — and libvirt's `managed='yes'`
can bind `vfio-pci` when the VM starts.

## Step 4 — Reboot

This is the **only host reboot** in the entire project.

```sh
$ sudo reboot
```

## Step 5 — Verify post-reboot state

### 5a — prime-select

```sh
$ prime-select query
```

**Verify:** returns `intel`.

### 5b — NVIDIA modules not loaded

```sh
$ lsmod | grep nvidia
```

**Verify:** no output.

### 5c — dGPU unbound

```sh
$ ls -la /sys/bus/pci/devices/0000:01:00.0/driver 2>/dev/null
```

**Verify:** "No such file or directory" — no driver claims the device.

### 5d — IOMMU group still clean

```sh
$ ls /sys/kernel/iommu_groups/17/devices/
```

**Verify:** only `0000:01:00.0`.

## Step 6 — Add VFIO hostdev to domain XML

Edit `vm/ros2-lyrical-dev.xml` to add the NVIDIA GPU passthrough
block. The existing `<video><model type='virtio'/>` stays — it
provides the display surface.

```sh
$ nano vm/ros2-lyrical-dev.xml
```

Add inside the `<devices>` section:

```xml
<hostdev mode='subsystem' type='pci' managed='yes'>
  <driver name='vfio'/>
  <source>
    <address domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
  </source>
</hostdev>
```

If you dumped the vBIOS in Step 2 and need it later, the block
becomes:

```xml
<hostdev mode='subsystem' type='pci' managed='yes'>
  <driver name='vfio'/>
  <source>
    <address domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
  </source>
  <rom file='/path/to/vbios-rtx2000ada.rom'/>
</hostdev>
```

Validate and apply:

```sh
$ xmllint --noout vm/ros2-lyrical-dev.xml
$ virsh undefine ros2-lyrical-dev --keep-nvram
$ virsh define vm/ros2-lyrical-dev.xml
```

**Verify:** `virsh dumpxml ros2-lyrical-dev | grep hostdev` shows the
VFIO hostdev block.

## Step 7 — Boot with GPU passthrough

```sh
$ virsh start ros2-lyrical-dev
$ virt-manager &
```

Open the VM console in virt-manager.

**Verify inside the guest:**

```sh
$ lspci | grep -i nvidia
```

Should show the RTX 2000 Ada on the guest's PCI bus.

**Watch out:** if the VM fails to start, check the host log:

```sh
$ journalctl -u libvirtd --no-pager | tail -30
```

Common issues:
- "cannot open /dev/vfio/17": `vfio-pci` module not loaded — libvirt
  should load it automatically; try `sudo modprobe vfio-pci` and retry.
- "device is busy": nvidia modules still loaded — recheck Step 5b.
- IOMMU errors: verify group isolation (Step 5d).

## Step 8 — Install NVIDIA driver in guest

Inside the guest:

```sh
$ sudo apt-get update && sudo apt-get upgrade -y
$ sudo ubuntu-drivers install
```

**Verify:**

```sh
$ dpkg -l | grep nvidia-driver
```

Expect `nvidia-driver-595-open` (or similar).

Reboot the guest (not the host):

```sh
$ sudo reboot
```

After reboot, verify the driver:

```sh
$ nvidia-smi
```

**Verify:** shows the RTX 2000 Ada with driver version and CUDA
version.

**Watch out:** if `nvidia-smi` fails or the driver shows "ERR!" for
the GPU:
- The vBIOS may be needed. Go back to Step 6 and add the
  `<rom file='...'/>` attribute to the hostdev block.
- If you didn't dump the vBIOS in Step 2, you'll need to temporarily
  restore `prime-select on-demand`, reboot, dump the vBIOS, then
  switch back to `prime-select intel` and reboot again.

## Step 9 — Verify GPU capabilities and display limitation

The guest has two GPUs: virtio-gpu (primary display surface for the
SPICE console) and the NVIDIA RTX 2000 Ada (VFIO passthrough, no
display outputs). GL applications default to virtio-gpu. To target
the NVIDIA GPU, set the PRIME offload environment variables:

```sh
$ __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia <command>
```

### 9a — Install verification tools

```sh
$ sudo apt-get update && sudo apt-get upgrade -y
$ sudo apt-get install -y mesa-utils glmark2-wayland
```

### 9b — Confirm default renderer (virtio-gpu)

```sh
$ glxinfo | grep 'OpenGL renderer'
```

**Verify:** shows `virgl` (the virtio-gpu paravirtualized renderer).

### 9c — Confirm NVIDIA renderer via PRIME offload

```sh
$ __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo | grep 'OpenGL renderer'
```

**Verify:** shows `NVIDIA RTX 2000 Ada Generation Laptop GPU` (or
similar NVIDIA string).

### 9d — glmark2 on NVIDIA (compute verification)

```sh
$ __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glmark2-wayland
```

**Expected behavior:** the benchmark runs and prints scores to the
terminal, but **the glmark2 window is invisible** — no rendered
frames appear on screen.

**Results (2026-05-26):**

| Config | Renderer | Score (800x600) | Window visible |
|--------|----------|-----------------|----------------|
| Default (virtio-gpu) | virgl (Mesa Intel Iris Xe) | 283 | Yes |
| PRIME offload (NVIDIA VFIO) | NVIDIA RTX 2000 Ada | 949 | No |

The NVIDIA GPU renders correctly (3.4x faster than virgl) but the
compositor cannot display the result.

### Why the window is invisible

Under Wayland PRIME render offload, the application renders on the
NVIDIA GPU and exports the framebuffer as a DMA-BUF. The compositor
(KWin) must then import that DMA-BUF onto the display GPU
(virtio-gpu) for compositing. The virtio-gpu kernel driver **does
not support DMA-BUF import** from real GPUs. The app renders
correctly (hence the score), but the compositor silently fails to
composite the result.

This is documented in:

- [Mesa MR !23896][mesa-mr] — "the original virgl doesn't support
  this feature because virtio-gpu driver doesn't support DMA
  operations so iGPU cannot import data from passthrough dGPU
  directly." (A Xen-specific workaround exists but is not merged
  for QEMU/KVM.)
- [KWin MR !3859][kwin-mr] — documents the cross-device buffer
  import path and its limitations.

A kernel patch series by Intel ("drm/virtio: Import scanout buffers
from other devices", [LWN][lwn-scanout]) would fix this, but it is
**not merged** as of kernel 7.0 and was tested only on Intel GPUs.

[Looking Glass][looking-glass] could bypass virtio-gpu entirely via
shared memory, but its Linux guest capture component is
"incomplete and not ready for usage" per the B7 documentation.

[mesa-mr]: https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/23896
[kwin-mr]: https://invent.kde.org/plasma/kwin/-/merge_requests/3859
[lwn-scanout]: https://lwn.net/Articles/998774/
[looking-glass]: https://looking-glass.io/docs/B7/install_host/

### What this means for the project

- **Visible GL apps** (rviz2, glmark2, Gazebo): run on virtio-gpu /
  virgl. This is the only renderer that can display on screen.
- **NVIDIA GPU**: available for **CUDA compute, NVENC encoding, and
  headless rendering**. Fully functional for non-display workloads.
- **rviz2**: renders on virgl (score 283 tier), not on the NVIDIA
  GPU. This is a hardware topology limitation, not a configuration
  issue.

## Step 10 — Snapshot

Shut down the guest:

```sh
$ sudo shutdown -h now
```

On the host:

```sh
$ sudo cp /var/lib/libvirt/images/ros2-lyrical-dev.qcow2 \
       /var/lib/libvirt/images/ros2-lyrical-dev.nvidia-passthrough.qcow2
```

---

## Done — Phase 3 exit criteria

- [ ] `prime-select query` returns `intel` on the host
- [ ] `lsmod | grep nvidia` returns no output on the host
- [ ] VM boots with VFIO passthrough (no start errors)
- [ ] `lspci` in guest shows the NVIDIA RTX 2000 Ada
- [ ] `nvidia-smi` works in guest — shows driver + CUDA version
- [ ] `glxinfo` with PRIME offload shows NVIDIA renderer string
- [ ] `glmark2-wayland` with PRIME offload scores on NVIDIA GPU
      (window invisible — DMA-BUF limitation documented in Step 9)
- [ ] Canonical XML at `vm/ros2-lyrical-dev.xml` includes the
      `<hostdev>` block
- [ ] Disk backup `ros2-lyrical-dev.nvidia-passthrough.qcow2` exists

## Extras

### Restoring host NVIDIA access

When the VM is not in use and you want the dGPU back on the host:

```sh
$ sudo prime-select on-demand
$ sudo reboot
```

After reboot, `nvidia-smi` works on the host again.

### Why not static `vfio-pci.ids=` binding?

Adding `vfio-pci.ids=10de:28b8` to the kernel command line makes
`vfio-pci` claim the device at boot. Drawback: the dGPU is
permanently bound to VFIO even when no VM runs. `managed='yes'` is
more flexible — libvirt binds `vfio-pci` at VM start and unbinds at
VM stop.

### Why not runtime nvidia unloading?

The NVIDIA driver package enables DRM kernel mode setting
(`nvidia_drm modeset=1`). With modeset active, the DRM/KMS subsystem
holds a permanent reference on `nvidia_drm` (refcnt 2+). `rmmod`
cannot unload a module with a non-zero reference count. The only
reliable release path is `prime-select intel` + reboot.

### Optional: IOMMU passthrough mode

For a minor host I/O performance benefit:

```sh
$ sudo nano /etc/default/grub
```

Change `GRUB_CMDLINE_LINUX_DEFAULT` to include `iommu=pt`:

```
GRUB_CMDLINE_LINUX_DEFAULT='quiet splash iommu=pt'
```

Then `sudo update-grub` and reboot. This is **not required** for
VFIO — it reduces IOMMU overhead for host-side DMA (NVMe, NIC).

## Next

[Phase 4 — Verification](04-verification.md)
