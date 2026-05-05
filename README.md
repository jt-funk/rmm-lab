# rmm-lab
TacticalRMM deployment on a QEMU/KVM Debian VM on an Arch Linux host. Documents NAT networking, port forwarding, and local access without external exposure.

## VM Launch

Bash function in `.bashrc`:

```bash
deb-rmm() {
    qemu-system-x86_64 \
    -enable-kvm \
    -m 4096 \
    -cpu host \
    -M q35 \
    -smp 2 \
    -hda /hdd/vm-data/debian-rmm.qcow2 \
    -netdev user,id=inet0,hostfwd=tcp:127.0.0.1:443-:443,hostfwd=tcp:127.0.0.1:80-:80,hostfwd=tcp:127.0.0.1:4222-:4222 \
    -device virtio-net-pci,netdev=inet0 \
    -vga virtio \
    -display sdl
}

alias debian-rmm="deb-rmm"
```

QEMU user-mode NAT gives the VM outbound internet but the host cannot reach the VM directly. The `hostfwd` entries forward host ports to VM ports:

| Host | VM | Service |
|---|---|---|
| 127.0.0.1:443 | :443 | HTTPS / RMM frontend |
| 127.0.0.1:80 | :80 | HTTP |
| 127.0.0.1:4222 | :4222 | NATS (agent communication) |

## Port Binding on Linux

By default Linux restricts binding ports below 1024 to root. QEMU's `hostfwd` inherits this restriction, so forwarding to ports 80 and 443 requires a fix. Running QEMU as root is not the answer — it gives the entire VM process root on the host.

The clean solution is lowering the unprivileged port floor via sysctl:

```bash
echo "net.ipv4.ip_unprivileged_port_start=80" | sudo tee /etc/sysctl.d/99-unprivileged-ports.conf
sudo sysctl --system
```

This allows any process running as your user to bind ports 80 and above. On a single-user machine this is a reasonable tradeoff. QEMU can then bind 443 and 80 directly via hostfwd with no elevated privileges and no intermediate forwarding layer.

## /etc/hosts

TacticalRMM's frontend is hardcoded to its install domain. Since there's no external DNS routing to this machine, the subdomains need to resolve to localhost:

```
127.0.0.1 zip-host.duckdns.org rmm.zip-host.duckdns.org api.zip-host.duckdns.org mesh.zip-host.duckdns.org
```

## DuckDNS

DuckDNS update script keeping all subdomains pointed at the current public IP. Run on a cron or as a systemd timer:

```bash
curl -s "https://www.duckdns.org/update?domains=zip-host,rmm.zip-host,api.zip-host,mesh.zip-host&token=YOUR_TOKEN&ip="
```

## Installation

Clean Debian install, no display manager. TacticalRMM installed via the official install script:

```bash
wget -q https://raw.githubusercontent.com/amidaware/tacticalrmm/master/install.sh
chmod +x install.sh
sudo ./install.sh
```

## Recovering Access

If you forget credentials, list users and reset password from inside the VM:

```bash
cd /rmm/api
source env/bin/activate
python manage.py shell -c "from accounts.models import User; print([u.username for u in User.objects.all()])"
python manage.py changepassword <username>
```

---
 
## Deploying a Windows Agent — MOOKLAW-DC
 
### Network Driver
 
The Windows Server VM (MOOKLAW-DC) launched without a `-netdev` line in its QEMU command, so it had no network adapter at all. Adding virtio networking exposed a second problem: Windows Server Core has no virtio drivers built in and won't recognize the adapter without them.
 
Fix: download the virtio driver ISO on the Arch host and attach it as a CD-ROM:
 
```bash
wget https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso
```
 
Add to the Windows VM launch command:
 
```bash
-netdev user,id=net0 \
-device virtio-net-pci,netdev=net0 \
-cdrom /path/to/virtio-win.iso \
```
 
Then install the network driver from inside the VM:
 
```powershell
pnputil /add-driver D:\NetKVM\2k25\amd64\netkvm.inf /install
```
 
Verify with:
 
```powershell
Get-NetAdapter
Test-NetConnection -ComputerName 8.8.8.8 -Port 80
```
 
### Transferring the Agent Install Script
 
TacticalRMM generates an 80-line PowerShell install script. I decided to serve the file over HTTP from the Arch host and pull it from the VM.
 
On the Arch host:
 
```bash
cd /path/to/script
python -m http.server 9999
```
 
From inside the Windows VM — the QEMU host is always reachable at `10.0.2.2`:
 
```powershell
Invoke-WebRequest -Uri "http://10.0.2.2:9999/agent-install.ps1" -OutFile "C:\agent-install.ps1"
```
 
### /etc/hosts on Windows
 
The Windows VM needs a host entry for `10.0.2.2`, to resolve DNS requests for TacticalRMM domains:
 
```powershell
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "10.0.2.2 zip-host.duckdns.org rmm.zip-host.duckdns.org api.zip-host.duckdns.org mesh.zip-host.duckdns.org"
```
 
### Running the Agent Install
 
```powershell
Set-ExecutionPolicy Bypass -Scope Process
.\agent-install.ps1
```
 
### MeshCentral Remote Control
 
The mesh agent installs alongside TacticalRMM automatically. Remote control via MeshCentral works but requires enabling **Use Remote Keyboard Map** in the session settings — without it keystrokes don't register in Server Core. This is a known compatibility issue with headless Windows VMs.
 
### Verifying Agent Functionality
 
Once the agent appeared in TacticalRMM, a PowerShell script was created via Settings → Script Manager and run against MOOKLAW-DC to confirm end-to-end script execution:
 
```powershell
Get-ComputerInfo | Select-Object CsName, OsName, OsVersion, CsProcessors, CsTotalPhysicalMemory
```
 
Output returned successfully through TacticalRMM, confirming the full loop: agent connected, script pushed remotely, output retrieved.
 
