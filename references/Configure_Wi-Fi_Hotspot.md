# Install Dependency
```bash
sudo apt update
sudo apt install hostapd dnsmasq net-tools

sudo systemctl stop hostapd
sudo systemctl stop dnsmasq
sudo systemctl disable hostapd
sudo systemctl disable dnsmasq
sudo systemctl reset-failed hostapd
sudo systemctl reset-failed dnsmasq
```
# Create Configuration - hostapd.conf
```bash
sudo nano /etc/hostapd/hostapd.conf
```

```
interface=wlo1
driver=nl80211
ssid=EPC-WCL-ROS2
country_code=TW
ieee80211d=1

hw_mode=g
channel=1

wmm_enabled=1
ieee80211n=1

auth_algs=1
wpa=2
wpa_passphrase=70604376
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP

```

# Create Configuration - hotspot.conf
```bash
sudo nano /etc/dnsmasq.d/hotspot.conf
```

```
interface=wlo1
dhcp-range=192.168.0.101,192.168.0.150,12h
bind-interfaces
domain-needed
bogus-priv
```

# Directly exclude wlo1 - NetworkManager
```bash
sudo nano /etc/NetworkManager/conf.d/wifi-hotspot.conf
```

```
[keyfile]
unmanaged-devices=interface-name:wlo1
```



# Create Wi-Fi Hotspot Service
```bash
sudo nano /etc/systemd/system/wifi-hotspot.service
```

```
[Unit]
Description=Wi-Fi Hotspot Stack (standalone hostapd+dnsmasq)
Wants=network-online.target
After=network-online.target NetworkManager.service
ConditionPathExists=/sys/class/net/wlo1

StartLimitIntervalSec=0

[Service]
Type=oneshot
RemainAfterExit=yes

# Type=oneshot, no need Resart Policy
Restart=no

ExecStart=/bin/bash -euxo pipefail -c '\
  # 1. Unblock Wi-Fi \
  /usr/sbin/rfkill unblock all || true; \
  \
  # 2. Wait until interface exists, then bring it up \
  for i in {1..60}; do \
    ip link show wlo1 >/dev/null 2>&1 && break; \
    sleep 1; \
  done; \
  ip link set wlo1 up; \
  \
  # 3. Assign hotspot IP \
  ip addr replace 192.168.0.100/24 dev wlo1; \
  \
  # 4. Start dnsmasq (standalone) \
  pkill -x dnsmasq 2>/dev/null || true; \
  /usr/sbin/dnsmasq --conf-file=/dev/null --conf-dir=/etc/dnsmasq.d --pid-file=/run/dnsmasq-hotspot.pid; \
  \
  # 5) start hostapd (standalone) \
  pkill -x hostapd 2>/dev/null || true; \
  /usr/sbin/hostapd -B -P /run/hostapd-hotspot.pid /etc/hostapd/hostapd.conf; \
  \
  # 6) Verify \
  test -s /run/dnsmasq-hotspot.pid; \
  test -s /run/hostapd-hotspot.pid; \
'

ExecStop=/bin/bash -euxo pipefail -c '\
  pkill -x hostapd 2>/dev/null || true; \
  pkill -x dnsmasq 2>/dev/null || true; \
  rm -f /run/hostapd-hotspot.pid /run/dnsmasq-hotspot.pid; \
'

[Install]
WantedBy=multi-user.target

```

# Start Wi-Fi Hotspot
```bash
sudo systemctl daemon-reload
sudo systemctl enable wifi-hotspot.service
sudo reboot
```

# Verify
```bash
# Expected output: inet 192.168.0.100/24
ip addr show wlo1

# Expected output: type AP
iw dev wlo1 info

sudo systemctl status wifi-hotspot.service

ip addr show wlo1

iw dev wlo1 info

pgrep -a hostapd
pgrep -a dnsmasq
```