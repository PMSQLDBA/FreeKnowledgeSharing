# Technical SOP: Clear and Regenerate MAC Address After Cloning Ubuntu VM in VMware Workstation

## 1. Purpose

This SOP explains how to safely clear and regenerate the MAC address after cloning an Ubuntu virtual machine in VMware Workstation.

In this scenario, the original VM is:

```text
UbuntuPGSQL
```

The cloned VM is:

```text
UbuntuRemoteServer
```

After cloning a VM, it is very important to ensure the cloned VM has a unique MAC address, UUID, hostname, and network identity. If both original and cloned VMs use the same MAC address, it can cause IP conflicts, SSH connection issues, DHCP conflicts, hostname confusion, and PostgreSQL remote connectivity problems.

---

## 2. Scope

This SOP applies to:

```text
Platform       : VMware Workstation
Guest OS       : Ubuntu Linux
Original VM    : UbuntuPGSQL
Cloned VM      : UbuntuRemoteServer
Use Case       : PostgreSQL Lab / Remote Server Lab
Network Adapter: ethernet0
```

This procedure is useful after cloning any Linux VM, especially when preparing multiple PostgreSQL servers for replication, backup, PITR, testing, or remote connectivity labs.

---

## 3. Why MAC Address Cleanup Is Required

When a VM is cloned, VMware may copy the same network adapter configuration from the source VM to the cloned VM.

If the cloned VM keeps the same static MAC address as the original VM, both VMs may appear as the same machine on the network.

This can cause the following issues:

```text
1. Duplicate MAC address conflict
2. Same IP address assigned by DHCP
3. Network connection failure
4. SSH connection confusion
5. PostgreSQL remote connection issues
6. DNS or hostname resolution problems
7. Incorrect server identity in monitoring tools
8. Problems during PostgreSQL replication setup
```

For a clean cloned VM, the cloned server should have:

```text
Unique MAC address
Unique IP address
Unique hostname
Unique UUID
Unique PostgreSQL server identity
```

---

## 4. Pre-Checks Before Editing the VMX File

Before editing the `.vmx` file, make sure the cloned VM is fully powered off.

Do not edit the `.vmx` file while the VM is running or suspended.

### Correct state

```text
VM should be powered off completely.
```

### Avoid these states

```text
Suspended
Running
Paused
Snapshot restore in progress
```

---

## 5. Locate the VMX File

Go to the folder where the cloned VM is stored.

Example:

```text
D:\VMware\UbuntuRemoteServer\
```

Inside the folder, locate the VMware configuration file:

```text
UbuntuRemoteServer.vmx
```

Open this file using Notepad or Notepad++.

Recommended editor:

```text
Notepad++
```

---

## 6. Original Problematic MAC Configuration

Before cleanup, the cloned VM had the following MAC-related configuration:

```text
ethernet0.addressType = "static"
ethernet0.virtualDev = "e1000"
ethernet0.present = "TRUE"
ethernet0.address = "00:50:56:32:65:2B"
```

This means VMware was using a fixed static MAC address:

```text
00:50:56:32:65:2B
```

This is not recommended for a cloned VM unless you intentionally want to manually control the MAC address.

---

## 7. Correct MAC Cleanup Changes

To allow VMware to generate a new unique MAC address automatically, change the network configuration.

### Remove this line

```text
ethernet0.address = "00:50:56:32:65:2B"
```

### Change this line

From:

```text
ethernet0.addressType = "static"
```

To:

```text
ethernet0.addressType = "generated"
```

---

## 8. Final Correct Network Configuration

After cleanup, the network section should look like this:

```text
ethernet0.addressType = "generated"
ethernet0.virtualDev = "e1000"
ethernet0.present = "TRUE"
```

This is the correct configuration.

VMware will now generate a new MAC address automatically when the cloned VM starts.

---

## 9. Confirmed VMX Cleanup for UbuntuRemoteServer

The corrected `.vmx` file now contains:

```text
ethernet0.addressType = "generated"
ethernet0.virtualDev = "e1000"
ethernet0.present = "TRUE"
```

The old static MAC address line has been removed:

```text
ethernet0.address = "00:50:56:32:65:2B"
```

This confirms that the MAC cleanup has been completed properly.

