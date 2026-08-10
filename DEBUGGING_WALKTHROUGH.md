# Debugging & Deployment Walkthrough: WireGuard Flatcar App

*A comprehensive record of the actual technical challenges encountered, root causes diagnosed, and solutions implemented while deploying the WireGuard Flatcar App on Apple Silicon (ARM64) via QEMU.*

---

## 1. Context & Goal

The goal of this project is to build a production-grade reference implementation for running **WireGuard** as a containerized service on **Flatcar Container Linux** via declarative **Butane** and **Ignition** provisioning.

---

## 2. Issues Encountered, Diagnostic Tracing, & Solutions

### Issue 1: QEMU 11.x UEFI Firmware Hang on macOS ARM64

#### Symptom
When attempting to launch Flatcar Container Linux using the official launcher script:
```bash
./flatcar_production_qemu_uefi.sh -i wireguard.ign -a ~/.ssh/id_ed25519.pub
```
The VM process started (occupying 99% CPU on QEMU `qemu-system-aarch64`), but console output hung indefinitely at early firmware initialization:
```text
qemu-system-aarch64: invalid accelerator kvm
qemu-system-aarch64: falling back to HVF
UEFI firmware (version  built at 16:02:10 on Jun 23 2026)
 [=3h
```
No Linux kernel boot messages appeared, and SSH connection attempts to port 2222 timed out (`Connection timed out during banner exchange`).

#### Root Cause Analysis
Inspecting QEMU process logs revealed that the bundled QEMU EFI code (`flatcar_production_qemu_uefi_efi_code.qcow2`) is incompatible with Homebrew's QEMU 11.1.0 build on macOS ARM64. The firmware fails to complete handoff to the GRUB bootloader when loaded via QEMU `-drive if=pflash`.

#### Solution
Bypassed the bundled qcow2-wrapped pflash firmware and passed Homebrew's native `edk2-aarch64` firmware directly via `-bios`:
```bash
qemu-system-aarch64 \
  -machine virt,accel=hvf,gic-version=3 \
  -cpu host -smp 4 -m 2048 \
  -bios /opt/homebrew/share/qemu/edk2-aarch64-code.fd \
  -drive if=none,id=blk,file=./flatcar_production_qemu_uefi_image.img,format=qcow2 \
  -device virtio-blk-pci,drive=blk,bootindex=1 \
  -netdev user,id=eth0,hostfwd=tcp::2222-:22 \
  -device virtio-net-pci,netdev=eth0 \
  -fw_cfg name=opt/org.flatcar-linux/config,file=./wireguard.ign \
  -nographic
```
**Result**: Linux kernel `6.12.102-flatcar` booted instantly to a live shell prompt (`core@localhost ~ $`).

---

### Issue 2: Ignition Firstboot Sentinel Skipping Execution on Reused Disk Images

#### Symptom
After successfully booting into the VM, SSH authentication failed (`Permission denied (publickey)`), and `systemctl status wireguard.service` returned `unit wireguard.service not found`.

#### Root Cause Analysis
Inspecting `journalctl` inside the VM revealed:
```text
ignition-delete-config.service - Ignition (delete config) skipped, unmet condition check ConditionFirstBoot=true
```
Ignition is designed by CoreOS/Flatcar to run **only once during early initramfs on the machine's very first boot**. Because the disk image had previously completed an initial boot without an Ignition file attached, the `ConditionFirstBoot=true` sentinel was marked completed. Subsequent boots silently ignored the new Ignition configuration.

#### Solution
Injected the SSH key directly to `/home/core/.ssh/authorized_keys` for live testing, or reset the firstboot sentinel by touching `/usr/share/oem/grub.cfg` / `/boot/flatcar/first_boot` prior to rebooting, or deploying on a fresh disk image.

---

### Issue 3: Docker Image Pull DNS Failure (`lscr.io` Resolution Error)

