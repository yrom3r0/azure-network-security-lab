# Implementation Notes

This document captures the build logic and validation approach used in the Azure Network Security & Segmentation Lab.

## 1. Resource Group and Region

- Resource Group: `RG-CloudSecurity-Lab`
- Region: East US

## 2. Primary VNet

### `VNET-CloudSecurity-Lab`

Address space:

```text
10.0.0.0/16
```

Subnets:

```text
Subnet-Servers  10.0.10.0/24
Subnet-Clients  10.0.20.0/24
```

Primary workloads:

```text
VM-Server  10.0.10.4
VM-Client  10.0.20.4
```

## 3. Management VNet

### `VNET-Management`

Address space:

```text
10.1.0.0/16
```

Subnet:

```text
Subnet-Management  10.1.10.0/24
```

Management workload:

```text
VM-Management  10.1.10.4
```

## 4. NSG Design

The server subnet is protected using subnet-level Network Security Group controls.

The intended policy is:

- Client subnet can access the server application on TCP/80.
- Client subnet cannot use SSH against the server.
- Management subnet can use SSH for administrative access.
- Management subnet can use HTTP when application validation is required.

The design separates application traffic from privileged administrative traffic.

## 5. HTTP Test Service

A temporary Python HTTP service was used on `VM-Server` for connectivity testing.

Example startup sequence:

```bash
sudo mkdir -p /tmp/security-lab
sudo printf "============================\n     AZURE SECURITY LAB\n============================\n\nServer: VM-Server (10.0.10.4)\nService: HTTP (TCP/80)\nStatus: Reachable\n" | sudo tee /tmp/security-lab/index.html
cd /tmp/security-lab
sudo nohup python3 -m http.server 80 --bind 0.0.0.0 >/tmp/http-server.log 2>&1 &
```

Verify TCP/80 listener:

```bash
sudo ss -lntp | grep ':80'
```

## 6. Connectivity Validation

### HTTP test

```bash
curl -v --connect-timeout 5 http://10.0.10.4
```

Expected successful result:

```text
HTTP/1.0 200 OK
```

### TCP/22 restriction test

```bash
timeout 5 bash -c 'echo > /dev/tcp/10.0.10.4/22' && echo "TCP/22: ALLOWED - TEST FAILED" || echo "TCP/22: BLOCKED - SECURITY TEST PASSED"
```

## 7. Network Watcher Validation

The following Azure Network Watcher tools were used:

- IP Flow Verify
- Effective Security Rules
- Connection Troubleshoot
- Next Hop
- Network Topology

### IP Flow Verify logic

To validate the server-side NSG decision, the server VM was tested using inbound flows.

Example allowed flow:

```text
Direction: Inbound
Local IP: 10.0.10.4
Local Port: 80
Remote IP: 10.0.20.4
Protocol: TCP
```

Example restricted flow:

```text
Direction: Inbound
Local IP: 10.0.10.4
Local Port: 22
Remote IP: 10.0.20.4
Protocol: TCP
```

## 8. Route Table / UDR Test

A route table named `RT-Clients` was associated with `Subnet-Clients`.

A temporary route was used to demonstrate route precedence and an intentional blackhole condition:

```text
Name: Block-Server-Subnet
Destination: 10.0.10.0/24
Next Hop: None
```

This test demonstrated that a security rule can allow traffic while routing independently prevents delivery.

The blackhole route was removed after validation.

## 9. VNet Peering

`VNET-Management` and `VNET-CloudSecurity-Lab` were connected using VNet Peering.

Expected Next Hop result from `VM-Management` toward `VM-Server`:

```text
Next Hop Type: VirtualNetworkPeering
```

Cross-VNet application connectivity was validated from:

```text
VM-Management 10.1.10.4
```

to:

```text
VM-Server 10.0.10.4:80
```

## 10. Troubleshooting Case

During testing, the management VM reached the server but received:

```text
Connection refused
```

Network diagnostics showed that the route and peering path were functioning.

The destination was then checked for an active TCP/80 listener:

```bash
sudo ss -lntp | grep ':80'
```

No listener was present because the temporary HTTP service had stopped after the VM restart.

After restarting the service, the same connectivity test succeeded.

### Troubleshooting takeaway

Do not treat all connectivity failures as firewall problems.

A useful isolation sequence is:

```text
NSG / security policy
        ↓
Routing
        ↓
VNet Peering
        ↓
Destination host
        ↓
Application service
```

This avoids weakening security controls while troubleshooting an unrelated application issue.

## 11. Portfolio Evidence Checklist

Recommended evidence files:

```text
architecture/azure-network-security-architecture.png
screenshots/01-vnet-subnets.png
screenshots/02-nsg-server-rules.png
screenshots/03-client-http-allowed.png
screenshots/04-client-ssh-blocked.png
screenshots/05-ip-flow-verification.png
screenshots/06-routing-udr.png
screenshots/07-vnet-peering-validation.png
```

Before publishing screenshots, remove or hide subscription IDs, keys, credentials, tokens, and any unrelated account information.