---

## 10. UUID Cleanup

For cloned VMs, UUID cleanup is also recommended.

In the corrected `.vmx` file, the following UUID values are blank:

```text
uuid.bios = ""
uuid.location = ""
vc.uuid = ""
policy.vm.mvmtid = ""
```

This is acceptable.

When VMware starts the VM, it can regenerate required unique identifiers.

If VMware asks the following question during power-on:

```text
Did you move or copy this virtual machine?
```

Select:

```text
I copied it
```

This tells VMware to generate new unique identifiers for the cloned VM.

---

## 11. Save the VMX File

After making the changes:

```text
File → Save
```

Close the editor.

Do not change unrelated VMX settings unless required.

---

## 12. Power On the Cloned VM

Open VMware Workstation.

Start:

```text
UbuntuRemoteServer
```

If VMware asks:

```text
Did you move or copy this virtual machine?
```

Choose:

```text
I copied it
```

This is the correct option for a cloned VM.

---

## 13. Verify MAC Address Inside Ubuntu

After the VM boots, login to Ubuntu and run:

```bash
ip link show
```

Or check the specific network adapter:

```bash
ip link show ens33
```

You should see output similar to:

```text
link/ether 00:0c:29:xx:xx:xx
```

The value after `link/ether` is the MAC address.

Example:

```text
link/ether 00:0c:29:ab:12:34
```

This confirms that the VM has a MAC address assigned.

---

## 14. Verify IP Address

Run:

```bash
hostname -I
```

Or:

```bash
ip addr
```

Example output:

```text
192.168.0.145
```

If the IP address is different from the original VM, that is good.

The original and cloned VMs should not use the same IP address.

---

## 15. Restart Network Service If Required

If the VM does not get an IP address immediately, restart networking using one of the following methods.

### Option 1: Restart NetworkManager

```bash
sudo systemctl restart NetworkManager
```

### Option 2: Reboot the server

```bash
sudo reboot
```

After reboot, verify again:

```bash
hostname -I
ip link show
```

---

## 16. Update Hostname After Clone

Since the VM was cloned from `UbuntuPGSQL`, the hostname inside Ubuntu may still show the old name.

Check current hostname:

```bash
hostnamectl
```

Set the new hostname:

```bash
sudo hostnamectl set-hostname UbuntuRemoteServer
```

Verify:

```bash
hostname
hostnamectl
```

Expected output:

```text
UbuntuRemoteServer
```

---

## 17. Update `/etc/hosts`

Edit the hosts file:

```bash
sudo nano /etc/hosts
```

Find any old entry like:

```text
127.0.1.1 UbuntuPGSQL
```

Change it to:

```text
127.0.1.1 UbuntuRemoteServer
```

Final example:

```text
127.0.0.1 localhost
127.0.1.1 UbuntuRemoteServer
```

Save the file:

```text
CTRL + O
Enter
CTRL + X
```

Then reboot:

```bash
sudo reboot
```

---

## 18. Verify Final Server Identity

After reboot, run the following commands:

```bash
hostname
hostnamectl
hostname -I
ip link show
```

Expected results:

```text
Hostname should be UbuntuRemoteServer
IP address should be unique
MAC address should be unique
Network should be reachable
```

---

## 19. Optional: Verify VMware Network Adapter from GUI

You can also verify the MAC address from VMware Workstation.

Go to:

```text
Right-click UbuntuRemoteServer
→ Settings
→ Network Adapter
→ Advanced
```

The MAC address should be generated by VMware.

Do not manually reuse the MAC address from the original VM.

---

## 20. Optional: Release and Renew DHCP Lease

If the cloned VM still receives the old IP address, clear DHCP lease and request a new IP.

Run inside Ubuntu:

```bash
sudo dhclient -r
sudo dhclient
```

Then check IP:

```bash
hostname -I
```

If `dhclient` is not available, reboot the server:

```bash
sudo reboot
```

---

## 21. PostgreSQL Lab Considerations

If this Ubuntu VM will be used as a PostgreSQL remote server, verify the following after MAC and hostname cleanup:

