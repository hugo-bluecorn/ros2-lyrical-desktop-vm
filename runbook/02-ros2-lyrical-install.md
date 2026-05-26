# 02 — ROS 2 Lyrical install + baseline benchmark

Install ROS 2 Lyrical desktop inside the guest, verify the ROS 2
stack end-to-end, and record a GPU baseline benchmark on virtio-gpu
before adding NVIDIA passthrough in Phase 3.

## Goal

After this phase:

- `ros-lyrical-desktop` and `ros-dev-tools` are installed.
- `ros2` CLI works, talker/listener communicates, colcon builds.
- Environment auto-sourced on login.
- `glmark2-wayland` baseline score recorded on virtio-gpu (the
  "before" measurement for Phase 4 comparison).
- Disk backup taken.

## Sources

- [ROS 2 Lyrical Ubuntu install (apt)][ros2-install] — official
  installation guide using the `ros-apt-source` deb package.
- [glmark2 source][glmark2-src] — OpenGL 2.0 benchmark.

[ros2-install]: https://docs.ros.org/en/lyrical/Installation/Ubuntu-Install-Debs.html
[glmark2-src]: https://github.com/glmark2/glmark2

## Prerequisites

- Phase 1 complete: VM `ros2-lyrical-dev` boots to Kubuntu 26.04
  desktop, network working.

## Adapt

| Aspect | This runbook | If yours differs |
|--------|-------------|-----------------|
| ROS 2 distro | Lyrical | Substitute your distro name in package names |
| Install variant | `ros-lyrical-desktop` | Use `ros-lyrical-ros-base` for headless |

---

## Step 1 — Verify locale

Inside the guest:

```sh
$ locale
```

**Verify:** `LANG` shows a UTF-8 locale (e.g., `en_US.UTF-8`).
Kubuntu 26.04 sets this by default. If not:

```sh
$ sudo locale-gen en_US en_US.UTF-8
$ sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
```

## Step 2 — Enable the Universe repository

The ROS 2 apt source depends on packages in the Universe pocket.
Kubuntu 26.04 desktop enables Universe by default, but the official
ROS 2 install guide prescribes this step as a safety check.

```sh
$ sudo apt install software-properties-common
$ sudo add-apt-repository universe
```

**Verify:**

```sh
$ apt-cache policy | grep universe
```

Shows one or more `universe` lines (e.g.,
`l=Ubuntu,c=universe,b=amd64`). If no output, re-run the
`add-apt-repository` command above.

## Step 3 — Install the ROS 2 apt source

The ROS 2 project distributes a deb package that configures the apt
repository and signing key in one step.

```sh
$ sudo apt-get update && sudo apt-get install curl -y
$ export ROS_APT_SOURCE_VERSION=$(curl -s \
    https://api.github.com/repos/ros-infrastructure/ros-apt-source/releases/latest \
    | grep -F "tag_name" | awk -F'"' '{print $4}')
$ echo "ros-apt-source version: $ROS_APT_SOURCE_VERSION"
$ curl -L -o /tmp/ros2-apt-source.deb \
    "https://github.com/ros-infrastructure/ros-apt-source/releases/download/${ROS_APT_SOURCE_VERSION}/ros2-apt-source_${ROS_APT_SOURCE_VERSION}.$(. /etc/os-release && echo ${UBUNTU_CODENAME:-${VERSION_CODENAME}})_all.deb"
$ sudo dpkg -i /tmp/ros2-apt-source.deb
$ sudo apt-get update
```

**Verify:**

```sh
$ apt-cache policy ros-lyrical-desktop
```

Shows a candidate version from the ROS 2 repository (not "N/A").

## Step 4 — Install ROS 2 Lyrical desktop

```sh
$ sudo apt-get update && sudo apt-get upgrade -y
$ sudo apt install ros-lyrical-desktop
```

This pulls in rviz2, rqt, tf2, robot_state_publisher, demo nodes,
and the full ROS 2 desktop toolchain.

**Verify:**

```sh
$ dpkg -l | grep ros-lyrical | wc -l
```

Expect a large number of packages (200+).

## Step 5 — Install dev tools

```sh
$ sudo apt install ros-dev-tools
```

This adds colcon, rosdep, vcstool, and other build utilities.

## Step 6 — Environment setup

Add the ROS 2 environment to your shell profile:

```sh
$ echo 'source /opt/ros/lyrical/setup.bash' >> ~/.bashrc
$ source ~/.bashrc
```

**Verify:**

```sh
$ ros2 --help
$ echo $ROS_DISTRO
```

`ros2 --help` prints the CLI usage. `ROS_DISTRO` shows `lyrical`.

## Step 7 — Talker/listener smoke test

Open two terminals inside the guest.

**Terminal 1:**

```sh
$ ros2 run demo_nodes_cpp talker
```

**Terminal 2:**

```sh
$ ros2 run demo_nodes_py listener
```

**Verify:** Terminal 1 publishes `Hello World: N` messages. Terminal 2
receives and prints them. This confirms the ROS 2 middleware (DDS) is
working end-to-end.

Stop both with Ctrl+C.

## Step 8 — colcon build test

Create a minimal workspace and build it:

```sh
$ mkdir -p ~/ros2_ws/src
$ cd ~/ros2_ws
$ ros2 pkg create --build-type ament_cmake --node-name hello_node hello_pkg
$ colcon build
$ source install/setup.bash
$ ros2 run hello_pkg hello_node
```

**Verify:** `colcon build` completes with `0 errors`. The hello node
runs and prints output.

Clean up:

```sh
$ rm -rf ~/ros2_ws
```

## Step 9 — Baseline GPU benchmark (virtio-gpu)

Install and run `glmark2-wayland` to record the virtio-gpu baseline.
This score will be compared against the NVIDIA VFIO score in Phase 4.

```sh
$ sudo apt install glmark2-wayland
$ glmark2-wayland
```

`glmark2-wayland` runs a series of OpenGL 2.0 scenes and reports a
composite score at the end.

**Record these values:**

1. The **OpenGL renderer string** printed at the start (expect
   `virgl` or `llvmpipe` on virtio-gpu)
2. The **composite score** printed at the end (e.g.,
   `glmark2 Score: 1234`)

Write them down or save to a file:

```sh
$ glmark2-wayland 2>&1 | tee ~/glmark2-baseline-virtio-gpu.txt
```

**Watch out:** if `glmark2-wayland` fails with a display error,
try `glmark2-x11` as a fallback (runs via XWayland). Install with
`sudo apt install glmark2-x11`.

## Step 10 — Snapshot

Shut down the guest:

```sh
$ sudo shutdown -h now
```

On the host, create a disk backup:

```sh
$ sudo cp /var/lib/libvirt/images/ros2-lyrical-dev.qcow2 \
       /var/lib/libvirt/images/ros2-lyrical-dev.ros2-installed.qcow2
```

---

## Done — Phase 2 exit criteria

- [ ] `ros2 --help` works, `ROS_DISTRO` is `lyrical`
- [ ] `ros-lyrical-desktop` installed (200+ packages)
- [ ] Talker/listener communicates across terminals
- [ ] `colcon build` succeeds on a minimal workspace
- [ ] Environment auto-sourced on login (`~/.bashrc`)
- [ ] glmark2 baseline score recorded with renderer string
      (saved to `~/glmark2-baseline-virtio-gpu.txt`)
- [ ] Disk backup `ros2-lyrical-dev.ros2-installed.qcow2` exists

## Next

[Phase 3 — NVIDIA GPU passthrough](03-nvidia-gpu-passthrough.md)
