# Running Flatcar Linux and Kairos on QEMU

## Introduction  

This guide demonstrates two tasks on an Ubuntu 24.04 host (24 CPU cores, 128 GB RAM) using KVM virtualization: (1) Running Flatcar Container Linux in a QEMU/KVM virtual machine with an Ignition config, and (2) Using Kairos’s containerized **osbuilder** to build a custom immutable OS image. Both sections include step-by-step instructions, configuration files, and explanations.

## 1. Flatcar Container Linux on QEMU/KVM  
Flatcar Container Linux is an immutable, container-optimized OS (a successor to CoreOS). We will download Flatcar’s QEMU image and launch script, create a minimal Ignition config (`config.ign`), and run Flatcar in a VM using KVM acceleration.

### Prerequisites

- **QEMU/KVM Installed:** On Ubuntu, install the QEMU/KVM and libvirt packages (e.g. `sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients`). Ensure your user has permission to use KVM (typically by being in the `kvm` group) and that `/dev/kvm` is present (enabled in BIOS).  

- **SSH Key Pair:** Generate an SSH key (`ssh-keygen`) if you don’t have one. The Flatcar QEMU script will use this to allow SSH access into the VM.

### Steps to Set Up Flatcar VM

1. **Download the Flatcar QEMU image and launch script.** 
   
   _Case for updated installation only:_

   Create a working directory and use `wget` to fetch the official Flatcar QEMU launch script and disk image (Stable channel):

   ```bash
   # Using a temporary local folder
   mkdir flatcar && cd flatcar

   # getting the latest stable QEMU script
   wget https://stable.release.flatcar-linux.net/amd64-usr/current/flatcar_production_qemu.sh

   # getting the latest stable QEMU image
   wget https://stable.release.flatcar-linux.net/amd64-usr/current/flatcar_production_qemu_image.img

   # making script executable
   chmod +x flatcar_production_qemu.sh
   ```

   _Case for demonstration of upgrade_

   Create a working directory and use `wget` to fetch a previous version and the official Flatcar QEMU launch script and disk image from the Stable channel:

   ```bash
   # Using a temporary local folder
   mkdir flatcar-upgradable && cd flatcar-upgradable

   # We start with old version of flatcar:
   OLD_VER="4230.2.4"

   # getting the OLD_VER QEMU script
   wget "https://stable.release.flatcar-linux.net/amd64-usr/${OLD_VER}/flatcar_production_qemu.sh"

   # Downloading the QEMU image for the that version
   wget "https://stable.release.flatcar-linux.net/amd64-usr/${OLD_VER}/flatcar_production_qemu_image.img.bz2"

   # uncompressing
   bunzip2 flatcar_production_qemu_image.img.bz2

   # Get the update payload for the CURRENT stable version
   STABLE_CURRENT=$(curl -s https://flatcar.cdn.cncf.io/stable/amd64-usr/4459.2.3/version.txt | grep FLATCAR_VERSION= | cut -d= -f2)

   # Downloading the current stable release for later upgrade
   wget "https://update.release.flatcar-linux.net/amd64-usr/${STABLE_CURRENT}/flatcar_production_update.gz"
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

   **A more advanced config.ign ignition file**

   ```json
   cat <<EOF >config.ign
   {
      "ignition": { "version": "3.3.0" },
      "passwd": {
         "users": [
            {
               "name": "core",
               "sshAuthorizedKeys": [
                  "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAO5nEatDfsM5X2QGaC1mvjsUtsrRbrrmKngKME1MqiS fabrizio.sgura@va"
               ]
            }
         ]
      },
      "storage": {
         "files": [
            {
               "path": "/etc/hostname",
               "mode": 420,
               "overwrite": true,
               "contents": { "source": "data:,flatcar-demo" }
            },
            {
               "path": "/etc/rancher/k3s/config.yaml",
               "mode": 420,
               "overwrite": true,
               "contents": {
                  "source": "data:,write-kubeconfig-mode%3A%20%220644%22%0Adisable%3A%0A%20%20-%20traefik%0A"
               }
            }
         ]
      },
      "systemd": {
         "units": [
            {
               "name": "k3s-install.service",
               "enabled": true,
               "contents": "[Unit]\nDescription=Install K3s\nWants=network-online.target\nAfter=network-online.target\nConditionPathExists=!/etc/systemd/system/k3s.service\n\n[Service]\nType=oneshot\nRemainAfterExit=yes\nRestart=on-failure\nRestartSec=15\nExecStart=/usr/bin/bash -lc 'for i in $(seq 1 20); do curl -sfL https://get.k3s.io && break; sleep 10; done | INSTALL_K3S_EXEC=\"server\" sh -'\n\n[Install]\nWantedBy=multi-user.target"
            },
            {
               "name": "sshd.service",
               "enabled": true
            }
         ]
      }
   }
   EOF
   ```

3. **Launch the Flatcar VM with QEMU.**

   Use the provided script to start the VM, including our Ignition file:  

   ```bash
   ./flatcar_production_qemu.sh -i config.ign -- -nographic
   ```  

   The script uses user-mode networking by default, forwarding SSH port 22 in the VM to host port 2222.  

4. **Access the Flatcar VM via SSH.**  

   This will work only when using a NAT configuration for QEMU, for bridge we have to specify the IP address

   ```bash
   ssh -p 2222 core@localhost
   ```  
   Verify setup:  
   ```bash
   hostname
   ```  
   Expected output: `flatcar-demo`.

5. **(Optional) Bridged Networking:**

   ```bash
   # In terminal
   sudo ./flatcar_production_qemu.sh -i config.ign -- -nographic -netdev bridge,id=user.0,br=virbr0 -device e1000e,netdev=user.0,id=net0,mac=a6:68:13:1d:e4:2d

   # In QEMU window
   sudo ./flatcar_production_qemu.sh -i config.ign -- -netdev bridge,id=user.0,br=virbr0 -device e1000e,netdev=user.0,id=net0,mac=a6:68:13:1d:e4:2d
   ```  

6. **Commands and useful options**

   Check the current root device

   ```bash
   rootdev -s /usr
   ```

   To set the boot partition

   ```bash
   sudo cgpt prioritize /dev/sda4
   ```

   To show the current /usr A B partitions

   ```bash
   sudo cgpt find -t flatcar-usr
   ```

   NOTE: In case an ERROR on 0 to 2048, that's because ```cgpt``` command cannot read the partition table for inspecting the /usr, it will not cause issues.

   Show the whole disk partitioning

   ```bash
   sudo cgpt show -v /dev/sda
   ```

   Verify the current partition table

   ```bash
   sudo gdisk -l /dev/sda
   ```

   Check the current flatcar release

   ```bash
   cat /etc/os-release
   ```

   Check the current update engine status

   ```bash
	update_engine_client --status
   ```
   

7. **Updates and Releases**

Flatcar updates and releases, with changelogs and security fixes are available at ```https://www.flatcar.org/releases```


