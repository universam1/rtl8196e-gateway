# Remote Home Assistant over WireGuard

Use a gateway that sits behind a home router with a Home Assistant (ZHA) that
runs on a VPS on the internet. The radio is reached over a WireGuard tunnel
that the **gateway dials out**, so the home network needs no open port and no
second device.

> **Status: design and build notes, not a tested recipe.** The kernel and tools
> below were built and the build checks passed. Nothing was flashed to a
> gateway, and no tunnel has been brought up yet. Treat every step as
> unverified until you have run it, and keep a [backup](../3-Main-SoC-Realtek-RTL8196E/30-Backup-Restore/README.md)
> first.

## Why a tunnel is needed

The UART bridge (`rtl8196e-uart-bridge`) only **listens** on TCP port 8888
and accepts a single client. The driver has `listen_sock` and an accept worker
and no `kernel_connect`, so it has no client or reverse mode. A server that is
not on the gateway's LAN therefore cannot reach it unless something carries
the connection in. Port 8888 is also unauthenticated and unencrypted, so it
must not be exposed to the internet. See
[SECURITY.md](../3-Main-SoC-Realtek-RTL8196E/32-Kernel/files-7.2/drivers/net/rtl8196e-uart-bridge/SECURITY.md).

## Options that were considered

| Option | Verdict |
| --- | --- |
| Zigbee2MQTT next to the gateway, MQTT to the VPS | Most robust, but needs a second device |
| Reverse SSH tunnel (`dbclient -R`) | Needs no firmware change, but was found fragile in practice |
| **WireGuard on the gateway** | Chosen. Needs a custom kernel |
| New gateway-side Zigbee stack that speaks MQTT | Very large effort. Not scoped |
| Tailscale on the gateway | Not checked. A Go client on this MIPS-I core is probably impractical |

Zigbee2MQTT documents that a coordinator behind a serial-over-TCP proxy is
**not supported over WiFi, WAN or VPN**, because the serial protocols tolerate
neither packet loss nor latency. That warning applies to any option that
carries port 8888 across the internet, including this one. Expect occasional
reconnects.

The Zigbee2MQTT `ember` adapter and ZHA both speak EZSP, which is what the NCP
firmware here provides, so no new adapter or protocol work is needed. ZiGate
is a different radio family and does not apply.

## Architecture

```text
Gateway (WG client, dials out)  --UDP-->  VPS host (WG server, public UDP port)
  wg0  <gateway-tunnel-ip>                  wg0  <vps-tunnel-ip>
  bridge listens on :8888                   Home Assistant container
                                            -> socket://<gateway-tunnel-ip>:8888
```

- The gateway sets `PersistentKeepalive = 25`, so the NAT mapping at the home
  router stays open and no inbound rule is needed there.
- Home Assistant stays on its Docker bridge network. Docker NATs its traffic
  out through the VPS host, which routes it into `wg0`. This should need no
  `docker-compose.yaml` change, but it has **not been tested** from inside the
  container.
- Use a tunnel subnet that does not overlap your LAN, your Docker networks or
  the Tailscale range `100.64.0.0/10`.
- ZHA must be the only client of port 8888. Stop any other client first.

### If you use Tailscale instead

