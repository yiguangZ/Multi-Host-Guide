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
Enable nested virt in Host-VM-2
Back in the host’s virt-manager window, with Host-VM-2 selected, click the 🛈 (Show hardware details) icon.

Click the “XML” tab.

Locate the <cpu> element and change it from:

xml
Copy
Edit
<cpu mode="host-passthrough" …/>
to:

xml
Copy
Edit
<cpu mode="host-passthrough" …>
  <feature policy="require" name="vmx"/>
</cpu>
Click Apply, then Shut Down → Start Host-VM-2.

Inside Host-VM-2, verify:

bash
Copy
Edit
grep -E 'vmx|svm' /proc/cpuinfo
sudo kvm-ok
5. Install QEMU/KVM in Host-VM-2
In the Host-VM-2 console:

bash
Copy
Edit
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients \
                    bridge-utils virt-manager cpu-checker
sudo usermod -aG libvirt,kvm $USER
Log out and back in (or reboot) so your user is in the libvirt and kvm groups.

With both Host-VMs up, next you’ll repeat the nested-VM creation steps inside each (creating Nested-VM A in Host-VM-1 and Nested-VM B in Host-VM-2), hooking their NICs into br0, assigning them 10.0.0.10/20, and then testing connectivity.








You said:
  <cpu mode="host-passthrough" check="none" migratable="on"/>
Do the exact commands for me to copy


ChatGPT said:
Run:

bash
Copy
Edit
virsh edit Host-VM-2
Then find the single-line <cpu> entry:

xml
Copy
Edit
<cpu mode="host-passthrough" check="none" migratable="on"/>
Replace that line with exactly:

xml
Copy
Edit
<cpu mode="host-passthrough" check="none" migratable="on">
  <feature policy="require" name="vmx"/>
</cpu>
Save and exit. Then reboot Host-VM-2 for the change to take effect.

Create (or verify) a bridge inside Host-VM-1
You need a Linux bridge inside the VM so your nested guest can share the host-VM’s external link:

bash
Copy
Edit
# 1. Add and bring up the bridge
sudo ip link add name br0 type bridge
sudo ip link set dev br0 up

# 2. Enslave Host-VM-1’s physical interface (enp1s0) into it
sudo ip link set dev enp1s0 master br0
sudo ip link set dev enp1s0 up

# 3. DHCP for the bridge
sudo dhclient -v br0

# 4. Verify
ip addr show br0    # should show your real‐network IPv4
