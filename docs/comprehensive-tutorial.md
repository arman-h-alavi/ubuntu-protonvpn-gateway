# Ubuntu VPN Wi-Fi Gateway: Complete Setup and Explanation

This guide documents the completed setup in which an Ubuntu laptop connects to the internet over Wi-Fi, establishes a Proton VPN tunnel, and shares that VPN-protected connection through Ethernet to a D-Link DSL-124 R1 access point.

It covers the topology, addressing, access-point configuration, NetworkManager sharing, DHCP/DNS, routing, NAT, the forwarding kill switch, persistence, verification, troubleshooting, and a Windows alternative.

> **Important:** This guide describes the configuration that was tested in the original setup. Interface names, connection names, VPN server addresses, and some settings are machine-specific. Check them on your own system before copying commands.

---

## 1. Final network topology

```text
                     Internet
                        |
                 Fiber router
                 192.168.100.1
                        |
                 Wi-Fi connection
                        |
                 Ubuntu laptop
          +-------------+-------------+
          |                           |
      Upstream Wi-Fi              Ethernet
         wlo1                       eno1
  192.168.100.12/24             10.42.0.1/24
          |                           |
     Proton OpenVPN                D-Link DSL-124 R1
       tun0                         Access point only
     10.96.0.11                    10.42.0.2/24
          |                           |
          +---- VPN tunnel             Wi-Fi: VPN-Gateway-WiFi
                                      /    |     \
                                   Phone Tablet Other clients
                                  10.42.0.x addresses
```

Traffic from a client follows this path:

```text
Client -> D-Link AP (Layer 2 bridge) -> Ubuntu eno1
       -> Ubuntu routing/NAT -> Proton VPN tun0 -> Internet
```

The D-Link is not acting as a router in this design. It provides Wi-Fi access and bridges wireless clients to the Ethernet network connected to Ubuntu.

### Why this design?

- The Ubuntu laptop is the VPN gateway.
- Only the laptop establishes a Proton VPN connection.
- Multiple downstream devices share that one tunnel.
- NetworkManager provides DHCP and DNS forwarding for the downstream network.
- An nftables forwarding rule set prevents downstream client traffic from escaping through the ordinary upstream Wi-Fi route when the VPN tunnel is unavailable.

---

## 2. Addressing and subnetting

The completed setup used the following addresses:

| Component | Interface / role | Address |
|---|---|---|
| Main fiber router | Upstream gateway | `192.168.100.1` |
| Ubuntu upstream Wi-Fi | `wlo1` | `192.168.100.12/24` |
| Ubuntu downstream Ethernet | `eno1` | `10.42.0.1/24` |
| Proton VPN tunnel | `tun0` | `10.96.0.11/16` |
| Proton VPN DNS | DNS server on tunnel | `10.96.0.1` |
| D-Link access point | Management address | `10.42.0.2/24` |
| Downstream clients | DHCP pool | `10.42.0.10`–`10.42.0.254` |

### What does `/24` mean?

An IPv4 address with `/24` has a subnet mask of `255.255.255.0`. In the `10.42.0.0/24` network:

- Network address: `10.42.0.0`
- Usable host addresses: `10.42.0.1` through `10.42.0.254`
- Broadcast address: `10.42.0.255`

Ubuntu uses `10.42.0.1` as the gateway for downstream clients. The D-Link uses `10.42.0.2` as its own management address. DHCP starts at `.10`, so neither static address is handed out to a client.

The upstream network (`192.168.100.0/24`) and downstream network (`10.42.0.0/24`) are different subnets. This allows Ubuntu to route between them.

### Why give the D-Link `10.42.0.2`?

The D-Link needs an address on the same LAN as the Ubuntu Ethernet interface so that you can open its administration page. The address must be unique:

- `.1` is already Ubuntu's downstream gateway.
- `.2` is reserved for the D-Link.
- `.10` and above are available for DHCP clients.

The D-Link management address is not the clients' default gateway. Their default gateway is Ubuntu at `10.42.0.1`.

