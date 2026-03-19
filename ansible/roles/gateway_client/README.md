# gateway_client Role

## Purpose
Configures a server to use another server as its default gateway for internet access. This role is designed for batch servers that need internet connectivity through a NAT gateway (web server).

## Requirements
- NetworkManager must be installed and running
- Target server must have a private network interface connected to the gateway
- Gateway server must be configured with gateway_nat role first

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| gateway_server_ip | 192.168.0.10 | IP address of the gateway server |
| client_private_interface | eth1 | Private network interface name |
| gateway_verify_connectivity | true | Test internet connectivity after configuration |
| gateway_test_host | 8.8.8.8 | Host to ping for connectivity test |

## Usage

Add to your playbook:

```yaml
- role: gateway_client
  tags: [gateway_client]
```

## What This Role Does

1. Checks current default gateway configuration
2. Gets NetworkManager connection name for the private interface
3. Configures the gateway IP in NetworkManager
4. Restarts the network connection to apply changes
5. Verifies gateway is reachable
6. Tests internet connectivity (optional)

## Dependencies

- gateway_nat role must be applied to the gateway server first

## Notes

- Configuration is persistent across reboots
- Uses NetworkManager for network management (Rocky Linux 9 default)
- Only runs when gateway is not already configured correctly (idempotent)
