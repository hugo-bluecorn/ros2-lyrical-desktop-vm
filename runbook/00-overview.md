# ROS 2 Lyrical Desktop VM — Runbook Overview

General-purpose QEMU/KVM virtual machine: Kubuntu 26.04 LTS (Resolute)
with ROS 2 Lyrical desktop-full and NVIDIA RTX 2000 Ada GPU passthrough
via VFIO.

## Phases

| # | Phase | Runbook | Reboot? |
|---|-------|---------|---------|
| 1 | Guest provisioning | `01-guest-provisioning.md` | No |
| 2 | ROS 2 Lyrical install + baseline benchmark | `02-ros2-lyrical-install.md` | No |
| 3 | NVIDIA GPU passthrough | `03-nvidia-gpu-passthrough.md` | Yes (1) |
| 4 | Verification + performance comparison | `04-verification.md` | No |

Phases 1-2 run entirely on virtio-gpu — no NVIDIA card needed. The
host reboot (`prime-select intel`) happens in Phase 3 only after
ROS 2 is installed and a glmark2 baseline is recorded. This maximizes
pre-reboot work and gives a clean before/after GPU comparison.

## Host

- Dell Precision 3581 (Raptor Lake-P)
- Ubuntu 26.04 LTS (Resolute), kernel 7.0.0-15-generic
- Intel Iris Xe iGPU (drives all displays) + NVIDIA RTX 2000 Ada dGPU
  (MUXless, IOMMU group 17, passthrough target)
- QEMU 10.2.1 + libvirt 12.0.0 + virt-manager 5.1.0

## VM domain

Canonical libvirt XML lives at `vm/ros2-lyrical-dev.xml`.

To define or update the VM on any host:

```bash
# If redefining (XML changed), undefine first:
virsh undefine ros2-lyrical-dev --keep-nvram

# Then define:
virsh define vm/ros2-lyrical-dev.xml
```

## Conventions

- Phase docs commit only after verification.
- `CLAUDE.md` is the project-level instruction file.
- See `CLAUDE.md` § Working conventions for runbook authoring rules.