---

## 3. Configure the D-Link DSL-124 R1 as an access point

Connect an Ethernet cable from Ubuntu's `eno1` port to a **LAN** port on the D-Link (LAN1 in the tested setup). Do not use the DSL port. The intent is to use the device as a simple access point/bridge, not as a second router.

Configure the D-Link as follows:

| Setting | Value |
|---|---|
| LAN IP address | `10.42.0.2` |
| Subnet mask | `255.255.255.0` |
| DHCP server | Disabled |
| DHCP relay | Disabled |
| DNS relay/proxy | Disabled, if this option is present |
| Wi-Fi SSID | `VPN-Gateway-WiFi` |
| Wi-Fi security | WPA2-Personal |
| Encryption | AES |

### Why disable the D-Link DHCP server?

Ubuntu NetworkManager shared mode runs a DHCP service for the downstream Ethernet network. If the D-Link also runs DHCP, clients may receive offers from two different DHCP servers. They could then receive inconsistent IP addresses, gateways, or DNS servers.

There should be one DHCP authority for this subnet: Ubuntu.

### Why disable DHCP relay?

A DHCP relay is useful when DHCP clients and the DHCP server are on different network segments. Here, Ubuntu is directly connected to the downstream LAN and runs the DHCP service on that same segment. A relay is unnecessary.

### Why disable DNS relay/proxy?

The intended DNS endpoint for clients is Ubuntu (`10.42.0.1`). Disabling an additional DNS proxy layer on the D-Link keeps DNS handling straightforward and avoids introducing another forwarding component.

### Why LAN-to-LAN instead of DSL/WAN?

The LAN-to-LAN connection makes the D-Link bridge wireless clients onto Ubuntu's Ethernet subnet. Using a routed WAN/DSL connection would create another IP network and potentially another NAT layer, which is not needed here.

---

## 4. Ubuntu network interfaces

The completed setup used:

- `wlo1`: upstream Wi-Fi connection to the fiber router.
- `eno1`: Ethernet connection to the D-Link.
- `tun0`: Proton OpenVPN tunnel.

Inspect interface names and addresses:

```bash
ip -br addr
ip route
```

Interface names may differ on another computer. Do not assume that `wlo1` and `eno1` will be the same everywhere.

### NetworkManager shared profile

The Ethernet connection profile was named:

```text
netplan-eno1
```

It was configured with:

- IPv4 method: `shared`
- IPv4 address: `10.42.0.1/24`

NetworkManager's shared mode is more than assigning an IP address. It sets up the basic services needed to share the connection, including DHCP/DNS service and IPv4 masquerading.

Inspect the profile:

```bash
nmcli connection show
nmcli connection show "netplan-eno1"
```

The essential settings can be inspected with:

```bash
nmcli -f connection.id,ipv4.method,ipv4.addresses connection show "netplan-eno1"
```

If you are recreating the configuration on a different system, first identify which network manager or Netplan configuration owns the Ethernet profile. Do not blindly create a second competing profile.

A typical modification, only if the profile exists and is the correct profile to edit, is:

```bash
sudo nmcli connection modify "netplan-eno1" \
  ipv4.method shared \
  ipv4.addresses 10.42.0.1/24
```

Then activate it:

```bash
sudo nmcli connection up "netplan-eno1"
```

Changing a live network profile can interrupt connectivity. Perform this locally or ensure you have another way to access the machine.

---

## 5. DHCP and dnsmasq

In NetworkManager shared mode, NetworkManager starts a dnsmasq process for the shared interface. In the completed setup, the relevant process listened on `10.42.0.1` and served the downstream range:

```text
10.42.0.10 - 10.42.0.254
```

The lease duration was 3600 seconds (one hour).

A simplified view of the DHCP information delivered to a client is:

| DHCP information | Value |
|---|---|
| Client IP | An available `10.42.0.x` address |
| Subnet mask | `255.255.255.0` |
| Default gateway | `10.42.0.1` |
| DNS server | `10.42.0.1` |

