# Quick Setup: Ubuntu VPN Wi-Fi Gateway

This is the concise setup reference for sharing an Ubuntu laptop's Proton VPN connection with devices connected to a D-Link DSL-124 R1 access point.

## 1. Network layout

```text
Fiber router
    |
 Wi-Fi (wlo1)
    |
Ubuntu laptop
  |         |
tun0      eno1
VPN       10.42.0.1/24
            |
       D-Link DSL-124 R1
       AP management: 10.42.0.2
       DHCP: disabled
            |
       Wi-Fi clients
```

Completed machine-specific values:

| Component | Value |
|---|---|
| Upstream Wi-Fi | `wlo1` |
| Downstream Ethernet | `eno1` |
| VPN interface | `tun0` |
| Ubuntu Ethernet IP | `10.42.0.1/24` |
| D-Link management IP | `10.42.0.2/24` |
| Client DHCP range | `10.42.0.10`–`10.42.0.254` |
| Client gateway/DNS | `10.42.0.1` |
| VPN DNS | `10.96.0.1` |

Your interface names and VPN server details may differ. Check before applying commands.

## 2. D-Link access point settings

Connect Ubuntu Ethernet to a **LAN** port on the D-Link, not the DSL/WAN port.

- LAN IP: `10.42.0.2`
- Subnet mask: `255.255.255.0`
- DHCP server: Disabled
- DHCP relay: Disabled
- DNS relay/proxy: Disabled, if present
- Configure Wi-Fi SSID and WPA2-Personal/AES security as desired

Ubuntu provides DHCP and DNS for the downstream network. The D-Link only bridges Wi-Fi clients to Ethernet.

## 3. Ubuntu Ethernet sharing

The completed NetworkManager profile was named `netplan-eno1` and used:

- IPv4 method: `shared`
- IPv4 address: `10.42.0.1/24`

Inspect it:

```bash
nmcli connection show
nmcli connection show "netplan-eno1"
```

If this is the correct existing profile to modify:

```bash
sudo nmcli connection modify "netplan-eno1" \
  ipv4.method shared \
  ipv4.addresses 10.42.0.1/24

sudo nmcli connection up "netplan-eno1"
```

NetworkManager shared mode starts DHCP/DNS service and configures IPv4 masquerading. Do not run a second DHCP server on the D-Link.

## 4. Enable IPv4 forwarding persistently

Check:

```bash
sysctl net.ipv4.ip_forward
```

Enable immediately:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Persist across reboot:

```bash
sudo vim /etc/sysctl.d/99-vpn-gateway.conf
```

Add:

```ini
net.ipv4.ip_forward=1
```

Apply:

```bash
sudo sysctl --system
```

## 5. Connect Proton VPN

The completed setup used Proton OpenVPN through NetworkManager. The connection profile was `protonvpn-openvpn-connection`, and the tunnel interface was `tun0`.

Install the NetworkManager OpenVPN integration if needed:

```bash
sudo apt update
sudo apt install network-manager-openvpn network-manager-openvpn-gnome
```

Import a current Proton `.ovpn` configuration into NetworkManager and connect using the appropriate Proton OpenVPN credentials. Do not reuse another server's IP or configuration blindly.

Check:

```bash
nmcli connection show --active
ip addr show tun0
ip route
resolvectl status
```

## 6. Forwarding kill switch

Create the rules file:

```bash
sudo vim /etc/vpn-gateway.nft
```

Contents:

```nft
table inet vpn_gateway {
    chain forward_guard {
        type filter hook forward priority -10; policy accept;

        iifname "eno1" oifname "tun0" meta nfproto ipv4 accept
        iifname "eno1" drop

        iifname "tun0" oifname "eno1" ct state established,related accept
        oifname "eno1" drop
    }
}
```

Validate and load:

```bash
sudo nft -c -f /etc/vpn-gateway.nft
sudo nft -f /etc/vpn-gateway.nft
```

Inspect:

```bash
sudo nft list table inet vpn_gateway
```

This protects forwarded downstream client traffic: IPv4 traffic from `eno1` may leave through `tun0`, while other forwarded traffic from `eno1` is dropped. Return traffic from `tun0` is allowed. Client IPv6 forwarding is blocked.

**Limitation:** These rules do not provide a kill switch for applications running directly on Ubuntu. They protect forwarded client traffic.

If the table already exists and you are replacing its chain rules, avoid blindly reloading the file. In the completed setup, the existing chain was cleared before reloading:

```bash
sudo nft flush chain inet vpn_gateway forward_guard
sudo nft -f /etc/vpn-gateway.nft
```

Use only if the table and chain already exist.

## 7. Persist the firewall with systemd

Create:

```bash
sudo vim /etc/systemd/system/vpn-gateway-firewall.service
```

Contents:

```ini
[Unit]
Description=VPN Gateway Firewall
Before=NetworkManager.service

[Service]
Type=oneshot
ExecStart=/usr/sbin/nft -f /etc/vpn-gateway.nft
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now vpn-gateway-firewall.service
```

Check:

```bash
systemctl status vpn-gateway-firewall.service
systemctl is-enabled vpn-gateway-firewall.service
sudo nft list table inet vpn_gateway
```

`active (exited)` is normal for this oneshot service.

**Do not enable the generic `nftables.service` on the completed machine:** its `/etc/nftables.conf` contained `flush ruleset`, which could remove NetworkManager, Docker, and libvirt firewall rules. The dedicated service loads only the VPN gateway table.

## 8. Verify the setup

On Ubuntu:

```bash
ip -br addr
ip route
nmcli connection show --active
sysctl net.ipv4.ip_forward
resolvectl status
sudo nft list table inet vpn_gateway
```

On a downstream phone/client, check:

- IP address belongs to `10.42.0.0/24`.
- Gateway is `10.42.0.1`.
- DNS is `10.42.0.1` (Automatic via DHCP is fine).

Then test:

1. Connect the VPN; client internet should work.
2. Check the public IP from Ubuntu and the client; both should show the VPN exit IP.
3. Disconnect the VPN; downstream client internet should stop.
4. Local access to Ubuntu/D-Link should remain available.
5. Reconnect the VPN; client internet should return.
6. Reboot Ubuntu and repeat the checks.

## 9. Troubleshooting reference

| Symptom | First checks |
|---|---|
| Client gets no IP | D-Link DHCP off; Ethernet profile active/shared; cable in LAN port |
| Client has IP but no internet | VPN state, `ip route`, forwarding, nftables, NAT |
| DNS fails | Client DNS `10.42.0.1`; `resolvectl status`; DNS listeners |
| D-Link admin page unavailable | D-Link `10.42.0.2`; Ubuntu `10.42.0.1`; same `/24`; LAN-to-LAN |
| Firewall service fails | `journalctl -u vpn-gateway-firewall.service -b` |

Useful inspection commands:

```bash
ip -br addr
ip route
nmcli device status
nmcli connection show --active
resolvectl status
ss -lntup | grep ':53'
sudo nft list ruleset
journalctl -u NetworkManager -b
```

Do not run `nft flush ruleset` as a general troubleshooting step; it can remove firewall rules owned by other services.

## 10. Proton Free plan: multiple clients

The Ubuntu laptop makes one Proton VPN connection. Downstream devices share that existing tunnel through NAT; they do not each create their own Proton VPN session. So several devices behind this gateway use the laptop's one VPN connection. A second device independently connecting to Proton VPN would be another simultaneous connection.
