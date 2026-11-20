# Ubuntu Proton VPN Gateway

A practical, tested guide to using an Ubuntu laptop as a VPN gateway for devices connected through a separate Wi-Fi access point.

The laptop connects to the internet over Wi-Fi, establishes a Proton VPN tunnel, and shares that tunnel through Ethernet. A D-Link DSL-124 R1 is used as a Layer-2 access point.

## Network overview

```text
Internet router
      |
   Wi-Fi
      |
 Ubuntu laptop
  Wi-Fi + Proton VPN
      |
   Ethernet
      |
 Wi-Fi access point
      |
 Phones, tablets, and other clients
```

Downstream clients use the Ubuntu laptop as their gateway. NetworkManager provides connection sharing, DHCP/DNS service, and NAT. A dedicated nftables forwarding rule set blocks downstream traffic from falling back to the ordinary upstream interface if the VPN tunnel disconnects.

## Documentation

- [Comprehensive tutorial](docs/comprehensive-tutorial.md) — architecture, subnetting, access-point configuration, NetworkManager sharing, DHCP, DNS, routing, NAT, nftables, systemd persistence, verification, and troubleshooting.
- [Quick setup guide](docs/quick-setup.md) — concise configuration steps and verification commands.

## Main components

- Ubuntu Linux
- Proton VPN using OpenVPN
- NetworkManager connection sharing
- dnsmasq (managed by NetworkManager)
- nftables
- systemd
- A separate Wi-Fi access point configured in bridge/access-point mode

## Important notes

- The guides document one tested setup. Interface names, VPN profile names, server endpoints, and router settings may differ on other systems.
- The downstream forwarding kill switch protects client traffic forwarded through the Ubuntu laptop. It does **not** by itself provide a kill switch for applications running directly on Ubuntu.
- IPv6 forwarding from downstream clients is blocked by the documented rules; this is not an IPv6-through-VPN configuration.
- Review the guides and adapt the configuration to your own network before applying commands. Avoid blindly flushing the system firewall ruleset, as other services may manage their own firewall rules.
- The D-Link is used as an access point/bridge, not as a second router.

## Tested configuration

The documented setup was verified for downstream internet access through the VPN, loss of downstream internet access when the VPN disconnected, recovery after reconnecting, and persistence after reboot.
