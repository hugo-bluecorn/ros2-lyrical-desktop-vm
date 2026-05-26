# 01 — Guest provisioning

Create a Kubuntu 26.04 LTS (Resolute) VM in QEMU/KVM. This phase uses
virtio-gpu only — no NVIDIA passthrough yet. The GPU passthrough is
added in Phase 3 after ROS 2 is installed and a baseline benchmark is
recorded.

## Goal

After this phase:

- A Kubuntu 26.04 desktop VM (`ros2-lyrical-dev`) runs on
  virtio-gpu with virgl3D acceleration via the host's Intel iGPU.
- `spice-vdagent` is installed and working (clipboard sharing,
  automatic display resize).
- Network connectivity confirmed via libvirt's default NAT.
- Canonical domain XML is version-controlled at
  `vm/ros2-lyrical-dev.xml`.
- A clean-install disk backup exists for rollback.

## Sources

- [Kubuntu 26.04 release page][kubuntu-iso] — ISO download and
  SHA256 checksum.
- [Ubuntu virt-manager how-to][ubuntu-virtmgr] — GUI-based VM
  creation with virt-manager.
- [Ubuntu GPU virtualisation with QEMU/KVM][ubuntu-gpu] — virgl3D
  approach for virtio-gpu: "Use `-vga virtio` with a local display
  having a GL context."

[kubuntu-iso]: https://cdimage.ubuntu.com/kubuntu/releases/26.04/release/
[ubuntu-virtmgr]: https://ubuntu.com/server/docs/how-to/virtualisation/virtual-machine-manager/
[ubuntu-gpu]: https://ubuntu.com/server/docs/how-to/graphics/gpu-virtualization-with-qemu-kvm/

## Prerequisites

**Software (already installed from sibling project):**
- `qemu-system-x86` 10.2.1
- `libvirt-daemon-system` 12.0.0
- `virt-manager` 5.1.0
- `ovmf` (UEFI firmware)

**Groups:**
- Your user in `libvirt` and `kvm` groups
- `libvirt-qemu` in the `render` group (for virtio-gpu EGL)

**Network:**
- libvirt default NAT network active (`virsh net-list --all`)

## Adapt

| Aspect | This runbook | If yours differs |
|--------|-------------|-----------------|
| Guest distro | Kubuntu 26.04 (Resolute) | Substitute your ISO and OS variant |
| Disk size | 60 GB | Adjust if you need more for workspaces |
| RAM | 8 GB | 4 GB minimum for desktop; 16 GB if your host has enough |
| vCPU | 4 | Match your host core count minus headroom |

---

## Step 1 — Download and verify the Kubuntu 26.04 ISO

```sh
$ cd /var/lib/libvirt/images/
$ sudo wget https://cdimage.ubuntu.com/kubuntu/releases/26.04/release/kubuntu-26.04-desktop-amd64.iso
$ sudo wget https://cdimage.ubuntu.com/kubuntu/releases/26.04/release/SHA256SUMS
$ sha256sum -c SHA256SUMS 2>/dev/null | grep kubuntu-26.04-desktop-amd64.iso
```

**Verify:** output shows `kubuntu-26.04-desktop-amd64.iso: OK`.

## Step 2 — Create the qcow2 disk image

Pre-creating the disk (rather than letting virt-install allocate it)
lets us pin the cluster size and lazy refcounts.

```sh
$ sudo qemu-img create -f qcow2 \
       -o cluster_size=65536,lazy_refcounts=on \
       /var/lib/libvirt/images/ros2-lyrical-dev.qcow2 60G
```

**Verify:**

```sh
$ sudo qemu-img info /var/lib/libvirt/images/ros2-lyrical-dev.qcow2
```

Shows `file format: qcow2`, `virtual size: 60 GiB`,
`cluster_size: 65536`.

## Step 3 — Create and start the VM

Use virt-manager for the graphical installer. Open virt-manager:

```sh
$ virt-manager &
```

Create a new VM with these settings:

1. **Connection:** QEMU/KVM
2. **Installation media:** Local install media →
   `/var/lib/libvirt/images/kubuntu-26.04-desktop-amd64.iso`
3. **OS type:** if Kubuntu 26.04 is not listed, select
   `Ubuntu 24.04 LTS` (`ubuntu-lts-latest`) — the closest available
   variant. The OS type mainly affects default device choices; virtio
   support is the same.
4. **Memory:** 8192 MB
5. **CPUs:** 4
6. **Storage:** select existing disk →
   `/var/lib/libvirt/images/ros2-lyrical-dev.qcow2`
