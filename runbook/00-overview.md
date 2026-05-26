# ROS 2 Lyrical Desktop VM — Runbook Overview

General-purpose QEMU/KVM virtual machine: Kubuntu 26.04 LTS (Resolute)
with ROS 2 Lyrical desktop-full and NVIDIA RTX 2000 Ada GPU passthrough
via VFIO.

## Phases

| # | Phase | Runbook | Status |
|---|-------|---------|--------|
| 1 | Host VFIO preparation | `01-host-vfio-preparation.md` | Not started |
| 2 | Guest provisioning with GPU passthrough | `02-guest-provisioning.md` | Not started |
| 3 | ROS 2 Lyrical desktop-full install | `03-ros2-lyrical-install.md` | Not started |
| 4 | Verification | `04-verification.md` | Not started |

## Host

- Dell Precision 3581 (Raptor Lake-P)
- Ubuntu 26.04 LTS (Resolute), kernel 7.0.0-15-generic
- Intel Iris Xe iGPU (drives all displays) + NVIDIA RTX 2000 Ada dGPU
  (MUXless, IOMMU group 17, passthrough target)
- QEMU/KVM + libvirt + virt-manager already installed

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
