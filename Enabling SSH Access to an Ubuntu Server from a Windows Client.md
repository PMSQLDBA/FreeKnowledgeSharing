# Standard Operating Procedure (SOP)
## Enabling SSH Access to an Ubuntu Server from a Windows Client

### Purpose
This SOP describes how to install, configure, and troubleshoot SSH access to an Ubuntu server from a Windows client.
---
# Prerequisites

* Ubuntu Server/Desktop
* Windows client with the OpenSSH client installed
* User account with `sudo` privileges
* Network connectivity between Windows and Ubuntu

---

# Step 1 – Verify the Ubuntu Server IP Address

On the Ubuntu server, run:

```bash
hostname -I
```

Example:

```text
192.168.0.143
```

Use the IPv4 address for SSH connections.

---

# Step 2 – Verify Network Connectivity from Windows

Open Command Prompt and run:

```cmd
ping 192.168.0.143
```

Expected result:

```text
Reply from 192.168.0.143...
```

If the ping fails, troubleshoot the network before proceeding.

---

# Step 3 – Install OpenSSH Server (if not installed)

Attempt to verify the SSH service:

```bash
sudo systemctl status ssh
```

If you receive:

```text
Unit ssh.service could not be found.
```

Install OpenSSH Server:

```bash
sudo apt update
sudo apt install openssh-server
```

Start and enable the service:

```bash
sudo systemctl enable --now ssh
```

---

# Step 4 – Verify SSH Service

Check the service status:

```bash
sudo systemctl status ssh
```

Expected:

```text
Active: active (running)
```

---

# Step 5 – Verify SSH is Listening

Run:

```bash
sudo ss -tlnp | grep :22
```

Expected output:

```text
LISTEN 0 128 0.0.0.0:22
```

or

```text
LISTEN 0 128 [::]:22
```

This confirms SSH is listening on port 22.

---

# Step 6 – Check Ubuntu Firewall (UFW)

Display firewall status:

```bash
sudo ufw status verbose
```

Example before fix:

```text
Status: active

5432/tcp ALLOW IN Anywhere
```

If SSH is not listed, allow it:

```bash
sudo ufw allow 22/tcp
```

or

```bash
sudo ufw allow OpenSSH
```

Verify:

```bash
sudo ufw status numbered
```

Expected:

```text
22/tcp     ALLOW IN Anywhere
5432/tcp   ALLOW IN Anywhere
```

---

# Step 7 – Connect from Windows

Open Command Prompt or PowerShell:

```cmd
ssh username@192.168.0.143
```

Example:

```cmd
ssh pmsqldba@192.168.0.143
```

Enter the user's password when prompted.

---

# Troubleshooting

## Error

```text
sudo: command not found
```

### Cause

Linux commands were executed from Windows Command Prompt.

### Resolution

SSH into the Ubuntu server first, then execute Linux commands.

---

## Error

```text
ssh: connect to host <IP> port 22: Connection timed out
```

### Possible Causes

* SSH server not installed
* SSH service not running
* Firewall blocking port 22
* Incorrect IP address
* Network connectivity issue

### Resolution

1. Verify the server IP:

   ```bash
   hostname -I
   ```

2. Verify the SSH service:

   ```bash
   sudo systemctl status ssh
   ```

3. Verify SSH is listening:

   ```bash
   sudo ss -tlnp | grep :22
   ```

4. Allow SSH through the firewall:

   ```bash
   sudo ufw allow OpenSSH
   ```

5. Retry the SSH connection.

---

# Validation Checklist

| Check          | Expected Result                                 |
| -------------- | ----------------------------------------------- |
| Server IP      | `hostname -I` displays the correct IPv4 address |
| Ping           | Windows successfully pings the Ubuntu server    |
| SSH Installed  | `systemctl status ssh` shows the service exists |
| SSH Running    | Status is `active (running)`                    |
| Port Listening | `ss -tlnp` shows port 22 in `LISTEN` state      |
| Firewall       | UFW allows `22/tcp` or `OpenSSH`                |
| Windows SSH    | Successful login using `ssh username@<IP>`      |

---

# Commands Summary

```bash
# Ubuntu

hostname -I

sudo apt update

sudo apt install openssh-server

sudo systemctl enable --now ssh

sudo systemctl status ssh

sudo ss -tlnp | grep :22

sudo ufw status verbose

sudo ufw allow OpenSSH

sudo ufw status numbered
```

```cmd
:: Windows

ping <Ubuntu_IP>

ssh <username>@<Ubuntu_IP>
```

## Outcome

Following this SOP ensures that:

* The OpenSSH Server is installed.
* The SSH service is enabled and running.
* Port 22 is listening.
* The Ubuntu firewall allows SSH traffic.
* Windows clients can successfully establish SSH connections to the Ubuntu server.