8. **The release upgrade case**

In case of going for the demo with upgrade, after running a previous release of FlatCar, it is necessary to proceed with the airgap case, to save time and use the pre-downloaded image.
First we copy the image on the running machine, discovering it's IP address.

   ```bash
   scp -P 2222 ./flatcar_production_update.gz core@localhost:~/
   ```

or in case of using a bridge network configuration, discover IP and use the related correct endpoint.

There is also another requirement for the QEMU image, we need to download the additional extension file

   ```bash
   wget https://update.release.flatcar-linux.net/amd64-usr/4459.2.3/oem-qemu.gz
   ```

This file also need to be copied to the flatcar node, using scp.

Then to perform the update, inside the VM, run the local update targeting the new version:

   ```bash
   sudo flatcar-update \
     --to-version "${STABLE_VERSION:=4459.2.3}" \
     --to-payload /home/core/flatcar_production_update.gz \
     --extension /home/core/oem-qemu.gz
   ```

A status can be checked during the phase of upgrade.

   ```bash
   journalctl -u update-engine -f
   ```

At this point the upgrade is completed.

## 2. Building a Kairos OS Image with Kairos **osbuilder**  

[Kairos](https://kairos.io/) is an open-source meta-distribution for building immutable Linux OS images. It allows you to create custom OS images (Ubuntu, openSUSE, Alpine bases) with atomic upgrades and reproducibility.

### Prerequisites  
- **Docker:** Ensure Docker is installed and the service is running.  
- **KVM access for Docker:** Run with `--privileged` and mount `/dev` for KVM/QEMU acceleration.

### Steps to Build and Test a Kairos Image

1. **Prepare `config.yaml`:**
   ```bash
   mkdir -p kairos && cd kairos
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
       install:
         device: "auto"
         reboot: true
         poweroff: false
         auto: true
       users:
         - name: "kairos"
           passwd: "kairos"
           groups:
             - admin
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
  -drive file=demo-kairos-disk.qcow2,if=virtio
```

Choose interactive installation and follow the steps.
Once completed, boot the machine with the following

```bash
sudo qemu-system-x86_64 \
  -enable-kvm \
  -cpu host \
  -smp 4 \
  -m 4096 \
  -boot order=d \
  -drive file=demo-kairos-disk.qcow2,if=virtio \
```

For bridge network, follow the next example

```bash
sudo qemu-system-x86_64 \
  -enable-kvm \
  -cpu host \
  -smp 4 \
  -m 4096 \
  -boot order=d \
  -drive file=demo-kairos-disk.qcow2,if=virtio \
  -netdev bridge,id=user.0,br=virbr0 \
  -device e1000e,netdev=user.0,id=net0,mac=a6:68:13:1d:e4:2d
```

and play with Kairos and the already installed k3s version 1.32.9 for this case.

To perform a Kairos upgrade, we can run the next commands:

```bash
# List available releases
sudo kairos-agent upgrade list-releases
# Select one and upgrade, as the next example
sudo kairos-agent upgrade --source oci:quay.io/kairos/fedora:40-standard-amd64-generic-v3.7.2-k3s-v1.35.0-k3s3
```

Once the upgrade is completed, the new kubernetes binary should be the version 1.35.0.
Running

```bash
# List available releases
sudo kairos-agent upgrade list-releases
```

the current version should be the installed one.

### Kairos Use Cases  
- **Custom Kubernetes nodes:** Build pre-configured k3s-enabled images.  
- **Edge or IoT devices:** Immutable images for reproducible deployments.  
- **Bare-metal automation:** ISO autoinstallers for field deployment.  

## Conclusion  
We demonstrated Flatcar setup and Kairos OS image building using QEMU/KVM on Ubuntu 24.04. Flatcar provides a quick immutable runtime, while Kairos offers a flexible way to build custom immutable OS images. Both illustrate the principles of **immutable infrastructure** and **automated provisioning**.