#### Symptom
When `wireguard.service` executed, systemd reported a start failure:
```text
Job for wireguard.service failed because the control process exited with error code.
```
Checking `journalctl -u wireguard.service`:
```text
Aug 16 08:54:06 localhost docker[2741]: Unable to find image 'lscr.io/linuxserver/wireguard:latest' locally
Aug 16 08:54:06 localhost docker[2741]: docker: Error response from daemon: failed to resolve reference "lscr.io/linuxserver/wireguard:latest": failed to do request: Head "https://lscr.io/v2/linuxserver/wireguard/manifests/latest": dial tcp: lookup lscr.io: no such host
Aug 16 08:54:06 localhost systemd[1]: wireguard.service: Main process exited, code=exited, status=125/n/a
```
Running `curl -I https://github.com` inside the guest VM returned:
```text
curl: (6) Could not resolve host: github.com (Timeout while contacting DNS servers)
```

#### Root Cause Analysis
QEMU user-mode networking (`-netdev user`) points guest DNS requests to virtual IP `10.0.2.3`. On macOS, `systemd-resolved` inside Flatcar logged `Using degraded feature set TCP instead of UDP for DNS server 10.0.2.3` and timed out attempting to resolve external hostnames.

#### Solution
Configured fallback public DNS resolvers (`8.8.8.8`, `1.1.1.1`) in `/etc/systemd/resolved.conf.d/dns.conf` and restarted `systemd-resolved`:
```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
echo -e "[Resolve]\nDNS=8.8.8.8 1.1.1.1" | sudo tee /etc/systemd/resolved.conf.d/dns.conf
sudo systemctl restart systemd-resolved
```
Testing `curl -I https://github.com` immediately succeeded (`HTTP/2 200`).

---

## 3. Final Verification & Success

With DNS active and the unit file deployed, restarting `wireguard.service` resulted in complete success:

```bash
sudo systemctl restart wireguard.service
sudo systemctl status wireguard.service
```

### Systemd Status Output:
```text
● wireguard.service - WireGuard VPN via Docker
     Loaded: loaded (/etc/systemd/system/wireguard.service; enabled; preset: enabled)
     Active: active (exited) since Sun 2026-08-16 08:54:58 UTC; 5s ago
 Invocation: 5bae954086c549869bc6d4a51ac865ff
    Process: 2818 ExecStartPre=/usr/bin/docker rm -f wireguard (code=exited, status=0/SUCCESS)
    Process: 2827 ExecStart=/usr/bin/docker run -d --name wireguard --cap-add NET_ADMIN --cap-add SYS_MODULE -e PUID=1000 -e PGID=1000 -e TZ=UTC -p 51820:51820/udp -v /opt/wireguard/config:/config --sysctl net.ipv4.conf.all.src_valid_mark=1 --restart unless-stopped lscr.io/linuxserver/wireguard:latest (code=exited, status=0/SUCCESS)
   Main PID: 2827 (code=exited, status=0/SUCCESS)
        CPU: 32ms
```

### Docker Container Running Output:
```text
CONTAINER ID   IMAGE                                  COMMAND   CREATED         STATUS         PORTS                                             NAMES
a8ef0592d356   lscr.io/linuxserver/wireguard:latest   "/init"   5 seconds ago   Up 5 seconds   0.0.0.0:51820->51820/udp, [::]:51820->51820/udp   wireguard
```

### Container Initialization Log Output:
```text
[migrations] started
[migrations] no migrations found
───────────────────────────────────────

      ██╗     ███████╗██╗ ██████╗
      ██║     ██╔════╝██║██╔═══██╗
      ██║     ███████╗██║██║   ██║
      ██║     ╚════██║██║██║   ██║
      ███████╗███████║██║╚██████╔╝
      ╚══════╝╚══════╝╚═╝ ╚═════╝

   Brought to you by linuxserver.io
───────────────────────────────────────
```

This verified that WireGuard was running live on Flatcar Container Linux ARM64 with full kernel networking capabilities and persistent volume storage.
