# Ubuntu Server Setup

**Wazuh SOC Home Lab**

- [Oracle VirtualBox](https://www.virtualbox.org/wiki/Downloads)
- [Ubuntu Server 24.04.5 LTS](https://ubuntu.com/download/server/thank-you?version=24.04.5&architecture=amd64&lts=true)

This guide covers the Ubuntu Server installation (guest VM) and network configuration used in my Wazuh home lab. A separate Host-only network is configured to keep lab communication isolated from the physical network.

Wazuh installation and agent configuration are documented separately.

You can follow this guide to set up your own Ubuntu Server for a Wazuh lab.

## 1. Create the virtual machine

Download the [Ubuntu Server 24.04 LTS ISO](https://ubuntu.com/download/server/thank-you?version=24.04.5&architecture=amd64&lts=true), then open **VirtualBox → New**.

| Setting                      | Selection                          |
| ---------------------------- | ---------------------------------- |
| Name                         | `Wazuh-Server`                     |
| OS / ISO                     | Ubuntu Server 24.04.5 LTS (64-bit) |
| RAM                          | 8192 MB (8 GB)                     |
| CPUs                         | 4                                  |
| Disk                         | 60 GB, dynamically allocated       |
| Skip Unattended Installation | Enabled (install manually)         |
| EFI                          | Disabled                           |
| Network Adapter 1            | NAT (other settings default)       |

## 2. Install Ubuntu Server

Start the VM and select **Try or Install Ubuntu Server**.

| Installer screen            | Selection                                    |
| --------------------------- | -------------------------------------------- |
| Installation type           | Ubuntu Server (standard), not minimized      |
| Third-party drivers         | Unchecked                                    |
| Network                     | Default DHCP                                 |
| Proxy                       | Leave blank                                  |
| Ubuntu archive mirror       | Keep default                                 |
| Storage                     | Use entire **60 GB VirtualBox disk**, `ext4` |
| LVM                         | Unchecked                                    |
| LUKS encryption             | Unchecked                                    |
| Server name                 | `wazuh-server`                               |
| Username                    | `socadmin` (example)                         |
| Ubuntu Pro                  | Skip for now                                 |
| OpenSSH server              | Checked                                      |
| SSH password authentication | Enabled                                      |
| Import SSH key              | Skip                                         |
| Featured server snaps       | None selected                                |

Before confirming storage changes, verify the selected drive is the **60 GB `VBOX_HARDDISK`**, not a physical disk. **Create a strong password, finish installation and reboot.**

## 3. Update Ubuntu

Log in and run:

```bash
sudo apt update
sudo apt upgrade -y
sudo reboot
```

After reboot, verify the installation:

```bash
lsb_release -a
uname -r
```

## 4. Add the Host-only network

Shut down Ubuntu before changing VirtualBox settings:

```bash
sudo shutdown -h now
```

In **VirtualBox → Wazuh-Server → Settings → Network**:

- **Adapter 1:** Keep **NAT** for internet access.
- **Adapter 2:** Enable **Host-only Adapter**, select **VirtualBox Host-Only Ethernet Adapter**, and make sure **Cable Connected** is checked.
- **Adapter Type:** Intel PRO/1000 MT Desktop (82540EM) - keep default.
- Click **OK** to save.

> **Note:** If you can't see Adapter 2, check the top-left corner of the VirtualBox settings window. Switch from **Basic** to **Expert** view to see the additional adapter options.

Next, open **VirtualBox → File → Tools → Network → Host-only Networks**.

Select the Host-only adapter and check its IPv4 and DHCP settings.

These were the settings in my lab:

| Setting             | Value             |
| ------------------- | ----------------- |
| Host adapter        | `192.168.56.1/24` |
| DHCP server         | `192.168.56.100`  |
| Lower address bound | `192.168.56.101`  |
| Upper address bound | `192.168.56.254`  |

We'll assign Ubuntu a static IP in the next step, making sure it's outside the DHCP range to avoid address conflicts.

## 5. Set a static IP in Ubuntu

Start Ubuntu and check the network interfaces and Netplan configuration file:

```bash
ip -4 addr
ip link show
ls /etc/netplan/
```

In my VM:

- `enp0s3` - NAT (internet access)
- `enp0s8` - Host-only (lab communication)

We'll assign `192.168.56.10/24` to the Host-only interface. This address belongs to our lab network but is outside the DHCP range (`192.168.56.101 - 254`).

### Configure Netplan

Open the existing configuration file:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Update the configuration:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.56.10/24
```

Make sure the indentation is correct, as YAML relies on spaces.

Save the file using **Ctrl+O → Enter → Ctrl+X**.

Test the configuration:

```bash
sudo netplan try
```

Press **Enter** to accept the configuration if everything works.

Verify the Host-only IP:

```bash
ip -4 addr show enp0s8
```

Expected address: `192.168.56.10/24`.

> **Note:** Network interface names, Netplan filenames, and IP ranges may differ between systems. Check your own configuration before copying these values.

## 6. Verify connectivity and SSH

From the **Ubuntu Server**, check internet connectivity:

```bash
ping -c 4 google.com
```

From **Windows Terminal (Host Machine)**, test the Host-only network:

```powershell
ping 192.168.56.10
```

If the connection works, connect to Ubuntu through SSH:

```powershell
ssh socadmin@192.168.56.10
```

On your first SSH connection, you'll be asked to confirm the server's host key. Verify the fingerprint, then type `yes` to trust it and enter your Ubuntu account password.

Once connected, you should see a terminal prompt similar to:

```text
socadmin@wazuh-server:~$
```

**Result:** Ubuntu Server is installed, updated, and accessible from Windows through SSH. It's now ready for Wazuh installation.

> **Note:** The Ubuntu VM must be running for SSH and Wazuh to work. For everyday use, you can start the VM using **Start without GUI** (Headless Mode) in VirtualBox and manage it directly through Windows Terminal.
