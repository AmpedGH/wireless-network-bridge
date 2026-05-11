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

---

## License

MIT
