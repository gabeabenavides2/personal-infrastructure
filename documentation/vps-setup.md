# VPS Setup

## Overview

This document describes the process used to provision, configure, and secure a Linux Virtual Private Server (VPS).

The goal of the initial setup is to create a secure foundation that can later be used to host websites, applications, containers, and other services.

## Server Configuration

- **Provider:** OVHcloud
- **Operating System:** Ubuntu 24.04 LTS
- **vCPUs:** 2
- **Memory:** 4 GB RAM
- **Storage:** 40 GB
- **Remote Administration:** SSH
- **Firewall:** UFW

> IP addresses, passwords, private keys, and other sensitive information are intentionally excluded from this repository.

## 1. Initial SSH Connection

After the VPS was provisioned, the server could be accessed remotely using SSH (Secure Shell).

```bash
ssh ubuntu@<VPS_IP>
```

Where:

- `ssh` starts an SSH connection.
- `ubuntu` is the user account on the remote server.
- `<VPS_IP>` represents the public IPv4 address assigned to the VPS.

The first connection displays the server's host-key fingerprint and asks whether the host should be trusted.

After accepting the fingerprint, the server is added to the local machine's `known_hosts` file.

On the initial login, the temporary password provided by the VPS provider was used. Ubuntu then required the temporary password to be replaced with a new password.

## 2. Updating the Server

Before installing or configuring additional services, the operating system was updated to ensure the latest package versions and security patches were installed.

### Update the Package Index

```bash
sudo apt update
```

`sudo` stands for "superuser do". It is a command that lets an authorized user run a program with the security permissions of the system admin, or root user.

`apt` is Ubuntu's package manager. The `update` command downloads the latest package information from the configured software repositories.

This command does **not** upgrade the installed software. It only refreshes Ubuntu's information about which package versions are available.

### Upgrade Installed Packages

```bash
sudo apt upgrade -y
```

This installs available updates for packages already installed on the system.

- `upgrade` installs newer versions of installed packages.
- `-y` automatically answers "yes" to confirmation prompts.

The difference between the two commands is:

```text
apt update
    ↓
Refresh available package information

apt upgrade
    ↓
Install available package updates
```

### Reboot After Kernel Update

During the upgrade, a newer Linux kernel was installed. Because the currently running kernel cannot simply be replaced while the system is running, the VPS was rebooted:

```bash
sudo reboot
```

The SSH connection closes while the server restarts. After the VPS becomes available again (wait about 1 minute), reconnect using:

```bash
ssh ubuntu@<VPS_IP>
```

### Verify the Running Kernel

The currently running Linux kernel can be checked with:

```bash
uname -r
```

This verifies that the VPS is running the updated kernel after the reboot.

## 3. SSH Key Authentication

Initially, the VPS was accessed using a username and password. SSH key authentication was configured to provide a more secure authentication method.

### How SSH Key Authentication Works

```text
MAC / LOCAL COMPUTER                         VPS / SERVER
────────────────────                         ────────────
        │                                         │
        │  "I'd like to log in as ubuntu"         │
        │ ──────────────────────────────────────> │
        │                                         │
        │      Authentication challenge           │
        │ <────────────────────────────────────── │
        │                                         │
        │  🔒 Private Key                         │
        │  ~/.ssh/id_ed25519                      │
        │         │                               │
        │         └─ Creates cryptographic        │
        │            signature                   │
        │                                         │
        │      Cryptographic signature            │
        │ ──────────────────────────────────────> │
        │                                         │
        │                         🔑 Public Key    │
        │                         authorized_keys │
        │                              │          │
        │                              ▼          │
        │                       Verifies signature│
        │                              │          │
        │                              ▼          │
        │                             ✅          │
        │                                         │
        │          ACCESS GRANTED                 │
        │ <────────────────────────────────────── │
        │                                         │
```

SSH key authentication uses asymmetric cryptography. A key pair consists of:

- **Private key** — remains on the local computer and must never be shared.
- **Public key** — can safely be copied to servers that the user is authorized to access.

The server can verify that the connecting computer possesses the corresponding private key without the private key ever being transmitted to the server.

### Check for Existing SSH Keys

On the local computer, SSH keys are typically stored in:

```text
~/.ssh/
```

The directory can be inspected with:

```bash
ls -la ~/.ssh
```

Common files include:

```text
~/.ssh/
├── id_ed25519       # Private key
├── id_ed25519.pub   # Public key
└── known_hosts      # Previously trusted SSH servers
```

The `known_hosts` file serves a different purpose from the authentication key pair. It allows the local SSH client to remember the identities of servers it has previously connected to.

### Generate an Ed25519 Key Pair

If an SSH key pair did not already exist, a new Ed25519 key pair was generated:

```bash
ssh-keygen -t ed25519
```

Where:

- `ssh-keygen` generates and manages SSH authentication keys.
- `-t` specifies the type of key to generate.
- `ed25519` specifies the Ed25519 public-key algorithm.

The default key locations were used:

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

A passphrase was also added to protect the private key.

> The private key (`id_ed25519`) should never be shared, uploaded to a server, or committed to a Git repository.

### Copy the Public Key to the VPS

The public key was installed on the VPS using:

```bash
ssh-copy-id ubuntu@<VPS_IP>
```

This copies the local public key to the remote user's SSH configuration.

On the VPS, authorized public keys are stored in:

```text
/home/ubuntu/.ssh/authorized_keys
```

The resulting relationship is:

```text
Local Computer                         VPS

~/.ssh/id_ed25519                     ~/.ssh/authorized_keys
Private Key                           Public Key
     |                                     |
     |    Cryptographic proof              |
     +------------------------------------>|
                                           |
                                      Verification
                                           |
                                      Access granted
```

The private key remains on the local computer throughout authentication.

### Test Key Authentication

After installing the public key, SSH access was tested again:

```bash
ssh ubuntu@<VPS_IP>
```

Instead of requesting the VPS account password, the local SSH client used the private key for authentication.

If the private key is protected with a passphrase, the local computer may request that passphrase to unlock the key.

The passphrase and VPS password serve different purposes:

- **VPS password** authenticates directly against the remote user account.
- **SSH key passphrase** protects the private key stored on the local computer.

## 4. SSH Hardening

After confirming that SSH key authentication worked correctly, password-based SSH authentication was disabled.

This prevents an attacker from attempting to access the VPS by repeatedly guessing the `ubuntu` user's password.

With password authentication disabled, SSH access requires possession of an authorized private key.

### Verify Current SSH Authentication Settings

The effective SSH server configuration can be inspected using:

```bash
sudo sshd -T | grep -E 'passwordauthentication|pubkeyauthentication'
```

The desired configuration is:

```text
pubkeyauthentication yes
passwordauthentication no
```

- `PubkeyAuthentication yes` allows SSH key authentication.
- `PasswordAuthentication no` prevents SSH login using account passwords.

### Configure SSH Authentication

SSH configuration on Ubuntu can be split across the main SSH configuration file and additional configuration files located in `sshd_config.d`.

The main SSH configuration file was edited using:

```bash
sudo nano /etc/ssh/sshd_config
```

Password authentication was disabled by setting:

```text
PasswordAuthentication no
```

The effective SSH configuration was then checked:

```bash
sudo sshd -T | grep -E 'passwordauthentication|pubkeyauthentication'
```

Despite changing the main configuration, SSH still reported:

```text
pubkeyauthentication yes
passwordauthentication yes
```

This indicated that another SSH configuration file was affecting the effective configuration.

To locate all occurrences of the setting, the following command was used:

```bash
sudo grep -Rni "PasswordAuthentication" /etc/ssh/
```

This revealed additional SSH configuration under:

```text
/etc/ssh/sshd_config.d/
```

The cloud-init configuration file was then edited:

```bash
sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf
```

The setting was changed to:

```text
PasswordAuthentication no
```

### Validate and Apply the Configuration

Before restarting SSH, the configuration was checked for syntax errors:

```bash
sudo sshd -t
```

The SSH service was then restarted:

```bash
sudo systemctl restart ssh
```

Finally, the effective configuration was verified again:

```bash
sudo sshd -T | grep -E 'passwordauthentication|pubkeyauthentication'
```

The expected result was:

```text
pubkeyauthentication yes
passwordauthentication no
```

## 5. Firewall Configuration with UFW

Because the VPS is publicly accessible over the internet, a firewall was configured to control which incoming network connections are allowed to reach the server.

Ubuntu includes **UFW (Uncomplicated Firewall)**, which provides a simpler interface for managing Linux firewall rules.

### Check Firewall Status

The initial firewall status was checked using:

```bash
sudo ufw status
```

Initially, UFW reported:

```text
Status: inactive
```

Before enabling the firewall, rules were created for services that needed to remain accessible.

### Allow SSH

SSH uses TCP port **22** by default.

Because SSH is used to remotely administer the VPS, this port must remain accessible:

```bash
sudo ufw allow OpenSSH
```

`OpenSSH` is an application profile that allows the ports required by the OpenSSH server.

This is equivalent to allowing SSH traffic on TCP port 22.

> SSH access should be allowed before enabling UFW. Enabling the firewall without allowing SSH first could prevent remote access to the VPS.

### Allow HTTP

Standard unencrypted web traffic uses TCP port **80**.

```bash
sudo ufw allow 80/tcp
```

Port 80 allows clients to make HTTP requests to the web server.

```text
Browser ─── HTTP :80 ───> VPS
```

Even after HTTPS is configured, port 80 can remain available so HTTP requests can be redirected to HTTPS.

### Allow HTTPS

Encrypted web traffic uses TCP port **443**.

```bash
sudo ufw allow 443/tcp
```

Port 443 allows clients to establish HTTPS connections with the web server.

```text
Browser ─── HTTPS :443 ───> VPS
```

HTTPS encrypts traffic between the client and the web server using TLS.

### Enable the Firewall

After the required rules were configured, UFW was enabled:

```bash
sudo ufw enable
```

UFW warned that enabling the firewall could disrupt existing SSH connections. Because the OpenSSH rule had already been created, the firewall could be safely enabled.

### Verify Firewall Rules

The active firewall configuration was inspected using:

```bash
sudo ufw status verbose
```

The important rules were:

```text
22/tcp (OpenSSH)    ALLOW IN    Anywhere
80/tcp              ALLOW IN    Anywhere
443/tcp             ALLOW IN    Anywhere
```

Equivalent IPv6 rules were also created automatically.

### Current Network Exposure

The firewall now follows a default-deny approach for incoming connections.

```text
                         INTERNET
                            │
                ┌───────────┼───────────┐
                │           │           │
              TCP 22      TCP 80      TCP 443
                │           │           │
               SSH         HTTP        HTTPS
                │           │           │
                └───────────┼───────────┘
                            ▼
                           VPS

                Other incoming ports
                         ✕ BLOCKED
```

This means that services running on other ports are not automatically reachable from the public internet.

Outgoing connections from the VPS remain allowed by default.