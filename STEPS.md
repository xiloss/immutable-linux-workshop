# Running Flatcar Linux on QEMU and Building a Kairos OS Image (Ubuntu 24.04 Host)

## Introduction  
This guide demonstrates two tasks on an Ubuntu 24.04 host (24 CPU cores, 128 GB RAM) using KVM virtualization: (1) Running Flatcar Container Linux in a QEMU/KVM virtual machine with an Ignition config, and (2) Using Kairos’s containerized **osbuilder** to build a custom immutable OS image. Both sections include step-by-step instructions, configuration files, and explanations.

## 1. Flatcar Container Linux on QEMU/KVM  
Flatcar Container Linux is an immutable, container-optimized OS (a successor to CoreOS). We will download Flatcar’s QEMU image and launch script, create a minimal Ignition config (`config.ign`), and run Flatcar in a VM using KVM acceleration.

### Prerequisites

- **QEMU/KVM Installed:** On Ubuntu, install the QEMU/KVM and libvirt packages (e.g. `sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients`). Ensure your user has permission to use KVM (typically by being in the `kvm` group) and that `/dev/kvm` is present (enabled in BIOS).  

- **SSH Key Pair:** Generate an SSH key (`ssh-keygen`) if you don’t have one. The Flatcar QEMU script will use this to allow SSH access into the VM.

### Steps to Set Up Flatcar VM

1. **Download the Flatcar QEMU image and launch script.** Create a working directory and use `wget` to fetch the official Flatcar QEMU launch script and disk image (Stable channel):
    ```bash
    mkdir flatcar && cd flatcar
    wget https://stable.release.flatcar-linux.net/amd64-usr/current/flatcar_production_qemu.sh
    wget https://stable.release.flatcar-linux.net/amd64-usr/current/flatcar_production_qemu_image.img
    chmod +x flatcar_production_qemu.sh
    ```  

2. **Create a minimal Ignition config (`config.ign`).** Ignition is Flatcar’s boot-time provisioning tool. We’ll use it to perform a basic configuration on first boot. For demo purposes, our Ignition config will set a custom hostname. Create a file **`config.ign`** with the following content:  

    ```json
    cat <<EOF >config.ign
    {
        "ignition": { "version": "3.0.0" },
        "storage": {
        "files": [{
            "path": "/etc/hostname",
            "mode": 420,
            "overwrite": true,
            "contents": { "source": "data:,flatcar-demo" }
        }]
        }
    }
    EOF
    ```  

3. **Launch the Flatcar VM with QEMU.** Use the provided script to start the VM, including our Ignition file:  
   ```bash
   ./flatcar_production_qemu.sh -i config.ign -- -nographic
   ```  
   The script uses user-mode networking by default, forwarding SSH port 22 in the VM to host port 2222.  

4. **Access the Flatcar VM via SSH.**  
   ```bash
   ssh -p 2222 core@localhost
   ```  
   Verify setup:  
   ```bash
   hostname
   ```  
   Expected output: `flatcar-demo`.

5. **(Optional) Bridged Networking:** For LAN access, set up a TAP interface and bridge it with a host NIC, then use `-netdev tap` and `-device virtio-net-pci` in QEMU. NAT suffices for most demos.

## 2. Building a Kairos OS Image with Kairos **osbuilder**  

[Kairos](https://kairos.io/) is an open-source meta-distribution for building immutable Linux OS images. It allows you to create custom OS images (Ubuntu, openSUSE, Alpine bases) with atomic upgrades and reproducibility.

### Prerequisites  
- **Docker:** Ensure Docker is installed and the service is running.  
- **KVM access for Docker:** Run with `--privileged` and mount `/dev` for KVM/QEMU acceleration.

### Steps to Build and Test a Kairos Image

1. **Prepare `config.yaml`:**
   ```yaml
   mkdir -p config && cd kairos
   cat <<EOF >config.yaml
   apiVersion: build.kairos.io/v1alpha2
   kind: OSArtifact
   metadata:
     name: demo-kairos
   spec:
     imageName: "quay.io/kairos/core-ubuntu-24-lts:latest"
     iso: true
     cloudConfig: |
       #cloud-config
       users:
       - name: "kairos"
         passwd: "kairos"
       install:
       device: "auto"
       reboot: true
       poweroff: false
       auto: true
   EOF
   ```  

2. **Build the Kairos OS image:**
   ```bash
   docker run --privileged -v /dev:/dev -v "${PWD}:/config" -ti quay.io/kairos/osbuilder build os --config /config/config.yaml
   ```  

3. **Locate the output artifact:**
   ```bash
   ls -lh
   ```  
   Expect `demo-kairos.iso` and checksum file.

4. **Test the ISO in a VM:**
   ```bash
   qemu-img create -f qcow2 demo-kairos-disk.qcow2 20G
   qemu-system-x86_64 -m 2048 -enable-kvm \
      -drive file=demo-kairos-disk.qcow2,if=virtio \
      -cdrom demo-kairos.iso \
      -boot order=d \
      -nic user,model=virtio
   ```  

   The VM installs Kairos automatically (via `auto: true`), reboots, and allows login with `kairos`/`kairos`.

## 3. Run Kairos without a build

For this case, download the kairos image from the github repository.
Search for a release and download it.
For this case, the `kairos-fedora-40-standard-amd64-generic-v3.5.4-k3sv1.32.9+k3s1.iso` image will be used.

Simply run a qemu machine to perform the installation
First create a qcow disk:

```bash
qemu-img create -f qcow2 demo-kairos-disk.qcow2 20G
```

and then run the installer

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -cpu host \
  -smp 4 \
  -m 4096 \
  -boot order=d \
  -drive file=kairos-fedora-40-standard-amd64-generic-v3.5.4-k3sv1.32.9+k3s1.iso,media=cdrom \
  -drive file=kairos-disk.qcow2,if=virtio
```

Choose interactive installation and follow the steps.
Once completed, boot the machine with

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -cpu host \
  -smp 4 \
  -m 4096 \
  -boot order=d \
  -drive file=kairos-disk.qcow2,if=virtio
```

and play with Kairos and the already installed k3s version 1.32.9 for this case.

### Kairos Use Cases  
- **Custom Kubernetes nodes:** Build pre-configured k3s-enabled images.  
- **Edge or IoT devices:** Immutable images for reproducible deployments.  
- **Bare-metal automation:** ISO autoinstallers for field deployment.  

## Conclusion  
We demonstrated Flatcar setup and Kairos OS image building using QEMU/KVM on Ubuntu 24.04. Flatcar provides a quick immutable runtime, while Kairos offers a flexible way to build custom immutable OS images. Both illustrate the principles of **immutable infrastructure** and **automated provisioning**.