```text
1. Server has unique hostname
2. Server has unique IP address
3. PostgreSQL listen_addresses is configured properly
4. pg_hba.conf allows remote client IP
5. Firewall allows port 5432
6. Windows client can ping UbuntuRemoteServer
7. Windows client can connect using psql or pgAdmin
```

Check PostgreSQL service:

```bash
sudo systemctl status postgresql
```

Check PostgreSQL port:

```bash
sudo ss -lntp | grep 5432
```

Check server IP:

```bash
hostname -I
```

Example remote connection from another machine:

```bash
psql -U postgres -d postgres -h <UbuntuRemoteServer_IP> -p 5432
```

Example:

```bash
psql -U postgres -d postgres -h 192.168.0.145 -p 5432
```

---

## 22. Final Checklist

| Step | Task                                          | Status      |
| ---- | --------------------------------------------- | ----------- |
| 1    | Power off cloned VM                           | Completed   |
| 2    | Open `UbuntuRemoteServer.vmx`                 | Completed   |
| 3    | Change `ethernet0.addressType` to `generated` | Completed   |
| 4    | Remove old `ethernet0.address` line           | Completed   |
| 5    | Clear UUID values                             | Completed   |
| 6    | Save `.vmx` file                              | Completed   |
| 7    | Power on cloned VM                            | Pending     |
| 8    | Select `I copied it` if prompted              | Pending     |
| 9    | Verify MAC using `ip link show`               | Pending     |
| 10   | Verify IP using `hostname -I`                 | Pending     |
| 11   | Change hostname to `UbuntuRemoteServer`       | Recommended |
| 12   | Update `/etc/hosts`                           | Recommended |
| 13   | Reboot VM                                     | Recommended |
| 14   | Test PostgreSQL remote connectivity           | Recommended |

---

## 23. Final Correct VMX Reference

The final MAC configuration should remain as follows:

```text
ethernet0.addressType = "generated"
ethernet0.virtualDev = "e1000"
ethernet0.present = "TRUE"
```

The following old static MAC line should not exist:

```text
ethernet0.address = "00:50:56:32:65:2B"
```

---

## 24. Troubleshooting

### Issue 1: VM still gets old IP address

Run:

```bash
sudo dhclient -r
sudo dhclient
hostname -I
```

Or reboot:

```bash
sudo reboot
```

---

### Issue 2: No network adapter shown inside Ubuntu

Run:

```bash
ip link show
```

Check if adapter exists as:

```text
ens33
ens160
eth0
```

If no adapter is visible, verify VMware settings:

```text
VM Settings → Network Adapter → Connected
VM Settings → Network Adapter → Connect at power on
```

---

### Issue 3: Cannot connect from Windows to Ubuntu

Check Ubuntu IP:

```bash
hostname -I
```

Check ping from Windows:

```powershell
ping <UbuntuRemoteServer_IP>
```

Example:

```powershell
ping 192.168.0.145
```

Check PostgreSQL port from Windows:

```powershell
Test-NetConnection 192.168.0.145 -Port 5432
```

---

### Issue 4: PostgreSQL port not listening remotely

Check PostgreSQL config:

```bash
sudo nano /etc/postgresql/*/main/postgresql.conf
```

Set:

```text
listen_addresses = '*'
```

Edit `pg_hba.conf`:

```bash
sudo nano /etc/postgresql/*/main/pg_hba.conf
```

Add allowed client network:

```text
host    all     all     192.168.0.0/24     scram-sha-256
```

Restart PostgreSQL:

```bash
sudo systemctl restart postgresql
```

---

## 25. Conclusion

The cloned VM `UbuntuRemoteServer` has been cleaned up correctly from the VMware MAC address perspective.

The key correction was changing:

```text
ethernet0.addressType = "static"
```

to:

```text
ethernet0.addressType = "generated"
```

And removing:

```text
ethernet0.address = "00:50:56:32:65:2B"
```

This allows VMware Workstation to generate a new unique MAC address for the cloned VM.

After booting the VM, complete the Linux-side cleanup by verifying the MAC address, confirming the IP address, changing the hostname, updating `/etc/hosts`, rebooting, and validating PostgreSQL remote connectivity.

You can use this as a GitHub `.md` article named:

```text
VMware_Ubuntu_Clone_MAC_Address_Cleanup_SOP.md
```
