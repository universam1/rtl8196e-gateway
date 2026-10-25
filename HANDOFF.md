# Handoff: remote Home Assistant over WireGuard

Working notes to continue this task in a new session. They are personal to
this fork's `wireguard-remote-home-assistant` branch. Do not include this file
in a pull request to upstream. Site details (hostnames, IPs, keys) are
deliberately absent: ask the user for them and never commit them.

## Goal

Home Assistant (ZHA) runs on a VPS on the internet. The Lidl Silvercrest
gateway sits behind a home router. Home Assistant must reach the gateway's
EZSP bridge (TCP 8888) with **no extra device at home** and no inbound port at
home. The gateway dials out over WireGuard.

The full design and steps are in
[docs/remote-home-assistant-wireguard.md](docs/remote-home-assistant-wireguard.md).
Read it first.

## Decisions already made

- The bridge only listens on 8888 and has no client mode (`rtl8196e-uart-bridge`
  has no `kernel_connect`), so a tunnel is required.
- Zigbee2MQTT is dropped in favour of **ZHA** (EmberZNet, `socket://<gateway-tunnel-ip>:8888`).
  Re-pairing devices is likely; a Z2M backup import was not checked.
- A reverse SSH tunnel was tried by the user and found fragile.
- Extra-device options (Z2M next to the gateway) were rejected.
- **WireGuard** is the choice, so the kernel needs `CONFIG_WIREGUARD=y`. This costs
  about 5 % Ethernet TX, which is irrelevant for Zigbee traffic.
- The VPS is an Oracle ARM host with Docker (compose-managed), Traefik in front of
  Home Assistant, and Tailscale already running. Home Assistant is on a Docker
  bridge network, not host networking. Its kernel has the WireGuard module.
  Plain kernel WireGuard on the VPS host was preferred over TailGuard because
  Home Assistant runs on the same host.
- Zigbee2MQTT's docs warn that serial-over-TCP across WAN or VPN is unsupported.
  Expect reconnects. This applies to ZHA as well.

## State

Done:
- Kernel 6.18.54 (Lidl) built with WireGuard, and `wg` / `wg-link` /
  `wireguardctl` built. Published as the pre-release
  `wireguard-kernel-6.18-r1` on the fork, with `SHA256SUMS`. Use it instead of
  rebuilding.
- Guide written and linked in `mkdocs.yml`.
- `1-Build-Environment/Dockerfile.lexra-only` for arm64 hosts.

Not done (everything touching hardware):
- The kernel was **never booted** and nothing was flashed to the gateway.
- No WireGuard keys exist. No tunnel was configured on either end.
- Nothing was changed on the VPS. `sudo` there needs a password, so the user
  runs privileged commands.
- Routing from the Home Assistant container (Docker bridge) into `wg0` is untested.

## Next steps

1. Confirm the gateway is a Lidl board and the NCP firmware is installed.
2. Run `backup_gateway.sh`. Get the user's go-ahead before flashing anything.
3. Flash the release kernel (this reboots the gateway; recovery needs the
   serial adapter and the bootloader's TFTP mode). Check Ethernet and port 8888 still work.
4. Generate the key pairs (keep them out of git), write the VPS and gateway configs,
   and open the WireGuard UDP port in the cloud provider's network rules.
5. Install the tools on the gateway, start the tunnel, check the handshake.
6. Test `nc -zv <gateway-tunnel-ip> 8888` from inside the Home Assistant
   container, then add ZHA.

## Pitfalls found

- On macOS, kernel source extraction fails (case-insensitive filesystem, about 60
  tar collisions). Build in a Docker volume, not a bind mount.
- The repo's Dockerfile fails on arm64 (i386 package indexes 404; Silabs tools are
  x86_64-only). Use `Dockerfile.lexra-only`.
- The Colima default VM (2 CPUs, 2 GiB) is too small. It was resized to 10 CPUs
  and 16 GiB on the original machine.
- In zsh with `noclobber`, `>` refuses to overwrite an existing file. Use `>|`.
- `scp` to the gateway needs `-O` (dropbear has no sftp-server).

## Working rules from the user

- Never add co-authored-by tags or other AI attribution to commits or PR text.
- Prefer Go when writing new code, unless another language is clearly better.
- Confirm before hard-to-reverse actions (flashing, deleting, pushing upstream).
- Mark anything unconfirmed and say where it was checked.
- GitHub user: `universam1`. The fork is `universam1/rtl8196e-gateway`; upstream is `jnilo1/rtl8196e-gateway`.
