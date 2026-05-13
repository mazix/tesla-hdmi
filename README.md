# tesla-hdmi — DEPRECATED

> This repository was the **early prototype** of what is now
> **[mazix/tesla-pi-station](https://github.com/mazix/tesla-pi-station)**.
> Active development continues there.

## Why

`tesla-hdmi` started narrowly: bring a USB HDMI capture stick (Xiaomi
TV Stick, Apple TV, console, etc.) into the Tesla browser through a
Raspberry Pi.

Once that was working, the same Pi grew to need:

- a Wi-Fi hotspot for the Tesla to join,
- carrier hotspot-detection bypass via iPhone USB tether (TTL=65),
- KasmVNC remote desktop with a Let's Encrypt cert and a port-less
  `https://your.domain` URL,
- Bluetooth A2DP audio routed to the car,
- a Tk GUI to pair Bluetooth devices,
- a *second* display backend (`x11vnc` + noVNC on the real `:0` Xorg
  with V3D acceleration) and a switcher CLI/GUI to flip between them,
- an on-screen virtual keyboard,
- a CarPlay / Android Auto kiosk via a Carlinkit CCPA USB dongle,
- and a fair number of Tesla-specific quirks (Chromium Private Network
  Access blocking RFC 1918 IPs, Tesla refusing A2DP for phone-class BT
  devices, DNS-over-HTTPS bypassing local dnsmasq hijacks, etc.).

By the time the second feature landed it was clear this is one cohesive
*Tesla Pi station* project, not an HDMI bridge, so it was rewritten as
a modular, idempotent installer at
**https://github.com/mazix/tesla-pi-station**.

The HDMI capture path from this repo now lives there as **module 12**
(`12-hdmi-capture.sh`).

## What about the old README?

The original `tesla-hdmi` README — uStreamer build, Caddy HTTPS, manual
Wi-Fi AP, USB tether priority, the lot — is preserved in git history at
[`44dc5bf` — Create README.md](https://github.com/mazix/tesla-hdmi/blob/44dc5bf/README.md).