The actual NetworkManager-managed dnsmasq process in the completed setup used arguments similar to:

```text
/usr/sbin/dnsmasq
  --conf-file=/dev/null
  --no-hosts
  --keep-in-foreground
  --bind-interfaces
  --except-interface=lo
  --clear-on-reload
  --strict-order
  --listen-address=10.42.0.1
  --dhcp-range=10.42.0.10,10.42.0.254,3600
  --dhcp-leasefile=/var/lib/NetworkManager/dnsmasq-eno1.leases
  --pid-file=/run/nm-dnsmasq-eno1.pid
  --conf-dir=/etc/NetworkManager/dnsmasq-shared.d
```

This is an observation from the completed machine, not a command to run manually. NetworkManager manages this process.

There was also a separate dnsmasq process associated with libvirt on `192.168.122.1`. It is unrelated to the VPN gateway and should not be stopped or modified as part of this setup.

---

## 6. IPv4 forwarding and routing

A Linux machine normally routes packets between interfaces only when IPv4 forwarding is enabled.

Check the current setting:

```bash
sysctl net.ipv4.ip_forward
```

The expected result is:

```text
net.ipv4.ip_forward = 1
```

To enable it immediately:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

To make it persistent across reboot, create a dedicated sysctl configuration file:

```bash
sudo vim /etc/sysctl.d/99-vpn-gateway.conf
```

Put this line in the file:

```ini
net.ipv4.ip_forward=1
```

Apply it:

```bash
sudo sysctl --system
```

### Default routes

The completed setup had a VPN default route through `tun0` with a lower metric than the ordinary Wi-Fi default route. The physical Wi-Fi route remained available to reach the VPN server itself.

Inspect routes:

```bash
ip route
```

The observed routing arrangement included:

- Default route through Proton's tunnel gateway `10.96.0.1` on `tun0`.
- A normal upstream default route through `192.168.100.1` on `wlo1`, with a higher metric.
- A specific route to the VPN server (`VPN_SERVER_IP`) through the physical router on `wlo1`.

The VPN server's endpoint must remain reachable outside the tunnel so that the tunnel can be established. The endpoint route is therefore not itself evidence of a leak; the forwarding firewall below controls downstream client traffic.

Do not copy the observed VPN server IP into another setup. Proton server addresses can change.

---

## 7. NAT (masquerading)

Downstream clients use private addresses such as `10.42.0.10`. Those addresses are not directly routable on the public internet.

Network address translation (NAT) rewrites the source address of client packets as they leave Ubuntu through the VPN interface. The replies can then be mapped back to the originating client.

NetworkManager shared mode created a masquerading rule equivalent to:

```text
ip saddr 10.42.0.0/24 ip daddr != 10.42.0.0/24 masquerade
```

In practical terms:

- Match packets originating from the downstream subnet.
- Do not NAT traffic whose destination is still inside that same subnet.
- Masquerade the remaining traffic as it leaves the shared connection.

Do not add duplicate NAT rules without first inspecting the existing rules. NetworkManager manages its own firewall rules, and Docker/libvirt may also maintain rules.

Inspect nftables rules:

```bash
sudo nft list ruleset
```

Do not use `nft flush ruleset` on this machine as a casual troubleshooting step. It can remove rules created by NetworkManager, Docker, libvirt, and other services.

---

## 8. DNS path

The intended DNS configuration is:

```text
Downstream client
      |
      | DNS query to 10.42.0.1
      v
NetworkManager dnsmasq on Ubuntu
      |
      v
systemd-resolved stub (127.0.0.53)
      |
      v
Proton VPN DNS (10.96.0.1) via tun0
```

In the completed setup:

- Clients received `10.42.0.1` as their DNS server.
- NetworkManager dnsmasq listened on `10.42.0.1:53`.
- `/etc/resolv.conf` pointed to the local `127.0.0.53` stub.
- `systemd-resolved` showed `10.96.0.1` as the DNS server for `tun0`.
- The VPN link was configured as the default DNS route.

