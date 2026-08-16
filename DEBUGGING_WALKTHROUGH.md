# Developer Debugging Notes & Technical Walkthrough

Hey there! This walkthrough documents the real-world setup, boot issues, and solutions I discovered while building and deploying this **WireGuard Flatcar App** reference implementation on Apple Silicon (ARM64) with QEMU 11.1.

---

## The Big Picture

My goal was simple: take a clean Flatcar Container Linux machine, provision a containerized **WireGuard** VPN service declaratively using **Butane** (`wireguard.bu`) and **Ignition** (`wireguard.ign`), and verify that the container starts up cleanly with all necessary kernel networking capabilities (`NET_ADMIN`, `SYS_MODULE`).

Along the way, I hit three real friction points. Here is what happened, why each issue occurred, and how I solved or avoided them directly in my Butane configuration.

---

## 1. Upstream Issue: QEMU 11.1 UEFI Firmware Boot Freeze
*Status: Upstream bug identified; reported and under investigation with core maintainer `@Chewi` on Flatcar Discord.*

### What Happened
When booting Flatcar ARM64 on macOS using the official launcher script:
```bash
./flatcar_production_qemu_uefi.sh -i wireguard.ign -a ~/.ssh/id_ed25519.pub
```
The VM process started up in QEMU, but the console output froze forever at the early UEFI firmware screen:
```text
qemu-system-aarch64: falling back to HVF
UEFI firmware (version built at 16:02:10 on Jun 23 2026)
 [=3h
```

### My Investigation & Debugging
I ran QEMU with hardware debug logging (`-d guest_errors,unimp`) and caught the exact error QEMU throws right before freezing:
```text
pflash_write: Unimplemented flash cmd sequence (offset 0000000000000008, wcycle 0x0 cmd 0x0 value 0x7)
```

I isolated lines 317–321 of `flatcar_production_qemu_uefi.sh`:
```bash
-drive if=pflash,unit=0,file="${VM_PFLASH_RO}",format=qcow2,readonly=on \
-drive if=pflash,unit=1,file="${VM_PFLASH_RW}",format=qcow2
```
Flatcar's bundled EDK2 firmware build issues flash write commands (`pflash_write`) during early boot. Under QEMU 11.1 on macOS Hypervisor.framework (`hvf`), QEMU drops these write cycles, trapping the firmware in a loop before GRUB loads.

### The Fix
Bypassing pflash chip emulation and passing Homebrew's upstream EDK2 bios directly via `-bios` bypasses the trap completely:
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
**Result**: Linux kernel `6.12.102-flatcar` boots to a live prompt in **8 seconds**.

---

## 2. Architecture Insight: Reused Disk Images & Ignition One-Shot Execution
*Status: Expected Flatcar architecture behavior; avoided by booting a fresh image.*

### What Happened
On an existing disk image that had already been booted once, passing a new Ignition config resulted in SSH key authorization failing and `wireguard.service` not being created.

### Why It Happened
`journalctl` inside the VM showed:
```text
ignition-delete-config.service - Ignition (delete config) skipped, unmet condition check ConditionFirstBoot=true
```
By design, Ignition runs **only once** in `initramfs` on the machine's very first boot. Once complete, it writes a completion marker. Reusing an old, already-booted disk image skips Ignition entirely.

### How I Avoided It
Always start testing with a fresh, un-booted image copy (`bunzip2 -k flatcar_production_qemu_uefi_image.img.bz2`).

---

## 3. QEMU Networking Insight: DNS Resolution Timestamps
*Status: Handled and avoided directly inside `wireguard.bu`!*

### What Happened
When `wireguard.service` started on macOS QEMU user networking (`-netdev user`), systemd reported a start failure:
```text
docker: Error response from daemon: failed to resolve reference "lscr.io/linuxserver/wireguard:latest": dial tcp: lookup lscr.io: no such host
```

### Why It Happened & How I Avoided It in Butane
QEMU user-mode networking points guest DNS to `10.0.2.3`. On macOS host networks, `systemd-resolved` inside Flatcar logged `Using degraded feature set TCP instead of UDP for DNS server 10.0.2.3` and timed out attempting to resolve external container registries.

Could I avoid this in my Butane config? **Yes!**
By declaring `/etc/systemd/resolved.conf.d/dns.conf` directly inside `wireguard.bu`, Ignition provisions fallback public DNS servers (`8.8.8.8`, `1.1.1.1`) on boot:

```yaml
storage:
  files:
    - path: /etc/systemd/resolved.conf.d/dns.conf
      mode: 0644
      contents:
        inline: |
          [Resolve]
          DNS=8.8.8.8 1.1.1.1
```

With this addition to `wireguard.bu`, `systemd-resolved` immediately resolves `lscr.io`, Docker pulls the WireGuard image without errors, and the service starts up seamlessly!

---

## Verification & Live Container Output

With the `-bios` firmware flag and the updated `wireguard.bu` config, here is the live verification from the running Flatcar ARM64 instance:

```bash
ssh -p 2222 core@localhost
```

### Systemd Status (`wireguard.service`)
```text
● wireguard.service - WireGuard VPN via Docker
     Loaded: loaded (/etc/systemd/system/wireguard.service; enabled; preset: enabled)
     Active: active (exited) since Sun 2026-08-16 13:24:31 UTC; 5s ago
    Process: ExecStart=/usr/bin/docker run -d --name wireguard --cap-add NET_ADMIN --cap-add SYS_MODULE -e PUID=1000 -e PGID=1000 -e TZ=UTC -p 51820:51820/udp -v /opt/wireguard/config:/config --sysctl net.ipv4.conf.all.src_valid_mark=1 --restart unless-stopped lscr.io/linuxserver/wireguard:latest (code=exited, status=0/SUCCESS)
```

### Docker Container Running (`docker ps`)
```text
CONTAINER ID   IMAGE                                  COMMAND   CREATED         STATUS         PORTS                                             NAMES
17db99f01133   lscr.io/linuxserver/wireguard:latest   "/init"   6 seconds ago   Up 5 seconds   0.0.0.0:51820->51820/udp, [::]:51820->51820/udp   wireguard
```

### Container Initialization Logs (`docker logs wireguard`)
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
