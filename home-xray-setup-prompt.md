# Home Xray (VLESS + REALITY + Vision) Setup — Context Prompt & Maintenance Guide

> **How to use this file:** Paste everything under "PROMPT" into an AI assistant (e.g. Claude) when you need help
> maintaining, troubleshooting, or changing this setup. The "Routine Maintenance" section is your own checklist.
>
> **Placeholders:** `<PI_LAN_IP>`, `<PI_USER>` and `<YOUR_SUBDOMAIN>` stand for your own values.
>
> **No secrets are stored here.** Keys, user IDs (UUIDs), and the DuckDNS token live only on the Raspberry Pi
> and in the Shadowrocket apps. Never paste them into a chat or share them.

---

## PROMPT

You are helping me maintain a self-hosted proxy at my home in the USA. I'm not a networking expert, so please
explain step by step, give one command at a time, and tell me what output to expect. Never ask me to paste private
keys, UUIDs, or my DuckDNS token into the chat.

### My goal
Use the internet through my home connection from anywhere — public Wi-Fi that blocks VPNs, and especially when I
travel to **mainland China** — while Chinese apps (WeChat, Alipay, Taobao, JD, Baidu, Ctrip, Qunar, Fliggy, Doubao,
Amap, Baidu Maps, China Everbright Bank, China Guangfa Bank) still connect directly from the local network.

### What was set up (2026-11-24)

**Hardware / network**
- Home router: **Verizon (Fios) router**, admin page at `https://192.168.1.1` (login password is on the router sticker).
- Server: **Raspberry Pi**, 64-bit Raspberry Pi OS (Debian trixie, aarch64), LAN IP **`<PI_LAN_IP>`** (e.g. 192.168.1.x),
  SSH user `<PI_USER>` (`ssh <PI_USER>@<PI_LAN_IP>`).
- Dynamic DNS: **DuckDNS**, hostname **`<YOUR_SUBDOMAIN>.duckdns.org`**. The Pi updates it every 5 minutes via cron:
  `*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1` (script at `~/duckdns/duck.sh`, last result in `~/duckdns/duck.log`).

**Xray server on the Pi**
- Installed with the official XTLS installer; version at setup: **Xray 26.11.9**. Runs as systemd service `xray`
  (enabled at boot).
- Config: `/usr/local/etc/xray/config.json` (owner root:xray, mode 640).
- Protocol: **VLESS + REALITY + Vision** (`flow: xtls-rprx-vision`), listening on **TCP 443**.
- REALITY target / SNI: **`www.samsung.com:443`** (chosen because www.microsoft.com triggers a "GFW" warning,
  and nvidia/logitech failed the no-SNI TLS test). Fingerprint in clients: `chrome`.
- One user (UUID) per device, labelled by `email`: **iphone**, **ipad**, **macbook**.
- Client links are stored on the Pi only (mode 600):
  `~/xray-client.txt` (iPhone), `~/xray-client-ipad.txt`, `~/xray-client-macbook.txt`.
- `qrencode` installed to show links as QR codes: `qrencode -t ansiutf8 < ~/xray-client-ipad.txt`.

**Router port forwards (Advanced → Security & Firewall → Port Forwarding)**
| Name | Port | Protocol | To | Notes |
|---|---|---|---|---|
| Xray | 443 | TCP | <PI_LAN_IP>:443 | Added for this setup |
| PiVPN | 1194 | UDP | <PI_LAN_IP>:1194 | Older OpenVPN server (PiVPN), still running, independent of Xray |
| (none) | 4577, 4567 | TCP | 127.0.0.1 | Verizon system rules — do not touch |

**Clients**
- **Shadowrocket** on iPhone, iPad, and Apple Silicon Mac (Mac version installed from the App Store with the
  same Apple ID). Server entries: `Home-Reality`, `Home-Reality-ipad`, `Home-Reality-macbook`.
