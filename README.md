# Wireless Network Bridge

A Raspberry Pi 3B+ configured as a wireless network bridge using an Alfa Wi-Fi adapter, extending internet access over a long distance via ethernet to an Arris G34 gateway. The Pi acts as a router — sharing its wireless connection through its ethernet port using NAT and IP forwarding.

---

## Hardware Requirements

| Component | Link |
|---|---|
| Raspberry Pi 3B+ | [Amazon](https://www.amazon.com/Raspberry-Pi-Model-Board-Plus/dp/B0BNJPL4MW/) |
| Alfa Wi-Fi Adapter AWUS036AXML (AXE3000) | [Amazon](https://www.amazon.com/ALFA-AWUS036AXML-802-11axe-Adapter-AXE3000/dp/B0BY8GMW32/) |
| Arris Surfboard G34 Gateway | [Amazon](https://www.amazon.com/ARRIS-Surfboard-G34-Approved-Spectrum/dp/B096W716BY/) |
| Cat6 UTP Ethernet Cable | [Amazon](https://www.amazon.com/Jadaol-Ethernet-High-Speed-Internet-Streaming/dp/B00WD017GQ/) |
| RJ45 Passthrough Connectors | [Amazon](https://www.amazon.com/Jadaol-Ethernet-High-Speed-Internet-Streaming/dp/B00WD017GQ/) |
| RJ45 Crimp Tool | [Amazon](https://www.amazon.com/dp/B0B69B94T2) |
| 5V Power Supply for Raspberry Pi | [Amazon](https://www.amazon.com/2-5A-Adapter-Raspberry-Power-Supply/dp/B09TT2N3WR/) |
| **8GB Micro SD Card** | [Amazon](https://www.amazon.com/SanDisk-Ultra-microSDHC-UHS-I-Adapter/dp/B073K14966/) |

---

## OS

**Raspberry Pi OS Lite (Debian Bookworm, 64-bit)**  
No desktop environment — headless/terminal only  
Flash to SD card using [Balena Etcher](https://etcher.balena.io/)  
Download: [raspberrypi.com/software](https://www.raspberrypi.com/software/)

> ⚠️ Make sure to select **Raspberry Pi OS Lite (64-bit)** specifically — not the full desktop version.

---

## How It Works

```
[Internet] → [Wi-Fi Network] → [Alfa Adapter on wlan1] → [Raspberry Pi] → [eth0 NAT] → [Ethernet Cable] → [Arris G34 LAN Port]
```

The Pi connects to Wi-Fi via the Alfa adapter (`wlan1`), then shares that connection through its ethernet port (`eth0`) using NetworkManager's shared mode, which handles NAT, IP forwarding, and DHCP automatically.

---

## Installation



### 1. Update Packages and Reinstall Keys

```bash
sudo apt update
sudo apt install --reinstall raspberrypi-archive-keyring
```

---

### 2. Install Build Tools and Kernel Headers

```bash
sudo apt install git build-essential dkms raspberrypi-kernel-headers -y
```

---
 
### 3. Note Your Current Adapters Before Installing Driver

```bash
iwconfig
```
 

Make note of what interfaces exist now — after installing the driver you should see one additional adapter appear.

 

---

### 4. Install Alfa Adapter Driver (RTL8812AU)

Clone and install the driver for the Alfa AWUS036AXML adapter:

```bash
git clone https://github.com/aircrack-ng/rtl8812au.git
cd rtl8812au
sudo make dkms_install
```
 
> ⚠️ Use `sudo make dkms_install` — do NOT use `sudo ./dkms-install.sh`
 

When prompted, select the default options and allow the DKMS install to complete.

---

### 5. Verify the Adapter is Detected

After install and reboot, confirm both interfaces are visible:

```bash
ip a
iwconfig
```

You should see `wlan0` (onboard Wi-Fi) and `wlan1` (Alfa adapter).

---

### 6. Enable IP Forwarding

```bash
sudo sysctl -w net.ipv4.ip_forward=1

  
```

Make it permanent across reboots:

```bash
sudo nano /etc/sysctl.conf
```

Uncomment or add this line:

```
net.ipv4.ip_forward=1
```

---

### 7. Configure NetworkManager for NAT and DHCP Sharing

This single command sets up NAT, IP forwarding, and DHCP on the ethernet interface automatically:

```bash
sudo nmcli connection add type ethernet ifname eth0 con-name eth0-shared ipv4.method shared
```
 
---

### 8. Clean Up Duplicate Connections (if needed)

If a default wired connection already exists, remove it to avoid conflicts:

```bash
nmcli connection show
sudo nmcli connection delete "Wired connection 1"
nmcli connection show
```

Confirm only `eth0-shared` remains for ethernet.
 
---

### 9. Configure the Arris G34 Router

- Log into the Arris G34 admin panel
- **Disable DHCP** on both IPv4 and IPv6 settings — the Pi handles DHCP now
- Connect an ethernet cable from the Pi's ethernet port (`eth0`) to one of the **LAN ports** on the Arris (labeled 1–4, not the WAN port)

---

### 10. Reboot Both Devices

```bash
sudo reboot
```

After reboot, devices connected to the Arris router will receive IPs from the Pi and route internet traffic through the Alfa adapter.

---

## Verify It's Working

Check the shared connection is active:

```bash
nmcli connection show
nmcli device show eth0
```

---

## Troubleshooting

**Alfa adapter not showing as wlan1**
- Confirm the driver installed correctly: `dkms status`
- Replug the USB adapter and run `ip a` again

**Connected devices not getting an IP**
- Confirm DHCP is disabled on the Arris G34
- Confirm ethernet is plugged into a LAN port, not the WAN port
- Restart NetworkManager: `sudo systemctl restart NetworkManager`

**No internet on connected devices**
- Confirm IP forwarding is active: `cat /proc/sys/net/ipv4/ip_forward` (should return `1`)
- Confirm the Pi itself has internet: `ping 8.8.8.8`

---

## Notes

- The Alfa AWUS036AXML uses the RTL8812AU chipset — the `8812au-20210629` driver from morrownr is the most stable option for Raspberry Pi OS Bookworm
- NetworkManager's `shared` mode automatically handles NAT via iptables — no manual iptables rules needed
- PoE is preferred for powering the Pi in a permanent installation to reduce cable clutter


## Optional: Mullvad VPN Gateway

Route all connected devices through Mullvad VPN automatically — no VPN app needed on individual devices.

### 1. Install WireGuard

```bash
sudo apt install wireguard openresolv -y
```

### 2. Add Your Mullvad Config

Download a WireGuard config from [mullvad.net](https://mullvad.net) → Account → WireGuard configuration → Linux. Copy the contents and paste into:

```bash
sudo nano /etc/wireguard/mullvad1.conf
```

### 3. Start the VPN

```bash
sudo wg-quick up mullvad1
sudo systemctl enable wg-quick@mullvad1
```

### 4. Update Firewall Rules

```bash
sudo apt install iptables -y
sudo iptables -t nat -A POSTROUTING -o mullvad1 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -o mullvad1 -j ACCEPT
sudo iptables -A FORWARD -i mullvad1 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo apt install iptables-persistent -y
```

### 5. Refresh VPN Connection

```bash
sudo systemctl stop wg-quick@mullvad1 && sudo systemctl start wg-quick@mullvad1
```
---
## Kill Switch (VPN Leak Protection)

By default, if the Mullvad VPN drops, devices will fall back to unencrypted internet without warning. The kill switch blocks all traffic if the VPN tunnel goes down.

### Enable Kill Switch

```bash
sudo iptables -A FORWARD -i eth0 -o wlan1 -j DROP
sudo iptables -A FORWARD -i wlan1 -o eth0 -j DROP
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

### Test It Works

Bring the VPN down:
```bash
sudo systemctl stop wg-quick@mullvad1
```
Devices on the Arris should lose internet completely — not fall back to unencrypted traffic.

Bring it back:
```bash
sudo systemctl start wg-quick@mullvad1
```

## VPN Watchdog (Auto-Restart on High Ping / Packet Loss)

A bash script that monitors the VPN connection every 5 minutes and automatically restarts it if ping exceeds 200ms or packet loss exceeds 20%.

### Create the Script

```bash
sudo nano /usr/local/bin/vpn-watchdog.sh
```

Paste the following:

```bash
#!/bin/bash

PING_TARGET="8.8.8.8"
PACKET_LOSS_THRESHOLD=20
PING_THRESHOLD=200
VPN_INTERFACE="mullvad1"
LOG="/var/log/vpn-watchdog.log"

PING_OUTPUT=$(ping -c 10 -W 2 $PING_TARGET 2>&1)
LOSS=$(echo "$PING_OUTPUT" | grep -oP '\d+(?=% packet loss)')
AVG_PING=$(echo "$PING_OUTPUT" | grep -oP '(?<=/)[\d.]+(?=/)' | tail -1)

echo "$(date) — Loss: ${LOSS}% Ping: ${AVG_PING}ms" >> $LOG

if [ "${LOSS:-100}" -ge "$PACKET_LOSS_THRESHOLD" ] || [ "${AVG_PING%.*}" -ge "$PING_THRESHOLD" ]; then
    echo "$(date) — VPN degraded. Restarting..." >> $LOG
    systemctl stop wg-quick@$VPN_INTERFACE
    sleep 3
    systemctl start wg-quick@$VPN_INTERFACE
    echo "$(date) — VPN restarted." >> $LOG
fi
```

### Make it Executable

```bash
sudo chmod +x /usr/local/bin/vpn-watchdog.sh
```

### Schedule via Cron (Every 5 Minutes)

```bash
sudo crontab -e
```
Add this line:

```bash
*/5 * * * * /usr/local/bin/vpn-watchdog.sh
```


## License

MIT
