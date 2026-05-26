# 04 — Verification + performance comparison

Confirm that ROS 2 Lyrical works end-to-end with NVIDIA GPU
acceleration. Quantify the GPU improvement over the Phase 2
virtio-gpu baseline using `glmark2-wayland`.

## Goal

After this phase:

- rviz2 renders on the NVIDIA GPU inside the VM.
- `glmark2-wayland` score on NVIDIA is recorded and compared to the
  Phase 2 virtio-gpu baseline.
- The full ROS 2 desktop stack (TF2, robot_state_publisher) is
  verified.
- Final disk backup taken.

## Prerequisites

- Phase 3 complete: VM boots with NVIDIA VFIO passthrough,
  `nvidia-smi` works, PRIME offload verified.
- Phase 2 glmark2 baseline saved at
  `~/glmark2-baseline-virtio-gpu.txt` inside the guest.

---

## Step 1 — glmark2 with NVIDIA (the "after" measurement)

Inside the guest, run `glmark2-wayland` with PRIME offload to target
the NVIDIA GPU:

```sh
$ __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia \
    glmark2-wayland 2>&1 | tee ~/glmark2-nvidia-vfio.txt
```

**Record:**

1. The **OpenGL renderer string** (expect `NVIDIA RTX 2000 Ada` or
   similar)
2. The **composite score**

Compare against the Phase 2 baseline:

```sh
$ echo "=== Baseline (virtio-gpu) ==="
$ grep -E 'GL_RENDERER|Score' ~/glmark2-baseline-virtio-gpu.txt
$ echo ""
$ echo "=== NVIDIA VFIO ==="
$ grep -E 'GL_RENDERER|Score' ~/glmark2-nvidia-vfio.txt
```

**Watch out:** if `glmark2-wayland` with PRIME offload falls back to
virgl, the NVIDIA EGL/GLX integration may not be picking up the
Wayland display. Try `glmark2-x11` with PRIME offload as a fallback:

```sh
$ sudo apt install glmark2-x11
$ __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia \
    glmark2-x11 2>&1 | tee ~/glmark2-nvidia-vfio-x11.txt
```

## Step 2 — rviz2 on NVIDIA

Launch rviz2 with PRIME offload:

```sh
$ __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia rviz2
```

**Verify:**

- rviz2 opens and renders the 3D viewport.
- Check the renderer: in rviz2, the OpenGL renderer is printed to
  stderr on launch, or check via:

```sh
$ __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia \
    glxinfo | grep 'OpenGL renderer'
```

Should show NVIDIA, not llvmpipe or virgl. **This is the project's
key validation.**

## Step 3 — TF2 / robot_state_publisher

Verify the full ROS 2 desktop stack beyond ros-base:

```sh
$ ros2 run tf2_ros static_transform_publisher --frame-id world --child-frame-id base_link
```

In another terminal:

```sh
$ ros2 topic echo /tf_static
```

**Verify:** the static transform is published and received.

## Step 4 — GPU utilization under load

While rviz2 is running (Step 2), check GPU utilization:

```sh
$ nvidia-smi
```

**Verify:** the GPU utilization percentage is non-zero (confirms the
GPU is actively rendering, not idle).

For continuous monitoring:

```sh
$ watch -n1 nvidia-smi
```

## Step 5 — Document performance comparison

Create a summary of the benchmark results. Inside the guest:

```sh
$ nano ~/gpu-performance-comparison.txt
```

Suggested format:

```
GPU Performance Comparison — ros2-lyrical-dev VM
================================================

Date: YYYY-MM-DD
Host: Dell Precision 3581, Ubuntu 26.04, kernel 7.0.0-15-generic

Phase 2 baseline (virtio-gpu / virgl):
  Renderer: <renderer string from Phase 2>
  glmark2 score: <score from Phase 2>

Phase 4 (NVIDIA RTX 2000 Ada via VFIO):
  Renderer: <renderer string from Step 1>
  glmark2 score: <score from Step 1>

Improvement: <NVIDIA score / virtio-gpu score>x
```

## Step 6 — Final snapshot

Shut down the guest:

```sh
$ sudo shutdown -h now
```

On the host:

```sh
$ sudo cp /var/lib/libvirt/images/ros2-lyrical-dev.qcow2 \
       /var/lib/libvirt/images/ros2-lyrical-dev.phase4-verified.qcow2
```

---

## Done — Phase 4 exit criteria

- [ ] glmark2 NVIDIA score recorded and compared to virtio-gpu
      baseline (both saved in `~/glmark2-*.txt`)
- [ ] rviz2 renders on the NVIDIA GPU (renderer string confirms)
- [ ] `nvidia-smi` shows non-zero GPU utilization during rviz2
- [ ] TF2 static_transform_publisher works
- [ ] Performance comparison documented in
      `~/gpu-performance-comparison.txt`
- [ ] Final disk backup
      `ros2-lyrical-dev.phase4-verified.qcow2` exists

## Project complete

All four phases are done. The VM is a general-purpose ROS 2 Lyrical
development environment with NVIDIA GPU acceleration via VFIO
passthrough.

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
