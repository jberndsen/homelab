# Current state

- UniFi gateway: Cloud Gateway Max.
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
