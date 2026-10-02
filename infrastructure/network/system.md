# Current state

- UniFi gateway: Cloud Gateway Max.
- Home switches are USW Lite 8 PoE. Proxmox and the UPS management port connect
  to the same UPS-powered switch. The Proxmox port has native Servers VLAN `30`
  and allows tagged VLANs.
- Default network: VLAN `1`, subnet `192.168.1.0/24`, gateway `192.168.1.1`,
  DHCP pool `192.168.1.100`–`192.168.1.254`.
- UniFi UPS 2U: `192.168.1.34` on Default VLAN `1`. Its NUT server uses TCP
  `3493`, UPS ID `ups`, and username `ups`; the password is in the owner's
  password manager and on the Proxmox host.
- The UPS powers Proxmox, the shared switch, and the NAS. The NAS and UPS are
  on Default VLAN `1`. The Cloud Gateway Max is not UPS-powered.
- VLAN `30` is routed through the UniFi NordVPN NordLynx client. Its source
  selector is `192.168.30.0/24` and its Kill Switch is enabled. UniFi owns
  this VPN policy and its fail-closed behavior.
- K3s workload traffic is masqueraded through the K3s VM, so UniFi cannot
  select an individual Pod or application for a separate route.
- Clients on the Default and Servers VLANs use the UniFi Gateway for DNS and
  can reach Traefik at `192.168.30.103` on TCP 80. The UniFi wildcard Host
  record `*.home.arpa` resolves to that address.
- Public DNS for `notech.foo` is managed in Cloudflare. The UniFi Gateway
  manages DDNS for `vpn.no.tech.foo`.