Useful checks:

```bash
resolvectl status
nmcli device show tun0
ss -lntup | grep ':53'
cat /etc/resolv.conf
```

The phone showing DNS as **Automatic** is expected. Automatic means it accepts the DNS server supplied by DHCP; it does not mean the phone independently chooses a public DNS server.

The checks above establish the configured resolver path. A packet capture would be needed to prove every individual DNS packet's exact path.

---

## 9. Forwarding kill switch with nftables

### Why a kill switch is necessary

Without a forwarding restriction, if the VPN tunnel disconnects, Ubuntu may still have its ordinary Wi-Fi route. Downstream client traffic could then be routed through `wlo1` directly to the fiber router.

The forwarding guard prevents that fallback for traffic being forwarded from the downstream Ethernet interface.

### Rules used in the completed setup

The file `/etc/vpn-gateway.nft` contained:

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

Meaning of the rules:

1. `table inet vpn_gateway`: creates a dedicated nftables table in the `inet` family, which can process IPv4 and IPv6.
2. `chain forward_guard`: creates a chain for forwarded packets.
3. `hook forward priority -10`: attaches the chain to the forwarding path, at priority `-10`.
4. `policy accept`: does not globally block forwarding; the explicit rules below determine the relevant traffic.
5. `iifname "eno1" oifname "tun0" meta nfproto ipv4 accept`: permits IPv4 packets arriving from downstream Ethernet and leaving through the VPN tunnel.
6. `iifname "eno1" drop`: drops other forwarded packets entering from downstream Ethernet, including attempts to leave through the ordinary Wi-Fi interface. It also blocks forwarded client IPv6 traffic.
7. `iifname "tun0" oifname "eno1" ct state established,related accept`: permits established or related return traffic from the VPN tunnel to downstream clients.
8. `oifname "eno1" drop`: drops other forwarded traffic destined for the downstream interface.

The chain is in the `forward` path. It does not block traffic addressed to Ubuntu itself, such as DHCP, DNS, or access to the D-Link management page. Those packets use the input path rather than the forwarding path.

### Important limitation

This is a kill switch for **forwarded downstream client traffic**. It does not, by itself, prevent applications running on the Ubuntu laptop from using its ordinary Wi-Fi route if the VPN disconnects. Protecting the host's own traffic requires a separate output-policy design and must be handled carefully so that the VPN endpoint remains reachable.

The rules also intentionally permit only IPv4 client traffic through `tun0`; client IPv6 forwarding is blocked rather than tunneled.

### Validate and load the rules

Check syntax without applying:

```bash
sudo nft -c -f /etc/vpn-gateway.nft
```

No output indicates that the syntax check succeeded.

Load the file:

```bash
sudo nft -f /etc/vpn-gateway.nft
```

Inspect the table:

```bash
sudo nft list table inet vpn_gateway
```

If the table already exists and you need to replace its chain contents, do not repeatedly load the same file without considering duplicate-chain/table errors. In the completed setup, the chain was cleared and the file reloaded:

```bash
sudo nft flush chain inet vpn_gateway forward_guard
sudo nft -f /etc/vpn-gateway.nft
```

Use that sequence only when the table and chain already exist.

---

## 10. Make the forwarding firewall persistent

The completed setup used a dedicated systemd oneshot service. It deliberately did **not** enable the generic `nftables.service`.

### Why not enable the generic nftables service?

The machine's `/etc/nftables.conf` contained a `flush ruleset` instruction. Enabling that service could flush rules maintained by NetworkManager, Docker, and libvirt. That is not appropriate for this machine's shared firewall environment.

Instead, the custom VPN gateway rules were kept in their own file and loaded by a dedicated service.

### Firewall file

Path:

```text
/etc/vpn-gateway.nft
```

Contents: the nftables table and chain shown in the previous section.

### systemd service

Path:

```text
/etc/systemd/system/vpn-gateway-firewall.service
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

`Before=NetworkManager.service` asks systemd to start this firewall service before NetworkManager. `Type=oneshot` is appropriate because the command loads rules and exits. `RemainAfterExit=yes` allows systemd to report the service as active after the command has completed.

Enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now vpn-gateway-firewall.service
```

Check it:

```bash
systemctl status vpn-gateway-firewall.service
systemctl is-enabled vpn-gateway-firewall.service
sudo nft list table inet vpn_gateway
```

An `active (exited)` status is normal for a successful oneshot service.

Do not add a naive `ExecReload` that simply runs the same file again: reloading an already-existing table may fail or create duplicate rules depending on the file structure.

---

## 11. OpenVPN / Proton VPN connection

The completed machine used Proton VPN's OpenVPN configuration through NetworkManager. The connection profile was:

```text
protonvpn-openvpn-connection
```

The tunnel interface was:

```text
tun0
```

The setup used NetworkManager's OpenVPN plugins:

```text
network-manager-openvpn
network-manager-openvpn-gnome
```

The Proton configuration was imported into NetworkManager and connected there. The exact server/configuration file is account- and server-dependent; use a current configuration downloaded from Proton rather than copying another user's endpoint or credentials.

Useful checks:

```bash
nmcli connection show
nmcli connection show --active
nmcli device status
ip addr show tun0
ip route
```

A connected VPN should create the tunnel interface and install the VPN routes and DNS settings.

---

## 12. Verification checklist

Run these checks after setup and again after reboot.

### A. Check the interfaces

```bash
ip -br addr
```

Expected roles:

- `wlo1`: upstream Wi-Fi address.
- `eno1`: `10.42.0.1/24`.
- `tun0`: Proton VPN tunnel address.

### B. Check the VPN connection and routes

```bash
nmcli connection show --active
ip route
```

Confirm that the VPN is connected and that the VPN default route is preferred.

### C. Check forwarding

```bash
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.ipv4.ip_forward = 1
```

### D. Check the firewall

```bash
sudo nft list table inet vpn_gateway
```

Confirm that the forwarding guard exists and contains the intended rules.

### E. Check the client network settings

On a phone or other client connected to the D-Link Wi-Fi, confirm:

- IP address is in `10.42.0.0/24`.
- Gateway is `10.42.0.1`.
- DNS is `10.42.0.1`.

### F. Check the public IP

Check the public IP from Ubuntu and from a downstream client. Both should show the VPN exit IP while the VPN is connected.

### G. Test the kill switch

1. Connect a client to the D-Link Wi-Fi.
2. Confirm that it can access the internet.
3. Disconnect the Proton VPN connection on Ubuntu.
4. Confirm that the client loses internet access.
5. Confirm that local access to Ubuntu/D-Link remains available where expected.
6. Reconnect Proton VPN and confirm that internet access returns.

The completed setup passed this test.

### H. Reboot test

After reboot:

```bash
systemctl status vpn-gateway-firewall.service
sudo nft list table inet vpn_gateway
nmcli connection show --active
ip route
```

Then repeat the client connectivity and VPN-disconnect tests. The completed setup was reboot-tested successfully.

---

## 13. Troubleshooting commands

These commands inspect the system; they do not automatically change configuration.

| What to inspect | Command |
|---|---|
| Interfaces and IPs | `ip -br addr` |
| Routes | `ip route` |
| NetworkManager devices | `nmcli device status` |
| Active connections | `nmcli connection show --active` |
| Ethernet profile | `nmcli connection show "netplan-eno1"` |
| VPN DNS | `resolvectl status` |
| DNS listeners | `ss -lntup \| grep ':53'` |
| IPv4 forwarding | `sysctl net.ipv4.ip_forward` |
| nftables rules | `sudo nft list ruleset` |
| VPN gateway table | `sudo nft list table inet vpn_gateway` |
| Firewall service | `systemctl status vpn-gateway-firewall.service` |
| NetworkManager logs | `journalctl -u NetworkManager -b` |
| Firewall service logs | `journalctl -u vpn-gateway-firewall.service -b` |

### Common failure patterns

