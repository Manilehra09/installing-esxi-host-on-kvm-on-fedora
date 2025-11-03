#  VMware ESXi 7.0U3g Installation on Fedora KVM (Working Configuration)

This guide documents the **tested and verified** steps to install **VMware ESXi 7.0U3g** inside **KVM on Fedora 43**.  
It includes correct CPU, chipset, bridge, and UEFI configuration — no purple screen (PSOD), no network errors.

---

## Prerequisites

### 1. Enable Virtualization
Check CPU virtualization support:
```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```
> Output > 0 means virtualization is supported.

---

### 2. Install KVM and Virtualization Tools
```bash
sudo dnf install -y @virtualization virt-manager libvirt libvirt-daemon-kvm qemu-kvm bridge-utils edk2-ovmf
sudo systemctl enable --now libvirtd
```

---

### 3. Create Network Bridge (`br0`)

Ensure your **Ethernet** interface is bridged to `br0` (for real LAN access):

```bash
sudo nmcli connection add type bridge ifname br0 con-name br0 ipv4.method auto
sudo nmcli connection add type ethernet ifname enp130s0 master br0
sudo nmcli connection up bridge-slave-enp130s0
sudo nmcli connection up br0
```

Confirm with:
```bash
ip addr show br0
```
✅ You should see your LAN IP (e.g. `192.168.1.x`) and state `UP`.

---

### 4. Prepare ISO and Storage
Download or locate your ESXi ISO:
```
<path of your iso file>/VMware-VMvisor-Installer-7.0U3g-20328353.x86_64.iso
```

Create a QCOW2 disk for ESXi:
```bash
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/esxi-host.qcow2 1700G
```

Create UEFI NVRAM file:
```bash
sudo mkdir -p /var/lib/libvirt/qemu/nvram
sudo cp /usr/share/OVMF/OVMF_VARS.fd /var/lib/libvirt/qemu/nvram/esxi-host_VARS.fd
```

---

## Step-by-Step Installation

### 🪄 1. Create and Start the ESXi VM
Run this **virt-install** command:

```bash
sudo virt-install --name esxi-host --memory 81920 --vcpus 20 --cpu host-passthrough --machine pc-i440fx-8.2 --os-variant generic --disk path=/var/lib/libvirt/images/esxi-host.qcow2,format=qcow2,bus=sata,size=1700 --cdrom <path of iso>/VMware-VMvisor-Installer-7.0U3g-20328353.x86_64.iso --network bridge=br0,model=e1000e --boot loader=/usr/share/OVMF/OVMF_CODE.fd,nvram=/var/lib/libvirt/qemu/nvram/esxi-host_VARS.fd --graphics vnc,listen=0.0.0.0,port=5901 --video vmvga
```

---

### 2. Access the Console
Either open via:
```bash
virt-manager
```
or from your MacBook using:
```
vnc://<fedora-ip>:5901
```

---

###  3. Install ESXi
Follow the steps:

1. **Accept EULA:** `F11`
2. **Select Disk:** `QEMU HARDDISK` (1.7 TB)
3. **Keyboard Layout:** Default (`US`)
4. **Set Root Password**
5. **Confirm Installation:** `F11`
6. Wait 5–10 minutes.
7. When done, **press Enter** to reboot.

---

### 4. After Reboot
ESXi will boot automatically.  
You’ll see:
```
VMware ESXi 7.0.3
IP Address: 192.168.1.xxx
```

---

###  5. Access Web UI
From your browser:
```
https://192.168.1.xxx/
```
Login:
```
Username: root
Password: <your-password>
```

✅ You’ll now be inside the ESXi Web Interface.

---

##  Post-Installation Tips

###  Verify Network
In the ESXi Web UI:
- Navigate to **Networking → VMkernel NICs**
- Confirm it has an IP from your LAN (`192.168.1.x`)

###  Enable Nested Virtualization (Optional)
If you plan to run VMs inside ESXi:
1. SSH to ESXi host:
   ```bash
   ssh root@192.168.1.xxx
   ```
2. Run:
   ```bash
   esxcli system settings kernel set -s nestedHVSupported -v TRUE
   ```

---