[TailGuard](https://github.com/juhovh/tailguard) is a Docker container that
joins a tailnet and terminates WireGuard, so a plain WireGuard peer such as
the gateway can reach tailnet nodes. Its `examples/simple-bridge.sh` gives
connecting WireGuard clients access to the tailnet. It makes sense when Home
Assistant is a different tailnet node from the WireGuard host. If Home
Assistant runs on the same VPS, plain kernel WireGuard on the VPS is simpler.
The TailGuard README was read, but its scripts were not, so check them before
use.

## Part 1: Build the kernel and tools

### 1.1 Enable WireGuard in the kernel config

The shipped kernels leave WireGuard out on purpose (see
[the WireGuard README](../3-Main-SoC-Realtek-RTL8196E/34-Userdata/wireguard/README.md)
for the ~5 % Ethernet TX cost). This branch sets, in
`3-Main-SoC-Realtek-RTL8196E/32-Kernel/config-6.18-realtek.txt`:

```text
CONFIG_WIREGUARD=y
```

Zigbee traffic is small, so the throughput cost does not matter for this use.
`CONFIG_WIREGUARD=m` with `CONFIG_MODULES=y` avoids the cost, but a loaded
module has not been measured and the shipped configs disable modules. The 7.2
config is unchanged.

### 1.2 Build in Docker on an arm64 Mac

Three problems came up building with the repo's Docker environment on Apple
Silicon, and each has a workaround.

**Docker VM size.** Colima defaults to 2 CPUs and 2 GiB. Resize it (this
restarts every container):

```bash
colima stop
colima start --cpu 10 --memory 16
```

**The Dockerfile does not build on arm64.** It enables `dpkg --add-architecture
i386`, and Ubuntu's `ports.ubuntu.com` has no i386 indexes, so `apt-get update`
fails with 404s. The Silabs tools it installs afterwards (ARM GCC, `slc-cli`,
Commander) are x86_64-only. None of that is needed for a kernel or `wg` build.
`1-Build-Environment/Dockerfile.lexra-only` is a copy of the Dockerfile that
keeps only these parts. Build it from `1-Build-Environment`:

- the apt package list, without the i386 packages, `openjdk-21-jre-headless`
  and `libgl1`;
- the `builder` user, crosstool-ng and the Lexra toolchain steps (about
  30 minutes);
- the Realtek tools step, the `.bashrc` step and the entrypoint.

```bash
docker build -f Dockerfile.lexra-only -t rtl8196e-gateway-builder .
```

This yields a 1.3 GB image. It cannot build the EFR32 firmware.

**The kernel source needs a case-sensitive filesystem.** The default macOS
volume is case-insensitive, and the kernel tree has file names that differ
only by case. `tar` hits about 60 collisions and leaves files with mode
`----------`, so the build fails with "Permission denied". Build on a Docker
volume instead:

```bash
docker volume create rtl-ws
docker run --rm --user root -v rtl-ws:/workspace -v "$PWD:/src:ro" \
  rtl8196e-gateway-builder bash -c \
  'rsync -a --exclude=.git --exclude=/x-tools \
     --exclude=/1-Build-Environment/11-realtek-tools/bin \
     --exclude="linux-*" /src/ /workspace/ && chown -R 1000:1000 /workspace'
```

If a failed run left `linux-6.18.54` and its tarball in
`32-Kernel/` on the Mac, delete them first.

### 1.3 Build

```bash
docker run --rm -v rtl-ws:/workspace rtl8196e-gateway-builder \
  /workspace/3-Main-SoC-Realtek-RTL8196E/32-Kernel/build_kernel.sh

docker run --rm -v rtl-ws:/workspace rtl8196e-gateway-builder bash -c \
  'cd /workspace/3-Main-SoC-Realtek-RTL8196E/34-Userdata/wireguard && sh build_wireguard.sh'
```

Copy the results out of the volume:

```bash
mkdir -p out && docker run --rm -v rtl-ws:/workspace -v "$PWD/out:/out" \
  rtl8196e-gateway-builder bash -c \
  'W=/workspace/3-Main-SoC-Realtek-RTL8196E
   cp $W/32-Kernel/kernel-img/lidl/kernel-6.18.img /out/kernel-6.18-wireguard.img
   cp -r $W/34-Userdata/wireguard/optional /out/optional'
```

Check the result before flashing:

- the build log ends with `Image ready: …/kernel-6.18.img`;
- `linux-6.18-rtl8196e/.config` in the volume contains `CONFIG_WIREGUARD=y`;
- `wg` and `wg-link` are static 32-bit MIPS ELF files.

On the build run, the image was 1.47 MB, about 29 KB larger than the shipped
one. For another board or the 7.2 line, set `BOARD=` or `KERNEL=` as in
`build_kernel.sh --help`, and enable WireGuard in the matching config.

## Part 2: VPS

Requirements: a public IP, a free UDP port (51820 in this guide), and a kernel
with WireGuard. Most current distribution kernels include it. Check with
`modinfo wireguard`.

1. Install `wireguard-tools`.
2. Generate keys on the VPS and on a trusted machine for the gateway:

   ```bash
   umask 077
   wg genkey | tee vps.key | wg pubkey > vps.pub
   wg genkey | tee gw.key  | wg pubkey > gw.pub
   ```

3. Write `/etc/wireguard/wg0.conf`:

   ```ini
   [Interface]
   Address    = <vps-tunnel-ip>/24
   ListenPort = 51820
   PrivateKey = <vps private key>

   [Peer]
   # the gateway
   PublicKey  = <gateway public key>
   AllowedIPs = <gateway-tunnel-ip>/32
   ```

4. Open UDP 51820 in the cloud provider's network rules (for example the
   security list or NSG on Oracle Cloud) and in the host firewall, if one is
   active.