**Client receives no IP address**
- Check that the Ethernet link is connected.
- Check that `netplan-eno1` is active and configured for shared mode.
- Ensure the D-Link DHCP server is disabled.
- Confirm that the cable is connected to a D-Link LAN port.

**Client gets an IP but no internet**
- Check that Proton VPN is connected.
- Check `ip route`.
- Check `net.ipv4.ip_forward`.
- Check the nftables forwarding guard and NetworkManager NAT rules.

**Internet works only when VPN is disconnected**
- Check the VPN default route and tunnel state.
- Check that the VPN connection is active and `tun0` exists.
- Check VPN DNS and routing.

**DNS fails but IP connectivity works**
- Check that the client DNS server is `10.42.0.1`.
- Check `resolvectl status`.
- Check dnsmasq and systemd-resolved listeners.
- Check that the VPN DNS server is reachable through `tun0`.

**D-Link administration page is unreachable**
- Confirm the D-Link address is `10.42.0.2/24`.
- Confirm Ubuntu Ethernet is `10.42.0.1/24`.
- Confirm that the D-Link is connected LAN-to-LAN.
- Ensure no other device uses `10.42.0.2`.

---

## 14. Proton VPN Free plan and multiple downstream devices

In this topology, Ubuntu establishes the Proton VPN connection. Downstream devices do not each establish their own Proton VPN session; they send traffic through Ubuntu's existing tunnel.

Therefore, multiple devices using the D-Link access point share the laptop's single VPN connection. They are not separate Proton VPN app connections. If another device independently connects to Proton VPN at the same time, that is an additional VPN connection and may exceed a one-connection plan limit.

---

## 15. Can the same idea be implemented on Windows?

Yes. A Windows PC can act as a VPN gateway, commonly using Internet Connection Sharing (ICS):

```text
Internet router -> Windows PC (Wi-Fi + VPN)
                -> Ethernet -> D-Link AP -> Wi-Fi clients
```

At a high level:

1. Connect Windows to the upstream network over Wi-Fi.
2. Connect the VPN on Windows.
3. Connect the PC's Ethernet port to a LAN port on the D-Link.
4. Configure the D-Link as an access point: DHCP off, LAN-to-LAN connection.
5. Enable Windows Internet Connection Sharing on the correct VPN-facing connection and share it to Ethernet, if the VPN client and adapter arrangement support it.
6. Verify client addressing, DNS, and public IP.

### Important Windows caveat

ICS provides sharing/NAT, but enabling ICS alone should not be treated as proof of a VPN kill switch. You must test what happens when the VPN disconnects. If client traffic can fall back to the ordinary Wi-Fi interface, add appropriate Windows Firewall rules or another supported kill-switch mechanism.

Windows VPN clients differ in whether they permit ICS, which adapter should be shared, and how they handle DNS and IPv6. Test the actual configuration rather than assuming that selecting an adapter guarantees all client traffic remains in the tunnel.

---

## 16. Final configuration summary

| Item | Completed value |
|---|---|
| Ubuntu upstream | Wi-Fi `wlo1` |
| Ubuntu downstream | Ethernet `eno1` |
| VPN interface | `tun0` |
| Downstream Ubuntu IP | `10.42.0.1/24` |
| D-Link management IP | `10.42.0.2/24` |
| D-Link mode | Access point / Layer-2 bridge |
| D-Link DHCP | Disabled |
| Downstream DHCP/DNS | NetworkManager shared mode |
| DHCP pool | `10.42.0.10`–`10.42.0.254` |
| Client gateway/DNS | `10.42.0.1` |
| IPv4 forwarding | Enabled |
| NAT | NetworkManager shared-mode masquerading |
| Forwarding kill switch | Custom nftables table `inet vpn_gateway` |
| Firewall persistence | `vpn-gateway-firewall.service` |
| Generic nftables service | Left disabled to avoid flushing other managed rules |

The final configuration was tested for normal client internet access, VPN-disconnect blocking, VPN reconnection, and persistence after reboot.
