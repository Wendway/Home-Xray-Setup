# Home Xray Server on a Raspberry Pi (VLESS + REALITY + Vision)

This repo documents how I turned a Raspberry Pi at home into a private proxy server using
[Xray-core](https://github.com/XTLS/Xray-core) with **VLESS + REALITY + XTLS-Vision**. It lets my iPhone, iPad and
Mac use my home internet connection from anywhere, including networks that block normal VPNs, while Chinese apps
and sites still connect directly from the local network (split routing in Shadowrocket).

> **All personal values are replaced with placeholders.** Replace them with your own:
>
> | Placeholder | Meaning |
> |---|---|
> | `<PI_USER>` | Your Linux username on the Pi |
> | `<PI_LAN_IP>` | The Pi's LAN address, e.g. `192.168.1.x` |
> | `<YOUR_SUBDOMAIN>` | Your DuckDNS subdomain (`<YOUR_SUBDOMAIN>.duckdns.org`) |
> | `<DUCKDNS_TOKEN>` | Your DuckDNS token |
> | `<UUID_IPHONE>` etc. | One Xray user ID per device |
> | `<PRIVATE_KEY>` / `<PUBLIC_KEY>` | The REALITY x25519 key pair |
> | `<SHORT_ID>` | REALITY short ID (hex) |
>
> Never publish your real UUIDs, private key, short ID or DuckDNS token. Anyone with a client link can use your server.
>
> The command outputs below are **examples** of what you should see. Your keys, IDs, version numbers and
> timestamps will differ.

The file [`home-xray-setup-prompt.md`](home-xray-setup-prompt.md) is a companion "context prompt" you can paste into
an AI assistant for help with maintenance, plus a routine maintenance checklist.

---

## Contents

1. [How it works](#1-how-it-works)
2. [What you need](#2-what-you-need)
3. [Prepare the Raspberry Pi](#3-prepare-the-raspberry-pi)
4. [Dynamic DNS with DuckDNS](#4-dynamic-dns-with-duckdns)
5. [Install Xray](#5-install-xray)
6. [Generate IDs and keys](#6-generate-ids-and-keys)
7. [Choose a REALITY target](#7-choose-a-reality-target)
8. [Write the server config](#8-write-the-server-config)
9. [Test and start the service](#9-test-and-start-the-service)
10. [Router: port forward and DHCP reservation](#10-router-port-forward-and-dhcp-reservation)
11. [Create client links and QR codes](#11-create-client-links-and-qr-codes)
12. [Set up Shadowrocket (iPhone / iPad / Mac)](#12-set-up-shadowrocket-iphone--ipad--mac)
13. [China split routing (`china-direct.conf`)](#13-china-split-routing-china-directconf)
14. [Verify everything](#14-verify-everything)
15. [Maintenance and troubleshooting](#15-maintenance-and-troubleshooting)

---

## 1. How it works

```
 iPhone / iPad / Mac (Shadowrocket)
        │  VLESS + REALITY + Vision over TCP 443
        ▼
 <YOUR_SUBDOMAIN>.duckdns.org  ──►  home public IP  (kept up to date by DuckDNS)
        │
 Home router: port forward TCP 443 ──► <PI_LAN_IP>:443
        │
 Raspberry Pi running Xray  ──►  Internet
```

- **VLESS** is a lightweight proxy protocol. Each device gets its own UUID.
- **REALITY** makes the connection look like a normal TLS visit to a real, popular website (the "target", here
  `www.samsung.com`). Anyone probing the server without the right key is passed through to the real site, so the server
  has no certificate of its own to fingerprint and needs no domain certificate.
- **Vision** (`xtls-rprx-vision`) removes the "TLS inside TLS" pattern that censors look for.
- **Shadowrocket** routes Chinese apps/domains/IPs **DIRECT** and everything else through the home server.

## 2. What you need

- A Raspberry Pi (64-bit capable) with **Raspberry Pi OS 64-bit** (this setup: Debian trixie, aarch64), wired to the
  home router.
- A home router where you can set **port forwarding** and a **DHCP reservation** (this setup: Verizon Fios router,
  admin page `https://192.168.1.1`).
- A public IPv4 address from your ISP (not CGNAT). Check that the WAN IP on the router matches what
  `curl https://ifconfig.me` shows.
- A free [DuckDNS](https://www.duckdns.org) account and subdomain.
- [Shadowrocket](https://apps.apple.com/app/shadowrocket/id932747118) on each Apple device.

## 3. Prepare the Raspberry Pi

SSH in from your computer:

```bash
ssh <PI_USER>@<PI_LAN_IP>
```

Update the OS and install a few tools:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl qrencode openssl
```

Check the architecture and OS:

```bash
uname -m
cat /etc/os-release | grep PRETTY_NAME
```

Example output:

```
aarch64
PRETTY_NAME="Debian GNU/Linux 13 (trixie)"
```

Make sure nothing else is already using port 443:

```bash
sudo ss -tlnp | grep ':443 '
```

Expected: **no output** (port is free).

## 4. Dynamic DNS with DuckDNS

Home IPs change, so a DuckDNS hostname always points at the current one.

1. Sign in at duckdns.org, create a subdomain (`<YOUR_SUBDOMAIN>`), and copy your token.
2. On the Pi, create the update script:

```bash
mkdir -p ~/duckdns && cd ~/duckdns
cat > duck.sh <<'EOF'
#!/bin/sh
echo url="https://www.duckdns.org/update?domains=<YOUR_SUBDOMAIN>&token=<DUCKDNS_TOKEN>&ip=" | curl -k -o ~/duckdns/duck.log -K -
EOF
chmod 700 duck.sh
./duck.sh
cat duck.log
```

Expected output:

```
OK
```

(`KO` means the subdomain or token is wrong.)

3. Run it every 5 minutes with cron:

```bash
(crontab -l 2>/dev/null; echo '*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1') | crontab -
crontab -l
```

Expected output:

```
*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
```

4. Check that the hostname resolves to your home public IP:

```bash
getent hosts <YOUR_SUBDOMAIN>.duckdns.org
curl -s https://ifconfig.me; echo
```

Both should show the **same** public IP.

## 5. Install Xray

Use the official XTLS installer:

```bash
sudo bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

Example end of output:

```
info: Xray v26.11.9 is installed.
You may need to execute a command to remove dependent software: apt purge curl unzip
Please execute the command: systemctl enable xray; systemctl start xray
```

Check the version and service:

```bash
xray version
systemctl is-enabled xray
```

Example output:

```
Xray 26.11.9 (Xray, Penetrates Everything.) ... (go1.xx linux/arm64)
enabled
```

The installer puts the config at `/usr/local/etc/xray/config.json` and runs the service as the `xray` user.

## 6. Generate IDs and keys

**One UUID per device** (so a lost device can be removed on its own):

```bash
xray uuid   # iphone
xray uuid   # ipad
xray uuid   # macbook
```

Example output (each run prints a new random ID):

```
xxxxxxxx-xxxx-4xxx-xxxx-xxxxxxxxxxxx
```

**REALITY key pair:**

```bash
xray x25519
```

Example output (recent Xray versions; older ones print `Private key:` / `Public key:` instead):

```
PrivateKey: <PRIVATE_KEY>
Password: <PUBLIC_KEY>
Hash32: ...
```

- `PrivateKey` goes in the **server** config.
- `Password` (the public key) goes in the **client** links (`pbk=`).

**Short ID** (8 bytes of hex):

```bash
openssl rand -hex 8
```

Example output:

```
<SHORT_ID>     # 16 hex characters, e.g. a1b2c3d4e5f60718
```

Save these somewhere private (not in this repo).

## 7. Choose a REALITY target

The target must be a real site that supports **TLS 1.3 and HTTP/2**, works **without SNI checks**, and is not blocked
in the country you will travel to. Test candidates from the Pi:

```bash
xray tls ping www.samsung.com
```

Example (good) output, trimmed:

```
Pinging without SNI
Handshake succeeded
TLS Version: TLS 1.3
ALPN: h2
...
Pinging with SNI
Handshake succeeded
TLS Version: TLS 1.3
ALPN: h2
Cert's allowed domains: [... www.samsung.com ...]
```

What I found:

| Target | Result |
|---|---|
| `www.microsoft.com` | Works, but Shadowrocket showed a "GFW" warning for it, so I avoided it |
| `www.nvidia.com`, `www.logitech.com` | Failed the **without SNI** handshake |
| `www.samsung.com` | Passed both tests, **used** |

## 8. Write the server config

```bash
sudo nano /usr/local/etc/xray/config.json
```

Contents (fill in your own values):

```json
{
  "log": { "loglevel": "warning" },
  "inbounds": [
    {
      "listen": "0.0.0.0",
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          { "id": "<UUID_IPHONE>",  "flow": "xtls-rprx-vision", "email": "iphone" },
          { "id": "<UUID_IPAD>",    "flow": "xtls-rprx-vision", "email": "ipad" },
          { "id": "<UUID_MACBOOK>", "flow": "xtls-rprx-vision", "email": "macbook" }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "dest": "www.samsung.com:443",
          "serverNames": ["www.samsung.com"],
          "privateKey": "<PRIVATE_KEY>",
          "shortIds": ["<SHORT_ID>"]
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls", "quic"]
      }
    }
  ],
  "outbounds": [
    { "protocol": "freedom", "tag": "direct" },
    { "protocol": "blackhole", "tag": "block" }
  ]
}
```

Lock down the file (it contains the private key):

```bash
sudo chown root:xray /usr/local/etc/xray/config.json
sudo chmod 640 /usr/local/etc/xray/config.json
ls -l /usr/local/etc/xray/config.json
```

Example output:

```
-rw-r----- 1 root xray 1234 Nov 24 20:00 /usr/local/etc/xray/config.json
```

## 9. Test and start the service

Validate the config first:

```bash
sudo xray run -test -config /usr/local/etc/xray/config.json
```

Expected output ends with:

```
Configuration OK.
```

Start it and enable at boot:

```bash
sudo systemctl enable --now xray
sudo systemctl restart xray
systemctl is-active xray
sudo ss -tlnp | grep ':443 '
```

Expected output:

```
active
LISTEN 0  4096  *:443  *:*  users:(("xray",pid=1234,fd=3))
```

If it is not active, read the log:

```bash
sudo journalctl -u xray -n 50 --no-pager
```

## 10. Router: port forward and DHCP reservation

On the Verizon router (`https://192.168.1.1`, password on the router sticker):

1. **Port forward:** Advanced → Security & Firewall → Port Forwarding

   | Name | Port | Protocol | Forward to |
   |---|---|---|---|
   | Xray | 443 | TCP | `<PI_LAN_IP>`:443 |

   (Leave the router's own built-in system rules alone.)

2. **DHCP reservation:** Advanced → Network Settings → IP Address Distribution → give the Pi a static lease for
   `<PI_LAN_IP>`. Without this, the Pi may get a new address after a reboot and the port forward will stop working.

Check from **outside** your home network (e.g. phone on mobile data, using a TCP port-check app or site) that
`<YOUR_SUBDOMAIN>.duckdns.org:443` is open.

## 11. Create client links and QR codes

A VLESS REALITY link has this shape (one per device, using that device's UUID):

```
vless://<UUID_IPHONE>@<YOUR_SUBDOMAIN>.duckdns.org:443?encryption=none&flow=xtls-rprx-vision&security=reality&sni=www.samsung.com&fp=chrome&pbk=<PUBLIC_KEY>&sid=<SHORT_ID>&type=tcp#Home-Reality
```

Save each link in a private file on the Pi:

```bash
umask 077
echo 'vless://<UUID_IPHONE>@...#Home-Reality'          > ~/xray-client.txt
echo 'vless://<UUID_IPAD>@...#Home-Reality-ipad'       > ~/xray-client-ipad.txt
echo 'vless://<UUID_MACBOOK>@...#Home-Reality-macbook' > ~/xray-client-macbook.txt
ls -l ~/xray-client*.txt
```

Example output:

```
-rw------- 1 <PI_USER> <PI_USER> 260 Nov 24 20:10 /home/<PI_USER>/xray-client-ipad.txt
-rw------- 1 <PI_USER> <PI_USER> 263 Nov 24 20:10 /home/<PI_USER>/xray-client-macbook.txt
-rw------- 1 <PI_USER> <PI_USER> 255 Nov 24 20:10 /home/<PI_USER>/xray-client.txt
```

Show a link as a QR code in the terminal to scan with the device:

```bash
qrencode -t ansiutf8 < ~/xray-client-ipad.txt
```

A QR code made of block characters is printed in the terminal.

## 12. Set up Shadowrocket (iPhone / iPad / Mac)

1. Open Shadowrocket → tap **+** (or the scan icon) → scan the QR code for that device. A server appears
   (`Home-Reality`, `Home-Reality-ipad`, `Home-Reality-macbook`).
2. Check its settings: Type VLESS, TLS/Reality on, SNI `www.samsung.com`, Fingerprint `chrome`, Flow
   `xtls-rprx-vision`.
3. On the Mac: install Shadowrocket from the App Store (Apple Silicon Macs, same Apple ID), then add the macbook link
   (paste it, or scan the QR code with the iPhone and share it).
4. Select the server and turn the switch on. Accept the VPN configuration prompt the first time.

## 13. China split routing (`china-direct.conf`)

The goal: Chinese apps and sites connect **directly** (fast, local), everything else goes through home.

Import a Shadowrocket config file (Config → **+** / import → select the file → **Use Config**), then set
**Global Routing = Config**. The rules used are:

```ini
[General]
bypass-system = true
ipv6 = false
dns-server = system, 223.5.5.5, 119.29.29.29
skip-proxy = 192.168.0.0/16, 10.0.0.0/8, 172.16.0.0/12, localhost, *.local, captive.apple.com

[Rule]
# Chinese apps (examples; extend as needed)
DOMAIN-SUFFIX,qq.com,DIRECT
DOMAIN-SUFFIX,weixin.qq.com,DIRECT
DOMAIN-SUFFIX,wechat.com,DIRECT
DOMAIN-SUFFIX,alipay.com,DIRECT
DOMAIN-SUFFIX,taobao.com,DIRECT
DOMAIN-SUFFIX,tmall.com,DIRECT
DOMAIN-SUFFIX,jd.com,DIRECT
DOMAIN-SUFFIX,baidu.com,DIRECT
DOMAIN-SUFFIX,ctrip.com,DIRECT
DOMAIN-SUFFIX,qunar.com,DIRECT
DOMAIN-SUFFIX,fliggy.com,DIRECT
DOMAIN-SUFFIX,doubao.com,DIRECT
DOMAIN-SUFFIX,amap.com,DIRECT
DOMAIN-SUFFIX,cebbank.com,DIRECT
DOMAIN-SUFFIX,cgbchina.com.cn,DIRECT
# All .cn domains
DOMAIN-SUFFIX,cn,DIRECT
# LAN / private ranges (hotel Wi-Fi login pages)
IP-CIDR,192.168.0.0/16,DIRECT
IP-CIDR,10.0.0.0/8,DIRECT
IP-CIDR,172.16.0.0/12,DIRECT
# China IP addresses
GEOIP,CN,DIRECT
# Everything else through the home server
FINAL,PROXY
```

> The exact domain list in my real file is longer; the above is a representative version. If a Chinese app shows a
> US location or is slow, find its domain in Shadowrocket's log (marked PROXY) and add
> `DOMAIN-SUFFIX,<domain>,DIRECT`.

## 14. Verify everything

1. **On the Pi:**

   ```bash
   systemctl is-active xray
   cat ~/duckdns/duck.log
   ```

   Expected:

   ```
   active
   OK
   ```

2. **From outside:** iPhone on **mobile data** (Wi-Fi off) → Shadowrocket on → open a "what is my IP" site.
   It should show your **home** public IP.
3. **Routing:** in Shadowrocket → Data / Log, Chinese sites show **DIRECT**, other sites show **PROXY**.
4. Also test once on a public Wi-Fi network and via iPhone Personal Hotspot for the Mac.

## 15. Maintenance and troubleshooting

See [`home-xray-setup-prompt.md`](home-xray-setup-prompt.md) for the full checklist. In short:

**Monthly:**

```bash
# Update Xray (keeps config, keys and links)
sudo bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
systemctl is-active xray        # should print: active

# Update the OS
sudo apt update && sudo apt upgrade -y
```

Also update Shadowrocket on every device and check `cat ~/duckdns/duck.log` shows `OK`.

**Before travelling:** do the updates, test, reboot the Pi once and confirm `systemctl is-active xray` prints
`active`, and **then stop changing the Pi** until you are home. A failed update can't be fixed remotely.

**Remove a lost device** (example: `ipad`):

```bash
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

Expected output ends with:

```
Configuration OK.
active
```

**Quick troubleshooting:**

| Symptom | Check |
|---|---|
| Nothing connects anywhere | `systemctl status xray`; `sudo journalctl -u xray -n 50 --no-pager` |
| Works at home but not outside | Router port forward TCP 443 → `<PI_LAN_IP>`; Pi's IP reservation still in place |
| Stopped after home IP changed | `cat ~/duckdns/duck.log` shows `OK`; `crontab -l` still has the `duck.sh` line |
| Connects but some sites fail | Update Xray + Shadowrocket; re-run `xray tls ping www.samsung.com` |
| Blocked abroad after working | Try a different REALITY target (test with `xray tls ping`), or get a new home IP |

---

## Disclaimer

This is a personal setup shared for learning. Make sure using a proxy is permitted by your ISP and by the laws of
the country you are in.
