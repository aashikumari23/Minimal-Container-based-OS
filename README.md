
# Minimal Container-Based OS

This README explains all steps from booting the OS to running containers with networking support. Simple and ready to use.

---

## 1. Install Linux Kernel with cgroups and namespaces support

Download kernel source and configure:

```bash
make menuconfig
```

Enable:
- General setup > Control Group Support (Enable)
- General setup > Namespaces Support (Enable)

Build the kernel:

```bash
make -j$(nproc)
make bzImage
```

Output: `arch/x86/boot/bzImage`

---

## 2. Install and setup BusyBox

Download and install:

```bash
make menuconfig
make install
```

This creates a minimal root filesystem with BusyBox utilities like `ls`, `sh`, etc.

You can also manually copy your installed `busybox` binary into your rootfs.

---

## 3. Install GRUB bootloader

Install GRUB:

```bash
sudo apt install grub-pc-bin
```

Make a bootable ISO:

```bash
grub-mkrescue -o myos.iso /path/to/rootfs
```

**Example GRUB boot output:**
```
Welcome to GRUB!
Booting Minimal Container OS...
Loading Linux bzImage...
```

---

## 4. Use Tini as init process

Download Tini:

```bash
wget https://github.com/krallin/tini/releases/download/v0.19.0/tini
chmod +x tini
```

Place `tini` in `/sbin/init` inside rootfs.

Example simple `init` script (`/init`):
```bash
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
exec /bin/sh
```

Or use Tini directly.

---

## 5. Run with QEMU

Boot your OS ISO:

```bash
qemu-system-x86_64 -kernel arch/x86/boot/bzImage -initrd rootfs.cpio.gz -nographic -append "console=ttyS0"
```

---

## 6. Install runc

Install runc:

```bash
git clone https://github.com/opencontainers/runc
cd runc
make
sudo make install
```

**Important**: `runc` requires `cgroup`, `namespace`, and a running Linux kernel.

---

## 7. Create and Manage Containers

### Step-by-Step

1. Create a container spec:

```bash
runc spec
```

2. Modify `config.json` as needed (e.g., set `rootfs` path).

3. Create container:

```bash
runc create mycontainer
```

4. Start container:

```bash
runc start mycontainer
```

5. Run container (directly creates and starts):

```bash
runc run mycontainer
```

6. Check container state:

```bash
runc state mycontainer
```

**Example `runc state` output:**
```json
{
    "id": "mycontainer",
    "status": "running",
    "pid": 12345,
    "bundle": "/containers/mycontainer",
    "rootfs": "/containers/mycontainer/rootfs",
    "created": "2025-04-18T08:00:00Z"
}
```

7. List all containers:

```bash
runc list
```

**Example `runc list` output:**
```
ID          PID         STATUS      BUNDLE                CREATED
mycontainer 12345       running     /containers/mycontainer 2025-04-18T08:00:00Z
```

8. Kill container:

```bash
runc kill mycontainer
```

9. Delete container:

```bash
runc delete mycontainer
```

---

## 8. Add Networking to Containers (CNI)

### Importance

Containers need their own IP addresses and network isolation. CNI (Container Network Interface) provides this.

### Steps

Install CNI plugins:

```bash
git clone https://github.com/containernetworking/plugins.git
cd plugins
./build_linux.sh
```

Create simple networking script:

```bash
#!/bin/bash
BRIDGE=br0
ip link add $BRIDGE type bridge
ip addr add 192.168.1.1/24 dev $BRIDGE
ip link set $BRIDGE up

# Create veth pair
ip link add veth0 type veth peer name veth1
ip link set veth0 master $BRIDGE
ip link set veth0 up
ip link set veth1 up
ip netns add mynetns
ip link set veth1 netns mynetns
ip netns exec mynetns ip addr add 192.168.1.2/24 dev veth1
ip netns exec mynetns ip link set lo up
ip netns exec mynetns ip link set veth1 up
```

**Example `ip address` output:**
```
3: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    inet 192.168.1.1/24 brd 192.168.1.255 scope global br0
5: veth0@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP group default
    link/ether 6a:55:ab:7f:6e:89 brd ff:ff:ff:ff:ff:ff link-netns mynetns
```

---

# 🎯 Final Example Flow

Boot OS -> init -> Busybox shell -> runc create container -> runc start container -> container gets network IP using CNI.

Simple container command flow:

```bash
runc spec
runc create testcont
runc start testcont
runc list
runc state testcont
runc kill testcont
runc delete testcont
```

Network setup for container using CNI script.

---

# Congratulations!

You have built a **minimal container-based operating system** from scratch!

Included:
- Linux Kernel + bzImage
- BusyBox
- GRUB Bootloader
- Tini Init
- runc containers
- CNI Networking
