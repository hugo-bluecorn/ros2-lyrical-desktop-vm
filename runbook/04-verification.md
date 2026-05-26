# 04 — Verification

Confirm the full ROS 2 Lyrical desktop stack works end-to-end in the
VM. Verify rviz2 renders on virtio-gpu, TF2 works, and CUDA compute
is available on the NVIDIA GPU.

## Goal

After this phase:

- rviz2 renders on virtio-gpu (virgl) inside the VM.
- The full ROS 2 desktop stack (TF2, robot_state_publisher) is
  verified.
- CUDA availability on the NVIDIA GPU is confirmed.
- GPU performance comparison (virtio-gpu vs NVIDIA VFIO) is
  documented.
- Final disk backup taken.

## Context — display limitation

Phase 3 Step 9 established that PRIME render offload to the NVIDIA
GPU works (glmark2 score 949) but the rendered frames are invisible
on screen because virtio-gpu cannot import DMA-BUFs from the real
GPU. All visible GL applications — including rviz2 — render on
virtio-gpu / virgl (score 283). The NVIDIA GPU is available for
CUDA compute and headless rendering only.

## Prerequisites

- Phase 3 complete: VM boots with NVIDIA VFIO passthrough,
  `nvidia-smi` works, display limitation documented.

---

## Step 1 — rviz2 on virtio-gpu

Launch rviz2 (renders on the default virtio-gpu / virgl):

```sh
$ rviz2
```

**Verify:**

- rviz2 opens and renders the 3D viewport.
- The window is visible and interactive.

Check the renderer in another terminal:

```sh
$ glxinfo | grep 'OpenGL renderer'
```

**Verify:** shows `virgl` — this is the display-capable renderer.

## Step 2 — TF2 / robot_state_publisher

Verify the full ROS 2 desktop stack beyond ros-base:

```sh
$ ros2 run tf2_ros static_transform_publisher --frame-id world --child-frame-id base_link
```

In another terminal:

```sh
$ ros2 topic echo /tf_static
```

**Verify:** the static transform is published and received.

## Step 3 — CUDA compute verification

Confirm the NVIDIA GPU is available for compute workloads:

```sh
$ nvidia-smi
```

**Verify:** shows the RTX 2000 Ada, driver version, and CUDA
version (13.2).

For a quick CUDA compute test (if CUDA toolkit is installed):

```sh
$ sudo apt-get install -y nvidia-cuda-toolkit
$ nvcc --version
```

## Step 4 — Document performance summary

```sh
$ nano ~/gpu-performance-summary.txt
```

Paste:

```
GPU Performance Summary — ros2-lyrical-dev VM
=============================================

Date: 2026-05-26
Host: Dell Precision 3581, Ubuntu 26.04, kernel 7.0.0-15-generic

Display GPU: virtio-gpu / virgl (Mesa Intel Iris Xe via host iGPU)
  glmark2-wayland score (800x600):  283
  glmark2-wayland score (1920x1080): 217
  All visible GL apps render here (rviz2, Gazebo, etc.)

Compute GPU: NVIDIA RTX 2000 Ada via VFIO passthrough
  glmark2-wayland score (800x600):  949  (3.4x faster, but invisible)
  Driver: 595.71.05
  CUDA: 13.2
  VRAM: 8192 MiB
  Available for: CUDA, NVENC, headless rendering

Limitation: virtio-gpu cannot import DMA-BUFs from the NVIDIA GPU.
PRIME render offload works (GPU renders correctly) but the
compositor cannot display the result. This is a kernel-level gap
(virtio-gpu driver lacks cross-device DMA import). An unmerged
patch series (drm/virtio: Import scanout buffers) would fix this
but is not in mainline as of kernel 7.0.
```

## Step 5 — Final snapshot

Shut down the guest:

```sh
$ sudo shutdown -h now
```

On the host:

```sh
$ sudo cp /var/lib/libvirt/images/ros2-lyrical-dev.qcow2 /var/lib/libvirt/images/ros2-lyrical-dev.phase4-verified.qcow2
```

---

## Done — Phase 4 exit criteria

- [ ] rviz2 renders on virtio-gpu (virgl) — window visible and
      interactive
- [ ] TF2 static_transform_publisher works
- [ ] `nvidia-smi` shows NVIDIA GPU with CUDA available
- [ ] Performance summary documented in
      `~/gpu-performance-summary.txt`
- [ ] Final disk backup
      `ros2-lyrical-dev.phase4-verified.qcow2` exists

## Project complete

All four phases are done. The VM is a general-purpose ROS 2 Lyrical
development environment with:

- **Display / GL**: virtio-gpu / virgl (rviz2, Gazebo, etc.)
- **Compute**: NVIDIA RTX 2000 Ada via VFIO (CUDA 13.2, 8 GB VRAM)

To start the VM in future sessions:

1. Ensure `prime-select query` returns `intel` on the host
   (if not, `sudo prime-select intel` + reboot).
2. `virsh start ros2-lyrical-dev`
3. Open the console: `virt-manager &`

To restore host NVIDIA access when done:

```sh
$ virsh shutdown ros2-lyrical-dev
$ sudo prime-select on-demand
$ sudo reboot
```