7. **Name:** `ros2-lyrical-dev`
8. Check **Customize configuration before install**
9. In the configuration window:
   - **Overview → Firmware:** select UEFI (OVMF)
   - **Video → Model:** Virtio (this gives virtio-gpu)
   - **Display → Type:** Spice, with `Listen type: None`
     and `OpenGL: ✓` (enables virgl3D)
10. Click **Begin Installation**

**Watch out:** if virt-manager does not offer OpenGL in the display
settings, ensure `libvirt-qemu` is in the `render` group (Phase 1
prerequisite) and that libvirtd was restarted after the group change.

## Step 4 — Walk through the Kubuntu installer

The Kubuntu 26.04 installer (Calamares) opens in the VM console.

1. Language: your preference
2. Location / timezone: your timezone
3. Keyboard: your layout
4. **Installation type:** select **Minimal installation**
5. Partitioning: **Erase disk** (the 60 GB virtio disk)
6. User account: set username, hostname, password
7. Wait for install to complete
8. Click **Restart Now** when prompted
9. The VM reboots into the installed Kubuntu desktop

**Watch out:** if the VM hangs at "Remove installation medium and
press Enter," just close the console window and force-off the VM
(`virsh destroy ros2-lyrical-dev`), then remove the CDROM from the
VM configuration and start again. virt-manager should handle this
automatically in most cases.

## Step 5 — Post-install housekeeping

Inside the guest:

```sh
$ sudo apt-get update && sudo apt-get upgrade -y
$ sudo apt install spice-vdagent
```

**Verify:**

- Resize the virt-manager console window — the guest desktop should
  resize to match (spice-vdagent working).
- Test clipboard: copy text on the host, paste inside the guest.

Verify network:

```sh
$ ping -c1 archive.ubuntu.com
```

**Verify:** ping succeeds (DNS resolution + connectivity via NAT).

## Step 6 — Export canonical domain XML

Shut down the guest cleanly:

```sh
$ sudo shutdown -h now
```

On the host, dump the current XML and save it to the repo:

```sh
$ virsh dumpxml ros2-lyrical-dev > /tmp/ros2-lyrical-dev-raw.xml
```

Clean up the XML before committing:

- Remove `<uuid>` (for cross-host portability)
- Remove `<mac address='...'/>` (libvirt generates a new one on
  define)
- Remove any auto-generated `<address>` elements inside devices
  (libvirt recalculates these)
- Remove the CDROM/ISO entry if still present

Save the cleaned XML:

```sh
$ nano vm/ros2-lyrical-dev.xml
```

Paste the cleaned XML content. Validate:

```sh
$ xmllint --noout vm/ros2-lyrical-dev.xml
```

**Verify:** `xmllint` exits silently (no errors).

Test the round-trip:

```sh
$ virsh undefine ros2-lyrical-dev --keep-nvram
$ virsh define vm/ros2-lyrical-dev.xml
$ virsh start ros2-lyrical-dev
```

**Verify:** VM boots from disk into the Kubuntu desktop.

Shut down again for the backup step:

```sh
$ virsh shutdown ros2-lyrical-dev
```

## Step 7 — Backup clean install

Create a disk-only backup of the clean install state.

```sh
$ sudo cp /var/lib/libvirt/images/ros2-lyrical-dev.qcow2 \
       /var/lib/libvirt/images/ros2-lyrical-dev.clean-install.qcow2
$ sudo cp /var/lib/libvirt/qemu/nvram/ros2-lyrical-dev_VARS.fd \
       /var/lib/libvirt/qemu/nvram/ros2-lyrical-dev_VARS.clean-install.fd
```

**Verify:** both backup files exist:

```sh
$ ls -lh /var/lib/libvirt/images/ros2-lyrical-dev.clean-install.qcow2
$ ls -lh /var/lib/libvirt/qemu/nvram/ros2-lyrical-dev_VARS.clean-install.fd
```

**Watch out:** the NVRAM filename depends on what libvirt chose when
creating the VM. Check with `virsh dumpxml ros2-lyrical-dev | grep nvram`
to find the actual path.

---

## Done — Phase 1 exit criteria

- [ ] VM `ros2-lyrical-dev` boots to Kubuntu 26.04 desktop
- [ ] `spice-vdagent` installed and working (resize + clipboard)
- [ ] `ping -c1 archive.ubuntu.com` succeeds from inside the guest
- [ ] Canonical XML at `vm/ros2-lyrical-dev.xml` validates with
      `xmllint` and round-trips through `virsh undefine` + `define`
- [ ] Clean-install disk backup exists at
      `ros2-lyrical-dev.clean-install.qcow2`

## Next

[Phase 2 — ROS 2 Lyrical install + baseline benchmark](02-ros2-lyrical-install.md)
