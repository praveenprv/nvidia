# Installing NVIDIA vGPU Manager on Ubuntu 24 KVM Hosts with L40S GPUs

This guide provides a step-by-step process for installing and configuring the NVIDIA vGPU Manager on Ubuntu 24 KVM hosts equipped with L40S GPUs, based on NVIDIA's official documentation. It includes integration with Rancher for GPU scheduling in a Kubernetes environment.

---

## Prerequisites Verification

Ensure your setup meets the following requirements:

```bash
# Check Ubuntu version
lsb_release -a

# Verify L40S GPUs are detected
lspci | grep -i nvidia

# Check KVM/QEMU installation
virsh --version
qemu-system-x86_64 --version

# Verify IOMMU is enabled
dmesg | grep -i iommu
```

---

## Step 1: Download NVIDIA vGPU Software

1. Access the NVIDIA Enterprise Support Portal:
   - Go to https://nvid.nvidia.com/dashboard/
   - Log in with your enterprise account.
   - Navigate to "Software Downloads" → "vGPU Software".
   - Download the Linux KVM package for your driver version.

2. Extract the package:
   ```bash
   unzip NVIDIA-Linux-x86_64-[version]-vgpu-kvm.zip
   ```

---

## Step 2: Prepare the Host System

On both physical servers:

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y build-essential dkms linux-headers-$(uname -r)

# Disable nouveau driver
echo 'blacklist nouveau' | sudo tee -a /etc/modprobe.d/blacklist-nouveau.conf
echo 'options nouveau modeset=0' | sudo tee -a /etc/modprobe.d/blacklist-nouveau.conf
sudo update-initramfs -u

# Reboot to apply changes
sudo reboot
```

---

## Step 3: Install vGPU Manager

On both physical servers:

```bash
# Navigate to extracted directory
cd NVIDIA-Linux-x86_64-[version]-vgpu-kvm/

# Make installer executable
chmod +x nvidia-installer

# Install vGPU Manager
sudo ./nvidia-installer --dkms

# Alternatively, if you have the .run file:
sudo sh ./NVIDIA-Linux-x86_64-[version]-vgpu-kvm.run --dkms
```

---

## Step 4: Configure vGPU Manager

```bash
# Load the nvidia-vgpu-vfio module
sudo modprobe nvidia-vgpu-vfio

# Verify installation
nvidia-smi vgpu

# Check available vGPU types for L40S
mdevctl types
```

---

## Step 5: Create vGPU Devices

Create mediated devices (mdev) for your VMs:

```bash
# List available mdev types for your GPU
ls /sys/class/mdev_bus/*/mdev_supported_types/

# Create vGPU instances (example for L40S)
# Replace [GPU_UUID] with actual GPU UUID from nvidia-smi -L
sudo mdevctl start -u [MDEV_UUID] -p [GPU_PCI_ADDRESS] -t [VGPU_TYPE]

# Make it persistent
sudo mdevctl define -u [MDEV_UUID] -p [GPU_PCI_ADDRESS] -t [VGPU_TYPE]
```

---

## Step 6: Configure KVM/QEMU

Edit your VM configuration to include the vGPU:

```bash
# For virsh/libvirt, edit the VM XML:
virsh edit [VM_NAME]
```

Add the following to the VM configuration:

```xml
<hostdev mode='subsystem' type='mdev' managed='yes' model='vfio-pci'>
  <source>
    <address uuid='[MDEV_UUID]'/>
  </source>
  <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
</hostdev>
```

---

## Step 7: Install vGPU Guest Driver

In your VMs (after VM restart):

```bash
# Download guest driver from NVIDIA portal
# Install the guest driver
sudo sh ./NVIDIA-Linux-x86_64-[version]-vgpu-kvm-guest.run

# Verify installation
nvidia-smi
```

---

## Step 8: Configure License Server

Set up NVIDIA vGPU licensing in each VM:

```bash
# Configure licensing
sudo nvidia-gridd -f

# Configure license server in /etc/nvidia/gridd.conf
sudo nano /etc/nvidia/gridd.conf
```

Add the following license server configuration:

```
ServerAddress=[LICENSE_SERVER_IP]
ServerPort=7070
FeatureType=1
```

---

## Step 9: Rancher Integration

Ensure GPU scheduling in Rancher-managed VMs:

```bash
# Label nodes with GPU capability
kubectl label nodes [NODE_NAME] nvidia.com/gpu=true

# Install NVIDIA Device Plugin
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/master/nvidia-device-plugin.yml
```

---

## Verification Steps

```bash
# On host - verify vGPU manager
nvidia-smi vgpu

# On VMs - verify guest driver
nvidia-smi

# Check mdev devices
mdevctl list

# Verify in Rancher
kubectl get nodes -o wide
kubectl describe node [NODE_NAME] | grep nvidia
```

---

## Key Implementation Points from NVIDIA Documentation

- **Host Requirements**: Section 2.1 of the NVIDIA vGPU documentation.
- **Driver Installation**: Section 3.2 for KVM hypervisor setup.
- **mdev Device Creation**: Section 4.3 for virtual GPU creation.
- **Guest VM Configuration**: Section 5.1 for VM setup.
- **Licensing**: Section 6.1 for license server configuration.

---

## Troubleshooting

```bash
# Check kernel modules
lsmod | grep nvidia

# Verify mdev support
ls /sys/class/mdev_bus/

# Check logs
dmesg | grep -i nvidia
journalctl -u nvidia-vgpu-mgr
```

---

## Notes

This setup enables Rancher-managed VMs to access vGPU resources from L40S GPUs on KVM hosts. Ensure appropriate resource limits and quotas are configured in Rancher for GPU scheduling. For further assistance, refer to the NVIDIA vGPU documentation.