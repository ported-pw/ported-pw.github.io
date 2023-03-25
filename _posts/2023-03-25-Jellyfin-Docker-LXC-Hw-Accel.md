---
title: NVIDIA GPU hardware acceleration for Jellyfin on Docker inside a Proxmox LXC
---
For virtualizing my own PC with Proxmox I had to add another GPU (a cheap Quadro P620) as the host output, which I also planned on using
for hardware acceleration for my media collection in Jellyfin. This is how to make that work.

## Overview of the setup
- Proxmox VE host
- Podman (or Docker) in an LXC
- Jellyfin running using the official container

## Guide

### Important hints upfront if you just came here seeking answers
- This is about using a GPU that is _not_ passed through to a VM, but rather used for the host.
- The NVIDIA driver in the LXC and on the Proxmox host need to be the same version.

### NVIDIA driver setup on the Proxmox host

For the hardware acceleration to work with all codecs supported by the GPU [^1][^2], we need the proprietary NVIDIA drivers on the host instead of `nouveau`.

Blacklist the `nouveau` driver kernel module from being loaded on boot in `/etc/modprobe.d/blacklist.conf` [^3], it should look like this:
```
blacklist nouveau
```

Download the proprietary drivers from [NVIDIAs download page](https://www.nvidia.com/download/index.aspx?lang=en-us), selecting the GPU you want to use and _Linux 64-bit_.

Install the drivers, choosing "Yes" for registering it with DKMS, e.g.:
```
chmod +x ./NVIDIA-Linux-x86_64-525.89.02.run
./NVIDIA-Linux-x86_64-525.89.02.run
```

Reboot, and run `nvidia-smi` to test if the driver works properly.


We will also need to load the CUDA kernel module. Add the following two lines to `/etc/modules-load.d/modules.conf`[^4]:  
```
nvidia
nvidia_uvm
```
and update `initramfs`:
```
update-initramfs -u
```

To have the required device nodes we need to pass to the LXC later created, add the following to `/etc/udev/rules.d/70-nvidia-rules`[^4]:
```
# Create /nvidia0, /dev/nvidia1 and /nvidiactl when nvidia module is loaded
KERNEL=="nvidia", RUN+="/bin/bash -c '/usr/bin/nvidia-smi -L && /bin/chmod 666 /dev/nvidia*'"
# Create the CUDA node when nvidia_uvm CUDA module is loaded
KERNEL=="nvidia_uvm", RUN+="/bin/bash -c '/usr/bin/nvidia-modprobe -c0 -u && /bin/chmod 0666 /dev/nvidia-uvm*'"
```

Reboot the server, and make sure you see the following device nodes:
```
/dev/nvidia-uvm
/dev/nvidia-uvm-tools
/dev/nvidia0
/dev/nvidiactl
```

### Setting up the LXC

Create a normal (_Unprivileged_ is fine) LXC in the GUI with the OS of your choice (it does _not_ have to be the same as the host), and make sure `nesting` and `keyctl` are enabled in _Options -> Features_.

Then, edit the LXC config file in `/etc/pve/lxc/<id>.conf` and add the following to mount the device nodes:
```
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file
```


### Setting up NVIDIA drivers inside the LXC

**Now in the LXC**:
Install **_the same version_** NVIDIA drivers downloaded before, passing `--no-kernel-module` since we are using the driver loaded in the host kernel. E.g.:
```
chmod +x ./NVIDIA-Linux-x86_64-525.89.02.run
./NVIDIA-Linux-x86_64-525.89.02.run --no-kernel-module
```
Then, run `nvidia-smi` to ensure the driver works and the GPU shows up.

### Setting up Docker/Podman in the LXC and running Jellyfin

Install either _Docker_ or _Podman_ depending on your preference (I recommend _Podman_ as its the replacement for the Docker daemon by RedHat).  
_Notes for Podman:_
1. Enable `podman.service` to start Podman on boot
2. Enable `podman-restart.service` to start all containers flagged with `restart: always` on boot.

Install the [NVIDIA Container toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html) according to their official instructions
based on what platform your LXC is using.

Follow the instructions on how to start a container with the NVIDIA container toolkit hooks, in my case with podman it looks like this:
```
podman run -d --name jellyfin --restart=always -v /srv/jellyfin/config:/config -v /srv/jellyfin/cache:/cache -v /mnt/media:/media --security-opt=label=disable --hooks-dir=/usr/share/containers/oci/hooks.d/ -p 8096:8096 -p 8920:8920 -p 1900:1900/udp -p 7359:7359/udp jellyfin/jellyfin:latest
```

The last step would now be to enable Hardware decoding in Jellyfin and check all options that your GPU supports.

#### References and credits:
[^1]: [NVENC/NVDEC support matrix](https://developer.nvidia.com/video-encode-and-decode-gpu-support-matrix-new), [Nvidia NVDEC (Wiki)](https://en.wikipedia.org/wiki/Nvidia_NVDEC), [Nvidia NVENC (Wiki)](https://en.wikipedia.org/wiki/Nvidia_NVENC)  
[^2]: [feedesktop wiki](https://nouveau.freedesktop.org/VideoAcceleration.html)  
[^3]: [Proxmox wiki](https://pve.proxmox.com/wiki/Developer_Workstations_with_Proxmox_VE_and_X11#Optional:_NVidia_Drivers)  
[^4] [This blog posted](https://bradford.la/2016/GPU-FFMPEG-in-LXC/) helped me get across the finish line.  