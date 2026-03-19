# gateway_nat Role

## Purpose
Configures a server (web server) to act as a NAT gateway, allowing other servers on the private network to access the internet through it.

## Requirements
- Server must have two network interfaces (public and private)
- Server must have iptables installed
- Proper network routing between interfaces

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| gateway_public_interface | eth0 | Public-facing network interface |
| gateway_private_interface | eth1 | Private network interface |
| gateway_nat_network | 192.168.0.0/24 | Private network CIDR to allow NAT |
| gateway_enable_ip_forward | true | Enable IP forwarding |
| gateway_install_iptables_services | true | Install iptables-services for persistence |
| gateway_enable_logging | false | Enable iptables logging for debugging |
| gateway_log_prefix | "GATEWAY-NAT: " | Log prefix for iptables logs |

## Usage

Add to your playbook:

```yaml
- role: gateway_nat
  tags: [gateway_nat]
```

## What This Role Does

1. Enables IP forwarding (runtime and persistent)
2. Installs iptables-services for rule persistence
3. Configures iptables NAT (MASQUERADE) rule
4. Configures iptables FORWARD rules:
   - Allow traffic from private to public interface
   - Allow established connections from public to private
5. Optionally enables logging for troubleshooting
6. Persists iptables rules across reboots
7. Displays configured rules for verification

## Dependencies

None

## Notes

- All iptables rules are idempotent (checks before adding)
- Rules are automatically saved to /etc/sysconfig/iptables
- IP forwarding is persisted in /etc/sysctl.conf
- Logging is disabled by default to reduce log volume

## Security Considerations

- Only allows outbound connections from private network
- Stateful firewall (only established connections allowed back)
- No inbound connections to private network from internet
- Consider enabling logging during initial setup, then disabling for production