5. `systemctl enable --now wg-quick@wg0`.

The VPS is the only side with a fixed endpoint, so the gateway peer carries no
`Endpoint` line here. The gateway's first packet teaches the VPS where it is.

## Part 3: Gateway

Every step in this part changes the gateway. Do it only after a backup.

1. **Back up** with `backup_gateway.sh`.
2. **Flash the kernel.** This reboots the gateway. A bad kernel can leave it
   unbootable, and recovery needs the serial adapter and the bootloader's
   TFTP mode.

   ```bash
   ./3-Main-SoC-Realtek-RTL8196E/flash_remote.sh -y kernel <gateway-ip>
   ```

   `flash_kernel.sh -i <file>` flashes an explicit image such as
   `kernel-6.18-wireguard.img`. If you use `flash_remote.sh`, check which image
   it picks up.
3. **Install the tools.** `/userdata` is writable JFFS2, so this survives a
   reboot but not a userdata reflash. Use `scp -O` (dropbear ships no sftp):

   ```bash
   GW=<gateway-ip>
   scp -O -r optional/etc/* root@$GW:/userdata/etc/
   scp -O optional/usr/bin/wg             root@$GW:/userdata/usr/bin/
   scp -O optional/usr/sbin/wg-link       root@$GW:/userdata/usr/sbin/
   scp -O optional/usr/sbin/wireguardctl  root@$GW:/userdata/usr/sbin/
   ssh root@$GW 'chmod +x /userdata/usr/bin/wg /userdata/usr/sbin/wg-link \
     /userdata/usr/sbin/wireguardctl /userdata/etc/init.d/S60wireguard'
   ```

4. **Configure.** `wireguardctl genkey` writes `wg0.conf` with mode 0600. Then
   edit `/userdata/etc/wireguard/wg0.conf`:

   ```ini
   [Interface]
   PrivateKey = <gateway private key>
   ListenPort = 51820

   [Peer]
   PublicKey           = <vps public key>
   AllowedIPs          = <vps-tunnel-ip>/32
   Endpoint            = <vps-hostname>:51820
   PersistentKeepalive = 25
   ```

   In `/userdata/etc/wireguard.conf` set `ENABLED=1`, `ADDRESS=<gateway-tunnel-ip>/24`
   and `ROUTES=<vps-tunnel-ip>/32`. Only the listed routes are installed, so
   the tunnel cannot replace the default route.
5. **Start** with `/userdata/etc/init.d/S60wireguard start`, then check
   `wg show` on both ends for a recent handshake.

If there is no handshake, check the endpoint hostname resolves from the
gateway, that UDP 51820 is open on the VPS, and that both public keys match.

## Part 4: Home Assistant (ZHA)

1. Flash NCP firmware if it is not already installed
   (`./flash_efr32.sh -y -g <gateway-ip> ncp`, see
   [Choose a radio mode](radio-options.md)).
2. From the Home Assistant container, test the path first:
   `nc -zv <gateway-tunnel-ip> 8888`.
3. Add the ZHA integration, radio type **EmberZNet**, serial device path
   `socket://<gateway-tunnel-ip>:8888`.

ZHA and Zigbee2MQTT are alternatives. Moving from Zigbee2MQTT to ZHA means
re-pairing devices unless a backup import works, which was not checked.
Replacing a radio also changes its IEEE address; read the
[troubleshooting note](troubleshooting.md) before swapping gateways.

## Security notes

- Port 8888 listens on all interfaces, including the LAN, and the tunnel does
  not change that. Binding it to the tunnel or loopback is covered in the
  bridge driver's `SECURITY.md`.
- Keep `AllowedIPs` to single `/32` addresses on both sides so the tunnel
  carries nothing else.
- Keep private keys out of the repository and out of backups you share.
- A second client replaces the first, so only ZHA should connect.

## Open items

- Boot the custom kernel on a real gateway and confirm that Ethernet and the
  bridge still work. Only the build-time checks have run.
- Measure Ethernet TX and the tunnel's throughput.
- Test Docker bridge to `wg0` routing from inside the Home Assistant container.
- Decide whether `CONFIG_WIREGUARD=m` is worth the extra work.
- Test how ZHA behaves over the tunnel when the link drops and recovers.
- The `wireguardctl` and `S60wireguard` scripts have not been run on a
  gateway in this setup.
