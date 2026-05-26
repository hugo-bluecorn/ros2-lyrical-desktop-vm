# ROS 2 Lyrical Desktop VM — QEMU/KVM with NVIDIA GPU Passthrough

## Project goal

Create a general-purpose QEMU/KVM virtual machine running Kubuntu 26.04
LTS (Resolute) with ROS 2 Lyrical desktop-full and NVIDIA GPU passthrough
(VFIO). The VM serves as a reusable ROS 2 development environment with
full GPU compute and 3D acceleration via the host's dedicated NVIDIA GPU.

## Host hardware (Dell Precision 3581)

- **CPU**: Intel Raptor Lake-P (6P + 8E cores), VT-d enabled
- **iGPU**: Intel Iris Xe (8086:a7a0 at `00:02.0`) — drives all displays
- **dGPU**: NVIDIA RTX 2000 Ada (AD107GLM, 10de:28b8 at `01:00.0`) —
  MUXless 3D controller, zero display connectors, driver 595.71.05
- **IOMMU group 17**: dGPU alone — clean passthrough, no ACS override
- **Host OS**: Ubuntu 26.04 LTS (Resolute), kernel 7.0.0-15-generic
- **Existing stack**: QEMU/KVM + libvirt + virt-manager already installed
  (from the PincherX-100 Humble runbook project at
  `../pincherx-100-runbook/`)

## Architecture (decided)

- **Guest**: Kubuntu 26.04 LTS desktop (same Resolute base as host),
  installed from the Kubuntu 26.04 desktop ISO with the installer's
  "minimal installation" option.
- **Graphics**: NVIDIA RTX 2000 Ada passed through via VFIO
  (`<hostdev mode='subsystem' type='pci' managed='yes'>`). virtio-gpu
  retained as the display device (MUXless dGPU has no physical display
  outputs). The guest gets both: virtio-gpu for display compositing,
  NVIDIA for 3D/compute. NVIDIA driver installed inside the guest.
- **Display**: SPICE or virtio-gpu console for initial setup; once
  NVIDIA driver is installed in guest, rviz2 and other GL apps use the
  NVIDIA GPU via EGL.
- **Network**: libvirt default NAT (`virbr0`).
- **Disk**: qcow2 on virtio-blk.
- **VM management**: libvirt, domain XML in version control at `vm/`.

## Implementation phases

Phases 1-2 run entirely on virtio-gpu — no NVIDIA card needed, no
host reboot. The host reboot (`prime-select intel`) happens in
Phase 3 only after ROS 2 is installed and a GPU baseline benchmark
is recorded.

1. **Phase 1 — Guest provisioning** — create Kubuntu 26.04 VM
   (4 vCPU, 8 GB RAM, 60 GB virtio disk) with virtio-gpu, UEFI,
   default NAT. Install from the Kubuntu 26.04 desktop ISO using the
   "minimal installation" option. Snapshot the clean install.
2. **Phase 2 — ROS 2 Lyrical install + baseline benchmark** — install
   `ros-lyrical-desktop` and `ros-dev-tools` via the ROS 2 apt
   repository. Verify talker/listener, colcon build. Run
   `glmark2-wayland` on virtio-gpu to record the baseline GPU score
   (the "before" measurement).
3. **Phase 3 — NVIDIA GPU passthrough** — verify IOMMU (no kernel
   params needed on kernel 7.0), `prime-select intel`, reboot (the
   only one), add VFIO `<hostdev>` to domain XML, install NVIDIA
   driver in guest. Handle MUXless vBIOS extraction if needed. Verify
   `nvidia-smi` and PRIME offload OpenGL.
4. **Phase 4 — Verification + performance comparison** — run
   `glmark2-wayland` on NVIDIA and compare to the Phase 2 baseline.
   Launch rviz2 on the NVIDIA GPU. Verify TF2, GPU utilization.
   Document the performance comparison. Final snapshot.

## Key constraints

- **MUXless dGPU has no display outputs.** All four physical connectors
  are wired to the Intel iGPU. The guest needs virtio-gpu for its
  display surface; the NVIDIA GPU provides compute and GL rendering
  but cannot drive a monitor directly. This is the coexistence
  configuration documented in Ubuntu's GPU-virtualization-with-QEMU/KVM
  guide: virtio-gpu (display) + VFIO (compute/3D).
- **`prime-select intel` required before VM start.** The host NVIDIA
  driver holds fds on the dGPU even when the card is RTD3-suspended.
  `managed='yes'` VFIO detach will fail unless the nvidia kernel
  modules are not loaded. `prime-select intel` + reboot achieves this.
  To use NVIDIA on host off-VM: `prime-select on-demand` + reboot.
- **vBIOS extraction may be needed.** MUXless dGPU loads vBIOS via
  ACPI `_ROM`, not PCIe ROM BAR. If the guest NVIDIA driver fails to
  init, dump the host vBIOS and provide it via `<rom file='...'/>` in
  the hostdev block.
- **Snapshots and PCI passthrough interact badly.** Detach PCI devices
  before taking live snapshots; or take disk-only snapshots when VM is
  shut off.
- **rviz2 requires XWayland.** rviz2 has three independent X11 hard
  dependencies (xcb in main.cpp, GLX in rviz_rendering, OGRE 1.x).
  It cannot run native Wayland. Kubuntu 26.04 (KDE Plasma Wayland)
  runs XWayland by default, which satisfies this requirement.

## Working conventions

- **RTFM before instructing.** Fetch and read actual documentation,
  READMEs, or source before recommending commands. Do not guess from
  shallow web search snippets.
- **Be precise, not lazy.** Verify line numbers, file paths, command
  flags, package versions, and URL schemes. Use exact matching.
- **Lead with the primary action.** Optional alternatives, tangents,
  and edge cases are separated under an "Extras" subheading or
  equivalent, not mixed into the main response.
- **Domain XML is canonical.** All libvirt VM definitions live in
  version control as XML. Recreate the VM on any host with
  `virsh define`.
- **Snapshot before destructive operations.** Internal snapshots
  (`virsh snapshot-create-as`) live inside the qcow2 and survive
  across sessions; use them liberally before risky changes.
- **Runbook-first.** Every phase produces a reproducible runbook doc
  in `runbook/`. The user runs commands by hand.
- **Primary upstream sources only.** Citations must come from primary
  upstream docs (Ubuntu wiki, canonical project pages,
  packages.ubuntu.com, libvirt.org, docs.kernel.org). Third-party
  blogs are not citations.
- **Nano + paste-block for file creation.** When instructing file
  creation, use `nano <path>` + paste-ready content block; never
  `cat > path <<'EOF' ... EOF`.
- **Defaults unless deviation is necessary.** Use the tool's default
  path/port/name unless there's a concrete reason to override.

## References

- Ubuntu GPU virtualisation with QEMU/KVM:
  https://ubuntu.com/server/docs/how-to/graphics/gpu-virtualization-with-qemu-kvm/
- Linux kernel VFIO driver API:
  https://docs.kernel.org/driver-api/vfio.html
- libvirt domain format — host device assignment:
  https://libvirt.org/formatdomain.html
- libvirt node device management:
  https://libvirt.org/drvnodedev.html
- ROS 2 Lyrical installation:
  https://docs.ros.org/en/lyrical/Installation.html
- Kubuntu 26.04 download:
  https://kubuntu.org/getkubuntu/
