# README: Simulating a Two‐Host QEMU/KVM Cluster on One Laptop

This guide walks you through all the commands and steps to mimic two separate hypervisor hosts (Host-VM-1 and Host-VM-2), each running their own nested guest (Nested-VM-A and Nested-VM-B), all on a single Ubuntu laptop. 

You will need a laptop, preferably one with good size RAM. You will also need a ethernet connection to the laptop.

---
### 1 On Your Ubuntu Laptop (Physical Host) Install KVM/QEMU, libvirt, virt-manager, etc.
```bash
sudo apt update
sudo apt install -y \
  qemu-kvm libvirt-daemon-system libvirt-clients \
  bridge-utils virt-manager cpu-checker
sudo usermod -aG libvirt,kvm $USER
# Log out & back in
```
### 2 Verify KVM Support
```
sudo kvm-ok
# → “Should print out KVM acceleration can be used”
```
### 3 Create a Linux Bridge on the Physical NIC
```
# identify the wired NIC name (mine was enx0826ae3a1b00)
ip link show

# 2. create & bring up br0
sudo ip link add br0 type bridge
sudo ip link set br0 up

# 3. enslave your NIC into br0
sudo ip link set enx0826ae3a1b00 master br0
sudo ip link set enx0826ae3a1b00 up


# 4. DHCP on br0sudo dhclient -v br
sudo dhclient -v br0
ip addr show br0
```
### 4 Create Two “Host” VMs (Level-1)
In virt-manager click New VM → Local install media → point at your Ubuntu ISO.

Name: Host-VM-1 | RAM: 1024 MiB | vCPUs: 1

Storage: 8 GiB qcow2 (default pool)

Network: Bridge br0, model virtio → Finish & install.

Repeat for Host-VM-2.