- Routing config **`china-direct.conf`** imported into Shadowrocket (Config → Use Config; Global Routing = Config):
  - The Chinese apps listed above, all `.cn` domains, and China IPs (GEOIP,CN) → **DIRECT**
  - LAN / private ranges → DIRECT (so hotel Wi-Fi login pages work)
  - Everything else → **PROXY** (the selected Home-Reality server)
  - IPv6 disabled; DNS = system, 223.5.5.5, 119.29.29.29
  - A copy of the file is on the Mac's Desktop.
- Tested: on iPhone mobile data, "what is my IP" showed the home IP; Shadowrocket's log showed Chinese sites as
  DIRECT and others as PROXY.

**Other notes**
- PiVPN (OpenVPN, UDP 1194) is still active: useful as a US backup and for reaching home LAN devices,
  but it **may or may not** work in China.

### Still to do (as of setup day)
1. **Reserve <PI_LAN_IP> for the Pi** in the Verizon router's DHCP settings (Advanced → Network Settings →
   IP Address Distribution / static lease), so the port forwards never break.
1. Test the Mac via iPhone Personal Hotspot, and test once on a public Wi-Fi network.
1. Consider a small UPS for the Pi + router before the China trip.

---

## Routine Maintenance (your checklist)

### Monthly (at home)
- [ ] **Update Xray** on the Pi (keeps config, keys and links):
  ```
  sudo bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
  systemctl is-active xray        # should print: active
  ```
- [ ] **Update Raspberry Pi OS**:
  ```
  sudo apt update && sudo apt upgrade -y
  ```
- [ ] **Update Shadowrocket** on iPhone, iPad and Mac (App Store).
- [ ] **Check DuckDNS is updating**: `cat ~/duckdns/duck.log` should show `OK`.
- [ ] **Quick test**: iPhone on mobile data → connect Shadowrocket → "what is my IP" shows your home IP.

### Before travelling to China (1–2 weeks before)
- [ ] Do all the monthly updates above, then **test** on mobile data **and** a public Wi-Fi.
- [ ] Confirm the Pi's IP reservation (<PI_LAN_IP>) is in place on the router.
- [ ] Confirm Shadowrocket + `china-direct.conf` are installed and active on every device you'll bring.
- [ ] Reboot the Pi once and confirm Xray comes back (`systemctl is-active xray`).
- [ ] **After testing, don't update the Pi again until you're home.** A failed update can't be fixed from China.

### While in China
- Updating **Shadowrocket** is OK if connections get flaky.
- **Do not** update or change the Pi.
- If a Chinese app shows a US location or is slow, check Shadowrocket's log for the domain that went via PROXY
  and add it to `china-direct.conf` as `DOMAIN-SUFFIX,<domain>,DIRECT`.
- Banking apps may refuse to run while any VPN is on — briefly turn Shadowrocket off to use them.

### If a device is lost or stolen
Remove only that device's user from the server (example: ipad):
```
sudo python3 - ipad <<'EOF'
import json, sys
p = "/usr/local/etc/xray/config.json"
c = json.load(open(p))
cl = c["inbounds"][0]["settings"]["clients"]
c["inbounds"][0]["settings"]["clients"] = [x for x in cl if x.get("email") != sys.argv[1]]
json.dump(c, open(p, "w"), indent=2)
EOF
sudo xray run -test -config /usr/local/etc/xray/config.json && sudo systemctl restart xray && systemctl is-active xray
rm ~/xray-client-ipad.txt
```

### Quick troubleshooting
| Symptom | Check |
|---|---|
| Nothing connects anywhere | `systemctl status xray`; `sudo journalctl -u xray -n 50 --no-pager` |
| Works on Wi-Fi at home but not outside | Router port forward TCP 443 → <PI_LAN_IP>; Pi's IP unchanged |
| Stopped working after home IP changed | `cat ~/duckdns/duck.log` (should be OK); `crontab -l` still has the duck.sh line |
| Connects but some sites fail | Update Xray + Shadowrocket; check the REALITY target still passes `xray tls ping www.samsung.com` |
| Blocked in China after working | China may have flagged the IP/target; options: change REALITY target (test with `xray tls ping`), restart modem for a new IP (needs someone at home) |
