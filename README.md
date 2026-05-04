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
