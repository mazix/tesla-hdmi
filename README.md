# TESLA HDMI PROJECT

![Status](https://img.shields.io/badge/status-experimental-orange) ![License](https://img.shields.io/badge/license-MIT-green) ![RaspberryPi](https://img.shields.io/badge/Raspberry%20Pi-HDMI%20Capture-red) ![Tesla](https://img.shields.io/badge/Tesla-Browser-blue)

> **Turn a Raspberry Pi into an in‑car HDMI → Tesla Browser streaming bridge with optional Bluetooth audio forwarding – fully headless, auto (re)connect, and local HTTPS.**

---

## ⚠️ Disclaimer / Safety

* This project can \*\*technically continue operating while the vehicle is in \*\****Drive*** (Tesla browser + Bluetooth do not automatically stop the pipeline). **You must NOT watch video or interact with this system while driving.** It is provided strictly for *parked* use, passenger entertainment, or track / demo scenarios where it is legal and safe.
* Obey all local laws & regulations. Driver distraction laws may prohibit on‑road use.
* You are responsible for network & Bluetooth security. Use unique passwords, regenerate keys/certs, and rotate them regularly.
* No warranty; use at your own risk. Authors / contributors are **not liable** for any damage, violations, or incidents arising from use or misuse.
* If in doubt, **disconnect power or do not load the page while driving**.

---

## 🗂️ Table of Contents

* [⚠️ Disclaimer / Safety](#️-disclaimer--safety)
* [✨ Features](#-features)
* [🚀 Quick Start](#-quick-start)
* [⚡ Full Scriptable Quick Start (Advanced)](#-full-scriptable-quick-start-advanced)
* [🧱 Hardware List](#-hardware-list)
* [🗺️ Architecture Overview](#️-architecture-overview)
* [🔐 Naming & Placeholder Conventions](#-naming--placeholder-conventions)
* [Base System Setup](#1-base-system-setup)
* [uStreamer Build](#2-build--install-ustreamer)
* [WebSocket Proxy](#3-mjpeg--websocket-proxy-optional-for-some-browsers)
* [HTTPS via Caddy](#4-local-https-with-caddy-self-signed)
* [Wi‑Fi AP](#5-wi-fi-ap-optional--metrics)
* [USB Tether Priority](#6-usb-tether-priority)
* [Conditional Bluetooth Pair / Auto Connect](#7-conditional-bluetooth-pair--auto-connect)
* [Bluetooth Audio Loopback](#8-bluetooth-audio-loopback-capture--tesla-a2dp)
* [Performance Tuning](#9-performance-tuning)
* [Troubleshooting](#10-troubleshooting)
* [Future Enhancements](#11-future-enhancements)
* [Security Notes](#12-security-notes)
* [Clean Uninstall](#13-clean-uninstall-core-components)
* [Folder Structure](#14-folder-structure-suggestion)
* [Usage (In Vehicle)](#15-usage-in-vehicle)
* [License](#16-license)
* [Credits](#17-credits--acknowledgments)
* [Final Notes](#18-final-notes)

---

## ✨ Features

| Component                                      | Purpose                                                          |
| ---------------------------------------------- | ---------------------------------------------------------------- |
| HDMI USB Capture (UVC + optional UAC)          | Captures external HDMI source (Android box / Apple TV / Console) |
| `uStreamer` + MJPEG + WebSocket proxy          | Low‑latency MJPEG frame serving inside Tesla browser             |
| Reverse proxy (Caddy) + self‑signed TLS        | Single `https://` URL inside car; avoids mixed content warnings  |
| Conditional BT pairing script                  | First boot opens pairing window, later boots auto connect only   |
| Audio loopback script                          | USB capture (UAC) → Tesla via A2DP (Bluetooth)                   |
| USB Tether (iPhone or phone hotspot via cable) | Primary WAN / fallback to Ethernet or other interface            |
| Local Wi‑Fi AP (optional)                      | Provide LAN for maintenance / alternate access                   |
| Systemd timers & services                      | Headless reliability / auto recovery                             |
| Latency tuning knobs                           | FPS, JPEG quality, loopback latency, buffering                   |

---

## 🚀 Quick Start

**Goal:** Minimal working video + (optional) audio to Tesla browser with conditional pairing.

```text
Time ~30–45 min (fresh Pi OS Lite, networked)
```

1. **Flash & Prepare**

   * Flash latest Raspberry Pi OS Lite (Bookworm) → enable SSH, set hostname & user.
   * Boot Pi, SSH in: `sudo apt update && sudo apt -y full-upgrade` → reboot.
2. **Install Core Packages**

   ```bash
   sudo apt install -y git build-essential libjpeg-dev libevent-dev \
     bluez bluez-tools pulseaudio pulseaudio-module-bluetooth alsa-utils \
     caddy jq curl
   ```
3. **Build uStreamer**

   ```bash
   cd /opt && sudo git clone https://github.com/pikvm/ustreamer.git
   cd ustreamer && sudo make WITH_WEBSOCKETS=1 && sudo make install
   ```
4. **uStreamer Service** (720p60 example):

   ```bash
   sudo tee /etc/systemd/system/ustreamer.service <<'EOF'
   [Unit]
   Description=uStreamer HDMI capture
   After=network-online.target
   [Service]
   ExecStart=/usr/local/bin/ustreamer \
     --device=/dev/video0 --format=MJPEG \
     --resolution=1280x720 --desired-fps=60 --quality=60 \
     --host=127.0.0.1 --port=8000 --allow-truncated-frames --drop-same-frames=0
   Restart=always
   [Install]
   WantedBy=multi-user.target
   EOF
   sudo systemctl enable --now ustreamer
   ```
5. **(Optional) WebSocket Proxy** – clone `mjpeg-ws` (or skip and use direct MJPEG) and run on port 9000.
6. **HTTPS Reverse Proxy (Caddy)**

   * Generate self‑signed cert for Pi IP.
   * Caddyfile reverse\_proxy → `127.0.0.1:9000` (or `8000` if skipping WS).
7. **Conditional Bluetooth Scripts**

   * Set system alias: `sudo bluetoothctl system-alias TeslaHDMIAudio`.
   * Install `bt-conditional-onboot.sh` (auto pair first boot, auto connect later).
   * Install `tesla-audio-maint.sh` + timer (loopback USB audio → BT sink).
8. **USB Tether Priority (optional)** – ensure phone tether iface metric < ethernet.
9. **Test (Bench)**

   * Open `https://<Pi_IP>/` in another LAN browser → video.
   * Manually pair once with any BT speaker (simulates Tesla) → verify loopback.
10. **Deploy in Car**

    * Power Pi via 12V → 5V adapter.
    * First run: Tesla pairs within 120 s.
    * Later runs: Pi auto attempts connect silently.
    * Browse: `https://<Pi_IP>/` → stream plays (even if car is in Drive; DO NOT watch while driving).

> After this quick path you can deep‑dive below for advanced Wi‑Fi AP, tuning & troubleshooting.

---

## ⚡ Full Scriptable Quick Start (Advanced)

<details><summary><strong>Click to expand full copy/paste automation block</strong></summary>

> Use this if you prefer a mostly automated setup. Otherwise the simpler **Quick Start** above may be enough. **Review & edit placeholders BEFORE running.**

```bash
# 1. Update OS & install core packages
sudo apt update && sudo apt -y full-upgrade
sudo apt install -y git build-essential libjpeg-dev libevent-dev libbsd-dev \
  bluez bluez-tools pulseaudio pulseaudio-module-bluetooth alsa-utils \
  caddy curl jq nodejs npm

# 2. Build & start uStreamer (adjust resolution/FPS)
cd /opt && sudo git clone https://github.com/pikvm/ustreamer.git && cd ustreamer
sudo make WITH_WEBSOCKETS=1 && sudo make install
cat <<'EOF' | sudo tee /etc/systemd/system/ustreamer.service
[Unit]
Description=uStreamer
After=network-online.target
[Service]
ExecStart=/usr/local/bin/ustreamer --device=/dev/video0 --format=MJPEG \
  --resolution=<CAP_RES> --desired-fps=<CAP_FPS> --quality=60 \
  --host=127.0.0.1 --port=<STREAM_HTTP_PORT> --allow-truncated-frames --drop-same-frames=0
Restart=always
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable --now ustreamer

# 3. (Optional) WebSocket proxy
cd /opt && sudo git clone https://github.com/geekman/mjpeg-ws.git && cd mjpeg-ws
sudo npm install --production
cat <<'EOF' | sudo tee /etc/systemd/system/mjpeg-ws-proxy.service
[Unit]
Description=MJPEG WS Proxy
After=ustreamer.service
[Service]
WorkingDirectory=/opt/mjpeg-ws
ExecStart=/usr/bin/node server.js --mjpeg-url http://127.0.0.1:<STREAM_HTTP_PORT>/stream --listen 127.0.0.1:<WS_PORT>
Restart=always
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable --now mjpeg-ws-proxy

# 4. Local self‑signed HTTPS (single IP)
sudo mkdir -p /etc/caddy/certs
cat <<'EOF' | sudo tee /etc/caddy/certs/openssl.cnf
[ req ]
distinguished_name = dn
x509_extensions = v3_req
prompt = no
[ dn ]
CN = <SELF_DOMAIN_OR_IP>
[ v3_req ]
subjectAltName = @alt_names
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
[ alt_names ]
IP.1 = <SELF_DOMAIN_OR_IP>
EOF
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/caddy/certs/self.key -out /etc/caddy/certs/self.crt \
  -config /etc/caddy/certs/openssl.cnf
cat <<'EOF' | sudo tee /etc/caddy/Caddyfile
:80 { redir https://{host}{uri} }
:443 {
  tls /etc/caddy/certs/self.crt /etc/caddy/certs/self.key
  encode zstd gzip
  reverse_proxy 127.0.0.1:<WS_PORT>
}
EOF
sudo systemctl restart caddy

# 5. Conditional BT pairing (first boot shows window; later boots auto-connect)
cat <<'EOF' | sudo tee /usr/local/bin/bt-conditional-onboot.sh
#!/usr/bin/env bash
set -euo pipefail
TARGET_NAME="<BT_DEVICE_NAME>"
log(){ echo "[BT-CONDITIONAL] $(date '+%F %T') $*"; }
for i in {1..5}; do bluetoothctl show >/dev/null 2>&1 && break; sleep 1; done
PAIRED_LINE=$(bluetoothctl devices Paired 2>/dev/null || true | grep -F "$TARGET_NAME" || true)
if [ -n "$PAIRED_LINE" ]; then
  MAC=$(echo "$PAIRED_LINE" | awk '{print $2}')
  log "Previously paired ($MAC). Connecting."; bluetoothctl connect "$MAC" >/dev/null 2>&1 || true; exit 0; fi
log "Entering 120s pairing window."; bluetoothctl power on >/dev/null 2>&1 || true
bluetoothctl agent NoInputNoOutput >/dev/null 2>&1 || true; bluetoothctl default-agent >/dev/null 2>&1 || true
bluetoothctl discoverable on >/dev/null 2>&1 || true; bluetoothctl pairable on >/dev/null 2>&1 || true; bluetoothctl scan on >/dev/null 2>&1 || true
sleep 120
bluetoothctl scan off >/dev/null 2>&1 || true; bluetoothctl discoverable off >/dev/null 2>&1 || true; bluetoothctl pairable off >/dev/null 2>&1 || true
log "Pair window closed."; exit 0
EOF
sudo chmod 755 /usr/local/bin/bt-conditional-onboot.sh
cat <<'EOF' | sudo tee /etc/systemd/system/bt-conditional-onboot.service
[Unit]
Description=Conditional BT Pair
After=bluetooth.service
[Service]
Type=oneshot
ExecStart=/usr/local/bin/bt-conditional-onboot.sh
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable --now bt-conditional-onboot.service
sudo bluetoothctl system-alias <BT_DEVICE_NAME>

# 6. Audio loopback maintenance (every 30s)
cat <<'EOF' | sudo tee /usr/local/bin/tesla-audio-maint.sh
#!/usr/bin/env bash
set -eo pipefail
LOG(){ echo "[AUDIO] $(date '+%F %T') $*"; }
USER_NAME="<PI_USER>"; TARGET_NAME="<BT_DEVICE_NAME>"; LATENCY_MS=80; SINK_MATCH=bluez_sink; SRC_MATCH=alsa_input.usb
LOG "Script start."
MAC=$(bluetoothctl devices Paired 2>/dev/null || true | awk -v n="$TARGET_NAME" '$0 ~ n {print $2; exit}')
[ -z "$MAC" ] && { LOG "No paired device"; exit 0; }
bluetoothctl info "$MAC" 2>/dev/null | grep -q "Connected: yes" || bluetoothctl connect "$MAC" >/dev/null 2>&1 || true
sudo -u "$USER_NAME" pactl info >/dev/null 2>&1 || sudo -u "$USER_NAME" pulseaudio --start 2>/dev/null || true
SINK=""; for _ in {1..10}; do SINK=$(sudo -u "$USER_NAME" pactl list short sinks | awk -v m="$SINK_MATCH" '$0 ~ m {print $1; exit}'); [ -n "$SINK" ] && break; sleep 1; done; [ -z "$SINK" ] && { LOG "No sink"; exit 0; }
LOG "Sink: $SINK"
SRC=$(sudo -u "$USER_NAME" pactl list short sources | awk -v m="$SRC_MATCH" '$0 ~ m {print $1; exit}')
[ -z "$SRC" ] && { LOG "No source"; exit 0; }
LOG "Source: $SRC"
if ! sudo -u "$USER_NAME" pactl list modules | grep -q "module-loopback.*$SRC.*$SINK"; then MID=$(sudo -u "$USER_NAME" pactl load-module module-loopback source="$SRC" sink="$SINK" latency_msec=$LATENCY_MS 2>/dev/null || echo ""); LOG "Loopback id=$MID"; fi
EOF
sudo chmod 755 /usr/local/bin/tesla-audio-maint.sh
cat <<'EOF' | sudo tee /etc/systemd/system/tesla-audio-maint.service
[Unit]
Description=BT audio loopback
After=bluetooth.service
[Service]
Type=oneshot
ExecStart=/usr/local/bin/tesla-audio-maint.sh
EOF
cat <<'EOF' | sudo tee /etc/systemd/system/tesla-audio-maint.timer
[Unit]
Description=BT audio loopback timer
[Timer]
OnBootSec=20
OnUnitActiveSec=30
[Install]
WantedBy=timers.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now tesla-audio-maint.timer

# 7. Browse from Tesla: https://<SELF_DOMAIN_OR_IP>/
#    Pair Bluetooth (if first time). Play HDMI source.
```

</details>

## 🧱 Hardware List

| Item                                                   | Notes                                                            |
| ------------------------------------------------------ | ---------------------------------------------------------------- |
| Raspberry Pi (3B+ or 4 recommended; 1GB works)         | More CPU = more headroom for 60 FPS                              |
| Quality USB‑C / micro‑USB power (stable)               | Avoid brown‑outs                                                 |
| HDMI → USB UVC capture dongle (MS2109, TC358743, etc.) | Prefer models with **UAC (audio)** if you want Bluetooth forward |
| Reliable microSD (≥16GB, A1 rated)                     | Endurance matters                                                |
| Optional: USB Audio ADC (if capture dongle lacks UAC)  | For analog source audio                                          |
| Smartphone (USB tether)                                | Internet fallback or primary                                     |
| Tesla vehicle with browser + Bluetooth                 | Target display & speakers                                        |

---

## 🗺️ Architecture Overview

```
HDMI Source ──(HDMI)──> USB Capture Dongle ──(USB)──> Raspberry Pi
   │                                                    │
   │ Video: uStreamer (MJPEG @ 30–60 FPS)               │
   │                                                    │
   └── Audio (if UAC) -> ALSA -> PulseAudio -> A2DP -> Tesla BT

Tesla Browser <--HTTPS (Caddy reverse proxy)<-- uStreamer + WS proxy (localhost)

USB Tether (usb0 or eth1) -> Internet
Ethernet / Wi‑Fi AP -> Local access / fallback
Systemd services -> Auto start, conditional pairing, maintenance loop
```

---

## 🔐 Naming & Placeholder Conventions

Replace *every* placeholder with your own values:

| Placeholder           | Meaning                                                     |
| --------------------- | ----------------------------------------------------------- |
| `<PI_USER>`           | Your Raspberry Pi username (non‑root)                       |
| `<AP_IP>`             | Static IP for Pi's Wi‑Fi AP interface (e.g. `10.50.0.1/24`) |
| `<AP_SSID>`           | Wi‑Fi SSID you advertise (e.g. `TeslaStream`)               |
| `<AP_PSK>`            | Strong WPA2 passphrase                                      |
| `<ETH_STATIC_IP>`     | (Optional) Static LAN IP if not using DHCP reservation      |
| `<BT_DEVICE_NAME>`    | Visible Bluetooth name (e.g. `TeslaHDMIAudio`)              |
| `<SELF_DOMAIN_OR_IP>` | IP (local) or mDNS name you will browse (e.g. Pi's LAN IP)  |
| `<STREAM_HTTP_PORT>`  | Internal uStreamer HTTP port (e.g. `8000`)                  |
| `<WS_PORT>`           | WebSocket proxy port (e.g. `9000`)                          |
| `<HTTPS_PORT>`        | Usually `443`                                               |
| `<CAP_RES>`           | Capture resolution (e.g. `1280x720`)                        |
| `<CAP_FPS>`           | Desired FPS (e.g. `60`)                                     |

---

## 1. Base System Setup

```bash
# Flash latest Raspberry Pi OS (Lite / Bookworm) using Raspberry Pi Imager.
# Enable SSH (imager advanced options) and set a unique password.

sudo apt update && sudo apt -y full-upgrade
sudo reboot

# Essential packages
sudo apt install -y git build-essential pkg-config libjpeg-dev libevent-dev \
  libbsd-dev curl unzip jq ca-certificates ufw \
  bluez bluez-tools pulseaudio pulseaudio-module-bluetooth alsa-utils \
  hostapd dnsmasq iptables-persistent caddy
```

> **Note:** If PipeWire ships by default, PulseAudio compatibility still works (pactl commands remain the same).

---

## 2. Build & Install uStreamer

```bash
cd /opt
sudo git clone https://github.com/pikvm/ustreamer.git
cd ustreamer
sudo make WITH_WEBSOCKETS=1        # Build binaries
sudo make install                   # Installs to /usr/local/bin
```

Check:

```bash
/usr/local/bin/ustreamer --help | head
```

### 2.1 Systemd Service for uStreamer

Create service (edit placeholders):

```bash
sudo tee /etc/systemd/system/ustreamer.service <<'EOF'
[Unit]
Description=uStreamer HDMI capture
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/ustreamer \
  --device=/dev/video0 \
  --format=MJPEG \
  --resolution=<CAP_RES> \
  --desired-fps=<CAP_FPS> \
  --quality=60 \
  --host=127.0.0.1 --port=<STREAM_HTTP_PORT> \
  --allow-truncated-frames \
  --drop-same-frames=0
Restart=always
RestartSec=2

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now ustreamer
journalctl -u ustreamer -n 20 -o cat
```

> **Tip:** If the capture dongle presents MJPEG already (`--format=MJPEG`) you offload CPU; otherwise consider YUYV + CPU encoder at lower FPS if performance issues arise.

---

## 3. MJPEG → WebSocket Proxy (Optional for Some Browsers)

Some browser environments handle a lightweight WebSocket wrapper more smoothly.

```bash
cd /opt
sudo git clone https://github.com/geekman/mjpeg-ws.git
cd mjpeg-ws
sudo npm install --production   # or yarn install --production
```

Service:

```bash
sudo tee /etc/systemd/system/mjpeg-ws-proxy.service <<'EOF'
[Unit]
Description=MJPEG to WebSocket Proxy
After=ustreamer.service

[Service]
WorkingDirectory=/opt/mjpeg-ws
ExecStart=/usr/bin/node server.js \
  --mjpeg-url http://127.0.0.1:<STREAM_HTTP_PORT>/stream \
  --listen 127.0.0.1:<WS_PORT>
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now mjpeg-ws-proxy
journalctl -u mjpeg-ws-proxy -n 20 -o cat
```

> Adjust repository if you choose a different lightweight proxy; or embed a simple `<img src="/stream">` if direct MJPEG suffices.

---

## 4. Local HTTPS with Caddy (Self‑Signed)

Self‑signed (local LAN IP) certificate to avoid mixed content / allow media autoplay toggles.

```bash
sudo mkdir -p /etc/caddy/certs
sudo tee /etc/caddy/certs/openssl.cnf <<'EOF'
[ req ]
distinguished_name = dn
x509_extensions = v3_req
prompt = no

[ dn ]
CN = <SELF_DOMAIN_OR_IP>

[ v3_req ]
subjectAltName = @alt_names
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[ alt_names ]
IP.1 = <SELF_DOMAIN_OR_IP>
EOF

sudo openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout /etc/caddy/certs/self.key \
  -out /etc/caddy/certs/self.crt \
  -config /etc/caddy/certs/openssl.cnf
```

Caddyfile:

```bash
sudo tee /etc/caddy/Caddyfile <<'EOF'
:80 {
  redir https://{host}{uri}
}
:443 {
  tls /etc/caddy/certs/self.crt /etc/caddy/certs/self.key
  encode zstd gzip
  reverse_proxy 127.0.0.1:<WS_PORT>
}
EOF

sudo caddy fmt --overwrite /etc/caddy/Caddyfile
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl restart caddy
journalctl -u caddy -n 20 -o cat
```

> Import the self‑signed certificate into the Tesla browser if possible (workflow varies) or accept the warning once. For multi‑device trust you can generate a local CA and sign instead.

---

## 5. Wi‑Fi AP (Optional) + Metrics

If you want Pi to provide an AP *only* for maintenance / fallback:

```bash
# Example using NetworkManager (Raspberry Pi OS Bookworm default) OR hostapd/dnsmasq.
# (If using NetworkManager) create a shared hotspot:
sudo nmcli connection add type wifi ifname wlan0 mode ap con-name tesla-ap ssid <AP_SSID>

sudo nmcli connection modify tesla-ap 802-11-wireless.band bg 802-11-wireless.channel 6 \
  wifi-sec.key-mgmt wpa-psk wifi-sec.psk '<AP_PSK>' ipv4.addresses <AP_IP> ipv4.method shared ipv4.route-metric 300

sudo nmcli connection up tesla-ap
```

If using classic hostapd/dnsmasq approach, adapt with static IP in `/etc/dhcpcd.conf` and hostapd config. Keep AP isolated if concerned about drive‑by clients.

---

## 6. USB Tether Priority

When a phone is tethered via USB it enumerates (e.g. `usb0` or `eth1`). Set metrics to prefer it over Ethernet/Wi‑Fi.

```bash
# Example using NetworkManager route metrics
sudo nmcli connection modify 'Wired connection 1' ipv4.route-metric 300
sudo nmcli connection modify 'Wired connection 2' ipv4.route-metric 100   # Suppose this is the iPhone tether
```

Alternatively with `dhcpcd.conf`:

```
interface usb0
  metric 100
interface eth0
  metric 300
```

Restart related service (`systemctl restart NetworkManager` or `dhcpcd`).

---

## 7. Conditional Bluetooth Pair / Auto Connect

Script only presents pairing window if device has **no prior pairing**; otherwise it attempts direct connect silently.

```bash
sudo tee /usr/local/bin/bt-conditional-onboot.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
TARGET_NAME="<BT_DEVICE_NAME>"
log(){ echo "[BT-CONDITIONAL] $(date '+%F %T') $*"; }
for i in {1..5}; do bluetoothctl show >/dev/null 2>&1 && break; sleep 1; done
PAIRED_LINE=$(bluetoothctl devices Paired 2>/dev/null || true | grep -F "$TARGET_NAME" || true)
if [ -n "$PAIRED_LINE" ]; then
  MAC=$(echo "$PAIRED_LINE" | awk '{print $2}')
  log "Previously paired ($MAC). Attempting direct connect."
  bluetoothctl power on >/dev/null 2>&1 || true
  bluetoothctl pairable off >/dev/null 2>&1 || true
  bluetoothctl discoverable off >/dev/null 2>&1 || true
  bluetoothctl connect "$MAC" >/dev/null 2>&1 || true
  exit 0
fi
log "No existing pair. Entering 120s pairing window."
bluetoothctl power on >/dev/null 2>&1 || true
bluetoothctl agent NoInputNoOutput >/dev/null 2>&1 || true
bluetoothctl default-agent >/dev/null 2>&1 || true
bluetoothctl discoverable on >/dev/null 2>&1 || true
bluetoothctl pairable on >/dev/null 2>&1 || true
bluetoothctl scan on >/dev/null 2>&1 || true
for sec in $(seq 1 120); do
  if [ $((sec % 2)) -eq 0 ]; then
    PAIRED_LINE=$(bluetoothctl devices Paired 2>/dev/null || true | grep -F "$TARGET_NAME" || true)
    [ -n "$PAIRED_LINE" ] && { log "Pair complete early."; break; }
  fi
  printf "\r[Pair Window] %3ds left. Pair '%s'... " $((120-sec)) "$TARGET_NAME"
  sleep 1
done
echo
bluetoothctl scan off >/dev/null 2>&1 || true
bluetoothctl discoverable off >/dev/null 2>&1 || true
bluetoothctl pairable off >/dev/null 2>&1 || true
log "Pair window closed."
EOF

sudo chmod 755 /usr/local/bin/bt-conditional-onboot.sh
```

Systemd service:

```bash
sudo tee /etc/systemd/system/bt-conditional-onboot.service <<'EOF'
[Unit]
Description=Conditional BT pair or auto-connect
After=bluetooth.service
Wants=bluetooth.service
[Service]
Type=oneshot
ExecStart=/usr/local/bin/bt-conditional-onboot.sh
[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable bt-conditional-onboot.service
```

Set device alias:

```bash
sudo bluetoothctl system-alias <BT_DEVICE_NAME>
```

---

## 8. Bluetooth Audio Loopback (Capture → Tesla A2DP)

Main maintenance script (idempotent):

```bash
sudo tee /usr/local/bin/tesla-audio-maint.sh <<'EOF'
#!/usr/bin/env bash
set -eo pipefail
LOG(){ echo "[AUDIO] $(date '+%F %T') $*"; }
USER_NAME="<PI_USER>"
TARGET_NAME="<BT_DEVICE_NAME>"
LATENCY_MS=80
SINK_MATCH=bluez_sink
SRC_MATCH=alsa_input.usb
LOG "Script start."
# MAC lookup (ignore errors)
MAC=$(bluetoothctl devices Paired 2>/dev/null || true | awk -v n="$TARGET_NAME" '$0 ~ n {print $2; exit}')
if [ -z "$MAC" ]; then LOG "No paired device named $TARGET_NAME."; exit 0; fi
if ! bluetoothctl info "$MAC" 2>/dev/null | grep -q "Connected: yes"; then
  LOG "Attempting connect $MAC"; bluetoothctl connect "$MAC" >/dev/null 2>&1 || true; sleep 2;
fi
# Start PulseAudio (user)
if ! sudo -u "$USER_NAME" pactl info >/dev/null 2>&1; then
  LOG "Starting PulseAudio for $USER_NAME"; sudo -u "$USER_NAME" pulseaudio --start 2>/dev/null || true; sleep 1;
fi
# Find sink
SINK=""
for _ in {1..10}; do SINK=$(sudo -u "$USER_NAME" pactl list short sinks 2>/dev/null | awk -v m="$SINK_MATCH" '$0 ~ m {print $1; exit}'); [ -n "$SINK" ] && break; sleep 1; done
[ -z "$SINK" ] && { LOG "No BT sink yet."; exit 0; }
LOG "Sink: $SINK"
# Find source (USB audio)
SRC=$(sudo -u "$USER_NAME" pactl list short sources | awk -v m="$SRC_MATCH" '$0 ~ m {print $1; exit}')
[ -z "$SRC" ] && { LOG "No USB audio source (video-only dongle?)."; exit 0; }
LOG "Source: $SRC"
# Loopback present?
if sudo -u "$USER_NAME" pactl list modules | grep -q "module-loopback.*$SRC.*$SINK"; then
  LOG "Loopback already active."; exit 0; fi
MID=$(sudo -u "$USER_NAME" pactl load-module module-loopback source="$SRC" sink="$SINK" latency_msec=$LATENCY_MS 2>/dev/null || echo "")
[ -n "$MID" ] && LOG "Loopback loaded id=$MID ($SRC->$SINK)" || LOG "Loopback load failed"
EOF

sudo chmod 755 /usr/local/bin/tesla-audio-maint.sh
```

Timer:

```bash
sudo tee /etc/systemd/system/tesla-audio-maint.service <<'EOF'
[Unit]
Description=Maintain Tesla Bluetooth audio loopback
After=bluetooth.service
[Service]
Type=oneshot
ExecStart=/usr/local/bin/tesla-audio-maint.sh
EOF

sudo tee /etc/systemd/system/tesla-audio-maint.timer <<'EOF'
[Unit]
Description=Periodic Tesla BT audio maintenance
[Timer]
OnBootSec=20
OnUnitActiveSec=30
[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now tesla-audio-maint.timer
```

Test tone (after sink appears):

```bash
sudo -u <PI_USER> pactl load-module module-sine frequency=440   # Unload with pactl unload-module <ID>
```

---

## 9. Performance Tuning

| Aspect           | Knob                | Notes                                                        |
| ---------------- | ------------------- | ------------------------------------------------------------ |
| FPS              | `--desired-fps`     | If unstable, try 45 or 30.                                   |
| JPEG Quality     | `--quality` (10–95) | Lower = less bandwidth & CPU.                                |
| Resolution       | `--resolution`      | 1280x720 is good compromise for Pi 3; Pi 4 can do 1080p\@30. |
| Loopback Latency | `LATENCY_MS`        | Lower (60) = less delay, may cause stutter.                  |
| BT Interference  | Channel selection   | Move Wi‑Fi AP off crowded channels.                          |
| Power            | Good PSU            | Avoid throttling; check `vcgencmd get_throttled`.            |

### Measuring FPS Quickly

```bash
curl -s http://127.0.0.1:<STREAM_HTTP_PORT>/state | jq '.result.source.captured_fps'
```

---

## 10. Troubleshooting

| Symptom                           | Check                                   | Fix                                                              |
| --------------------------------- | --------------------------------------- | ---------------------------------------------------------------- |
| Tesla page blank                  | Open HTTPS directly from another device | Cert / proxy misconfig → check Caddy logs                        |
| 0 FPS but service running         | `journalctl -u ustreamer`               | Capture dongle handshake; try replug / lower res/FPS             |
| Audio silent (no sink)            | `pactl list short sinks`                | Ensure Tesla connected (Bluetooth menu)                          |
| Audio silent (sink ok, no source) | `arecord -l`                            | Dongle lacks UAC → add USB audio ADC                             |
| Loopback repeats every timer      | Module already?                         | Script checks; if not, PulseAudio being restarted → inspect logs |
| High latency audio                | Lower `LATENCY_MS`                      | Balance vs stutter                                               |
| BT fails first try                | RF noise                                | Retry; ensure only one Pi advertising name                       |
| HTTPS browser warning             | Self‑signed cert                        | Accept once or deploy local CA                                   |

---

## 11. Future Enhancements

* H.264 hardware encoding (Pi Camera or different pipeline) + browser player.
* Adaptive bitrate (switch resolution on tether bandwidth).
* Web UI overlay (hide FPS, tap to show stats, dark mode).
* MQTT / REST control channel (start/stop stream remotely).
* PipeWire & AAC codec (if Tesla supports better codec negotiation).
* Auto suspend when vehicle driving (if detecting movement via CAN/OBD integration – **respect safety**).

---

## 12. Security Notes

* Restrict AP usage; rotate passwords.
* Optionally firewall outside ports: `ufw allow 443/tcp` / limit others.
* Disable discoverable mode after pairing (already handled by conditional script).

---

## 13. Clean Uninstall (Core Components)

```bash
sudo systemctl disable --now tesla-audio-maint.timer tesla-audio-maint.service \
  bt-conditional-onboot.service mjpeg-ws-proxy ustreamer caddy
sudo rm -f /etc/systemd/system/{tesla-audio-maint.service,tesla-audio-maint.timer,bt-conditional-onboot.service,mjpeg-ws-proxy.service,ustreamer.service}
# (Optionally) remove /opt/mjpeg-ws /opt/ustreamer
```

---

## 14. Folder Structure Suggestion

```
/opt/ustreamer/          # Source build
/opt/mjpeg-ws/           # WS proxy
/etc/caddy/              # TLS + Caddyfile
/usr/local/bin/          # Scripts (bt-conditional, tesla-audio-maint)
/var/log/journal/        # Systemd logs (query via journalctl)
```

---

## 15. Usage (In Vehicle)

1. Power Pi → If first run: pairing window (120s) → pair Tesla.
2. Browse to `https://<SELF_DOMAIN_OR_IP>/` in Tesla.
3. Start HDMI source playback.
4. Ensure Bluetooth connection (audio should play through vehicle speakers).
5. Enjoy (while **not driving**).

---

## 16. License

Recommend a permissive license (MIT) unless you require copyleft. Example:

```text
MIT License
Copyright (c) <YEAR> <YOUR_NAME>
Permission is hereby granted, free of charge, to any person obtaining a copy ... (standard MIT text)
```

---

## 17. Credits / Acknowledgments

* [uStreamer](https://github.com/pikvm/ustreamer) – lightweight MJPEG streaming.
* Community examples of Pi + HDMI capture pipelines.
* Contributors of BlueZ, Caddy, PulseAudio/PipeWire.

---

## 18. Final Notes

> Keep logs handy: `journalctl -u ustreamer -u mjpeg-ws-proxy -u caddy -u tesla-audio-maint.service -u bt-conditional-onboot.service -n 50 -o cat`.
>
> Tweak gradually: change one parameter at a time (resolution, FPS, quality) and observe CPU (`top`, `vcgencmd measure_temp`).

Happy building! Feel free to open issues / PRs to extend **TESLA HDMI PROJECT**.
