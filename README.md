# QCOW2VHD
Documentation on Converting KVM QCOW2 Virtual Machine Disks into Importable Hyper-V VHDX Format

## Overview

Converting a KVM qcow2 virtual machine to VHDX format for Hyper-V import requires more than just running `qemu-img convert`. There's a critical kernel driver issue that must be addressed first. When you simply convert the disk format without preparing the Linux VM, it will fail to boot in Hyper-V because the initramfs lacks the necessary Hyper-V drivers.

This guide provides the complete process to prepare your Linux VM and successfully convert it for Hyper-V.

## Prerequisites

- A running KVM Linux virtual machine with qcow2 disk format
- Root access to the Linux VM
- QEMU tools installed on the host system
- Sufficient disk space for the conversion process

## Step-by-Step Conversion Process

### Phase 1: Prepare the Linux VM (Critical!)

**Important:** These steps must be performed INSIDE the running Linux VM before conversion. Skipping this phase will result in a non-bootable VM in Hyper-V.

#### 1. Add Hyper-V drivers to dracut config

```bash
sudo tee /etc/dracut.conf.d/hyperv.conf >/dev/null <<'EOF'
add_drivers+=" hv_vmbus hv_storvsc hv_netvsc hv_utils "
hostonly="no"
EOF
```

This tells dracut to always include the Hyper-V bus, storage, and network drivers in the initramfs, and not to restrict to just the current host's drivers.

#### 2. Rebuild initramfs for the current running kernel

```bash
sudo dracut -f --kver "$(uname -r)"
```

This usually takes 15–45 seconds and regenerates `/boot/initramfs-<your-kernel>.img` with those Hyper-V modules baked in.

#### 3. (Optional but smart) Confirm the image exists

```bash
ls -lh /boot/initramfs-$(uname -r).img
```

→ Should see a 50–100 MB file.

#### 4. Check /etc/fstab

Make sure you're using UUID= entries (Hyper-V uses `/dev/sdX` rather than `/dev/vdX`):

```bash
cat /etc/fstab
blkid    # to get UUIDs if needed
```

#### 5. Shutdown the VM properly

```bash
sudo shutdown -h now
```

### Phase 2: Convert the Disk Format

Now that the VM is prepared, convert the qcow2 disk to VHDX format:

#### 1. Locate your qcow2 file

```bash
# Typical locations:
# /var/lib/libvirt/images/
# /home/user/VMs/
ls -lh /path/to/your-vm.qcow2
```

#### 2. Convert qcow2 to VHDX

```bash
qemu-img convert -f qcow2 -O vhdx /path/to/your-vm.qcow2 /path/to/output/your-vm.vhdx
```

**Options:**
- `-f qcow2`: Source format
- `-O vhdx`: Output format
- Add `-p` for progress indicator on large disks

#### 3. Verify the conversion

```bash
qemu-img info /path/to/output/your-vm.vhdx
```

### Phase 3: Import to Hyper-V

#### 1. Copy VHDX to Windows Hyper-V host

Transfer the `.vhdx` file to your Windows machine where Hyper-V is installed.

#### 2. Create new Hyper-V VM

1. Open Hyper-V Manager
2. Click "New" → "Virtual Machine"
3. Follow the wizard:
   - Choose "Generation 1" for better compatibility
   - Assign appropriate memory
   - Use existing virtual hard disk: select your converted `.vhdx` file
   - Configure network adapter

#### 3. Configure VM settings (before first boot)

- **Integration Services**: Enable all available services
- **Network Adapter**: Configure for your network setup
- **Memory**: Consider enabling Dynamic Memory if appropriate
- **Processor**: Assign appropriate CPU count

#### 4. First boot and verification

1. Start the VM
2. Verify network connectivity
3. Check that all hardware is detected properly
4. Install/update Hyper-V Integration Services if needed

## Troubleshooting

### VM won't boot in Hyper-V
- **Cause**: Initramfs missing Hyper-V drivers
- **Solution**: Go back to Phase 1 and ensure dracut configuration is correct

### Network not working
- **Cause**: Missing `hv_netvsc` driver
- **Solution**: Verify the dracut config includes `hv_netvsc` and rebuild initramfs

### Storage issues
- **Cause**: Missing `hv_storvsc` driver or wrong device names in fstab
- **Solution**: Check dracut config and ensure fstab uses UUIDs

### Performance issues
- **Cause**: Integration Services not properly installed
- **Solution**: Install/update Hyper-V Integration Services in the guest

## Additional Notes

- **Backup**: Always backup your original qcow2 file before conversion
- **Testing**: Test the converted VM thoroughly before decommissioning the original
- **Integration Services**: Install the latest Hyper-V Integration Services for optimal performance
- **Snapshots**: Consider taking a checkpoint immediately after successful conversion

## Supported Linux Distributions

This process has been tested with:
- RHEL/CentOS 7/8/9
- Ubuntu 18.04/20.04/22.04/24.04
- Fedora 35+
- Debian 10/11/12

Other systemd-based distributions using dracut should work similarly.

