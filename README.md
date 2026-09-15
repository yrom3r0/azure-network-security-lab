# Azure Network Security & Segmentation Lab

> A practical Azure network-security case study focused on segmentation, least-privilege access, routing behavior, private connectivity, and troubleshooting.

## Business Scenario

A growing company is moving an internal application and supporting workloads to Microsoft Azure. The company needs to separate standard user traffic, server workloads, and administrative access without exposing the environment unnecessarily to the public Internet.

The security team was asked to design a network where:

- Users can reach only the application services they need.
- Standard client systems cannot use administrative protocols against servers.
- IT and Security administrators use a dedicated management network.
- Server administration is separated from normal user traffic.
- Communication between Azure networks remains private.
- Routing and security controls can be validated with Azure-native troubleshooting tools.

This lab implements that scenario using Azure Virtual Networks, subnets, Network Security Groups, User Defined Routes, VNet Peering, and Network Watcher.

---

## Architecture

The environment uses two Azure VNets: one for application workloads and one for management access.

![Azure Network Security Architecture](architecture/azure-network-security-architecture.png)

### Network Design

| Network | CIDR | Purpose |
|---|---|---|
| `VNET-CloudSecurity-Lab` | `10.0.0.0/16` | Main workload network |
| `Subnet-Servers` | `10.0.10.0/24` | Server/application workloads |
| `Subnet-Clients` | `10.0.20.0/24` | Standard client workloads |
| `VNET-Management` | `10.1.0.0/16` | Dedicated administration network |
| `Subnet-Management` | `10.1.10.0/24` | IT/Security management workloads |

### Virtual Machines

| VM | Private IP | Role |
|---|---|---|
| `VM-Server` | `10.0.10.4` | Linux application/web server |
| `VM-Client` | `10.0.20.4` | Standard user/client workload |
| `VM-Management` | `10.1.10.4` | Administrative workstation |

---

## Security Model

The design follows a least-privilege model: application access and administrative access are treated as separate traffic classes.

### Client → Server

| Service | Policy | Purpose |
|---|---|---|
| HTTP `TCP/80` | **Allow** | Required application access |
| SSH `TCP/22` | **Block** | Prevent client-side administrative access |
| Other unnecessary traffic | **Restrict** | Reduce lateral movement opportunities |

### Management → Server

| Service | Policy | Purpose |
|---|---|---|
| SSH `TCP/22` | **Allow** | Server administration from the management network |
| HTTP `TCP/80` | **Allow when required** | Application validation and troubleshooting |

Administrative traffic originates from the dedicated management network instead of the standard client subnet.

---

## Network Security Groups

NSGs were applied at the subnet level to enforce communication between network segments.

`NSG-Servers` controls traffic reaching server workloads based on source network, protocol, and destination port.

Example policy:

| Source | Destination | Service | Action |
|---|---|---|---|
| `10.0.20.0/24` Client subnet | Server subnet | TCP/80 | Allow |
| `10.0.20.0/24` Client subnet | Server subnet | TCP/22 | Deny |
| `10.1.10.0/24` Management subnet | Server subnet | TCP/22 | Allow |
| `10.1.10.0/24` Management subnet | Server subnet | TCP/80 | Allow when required |

This separates normal application access from privileged administrative access.

### Evidence

![NSG server rules](screenshots/02-nsg-server-rules.png)

---

## Routing & UDR Testing

Azure system routes were reviewed first to understand the default path between subnets.

A custom route table, `RT-Clients`, was then associated with `Subnet-Clients`.

To demonstrate route precedence and failure isolation, a temporary UDR was created:

```text
Destination: 10.0.10.0/24
Next hop:    None
```

This intentionally created a blackhole route toward the server subnet.

The test demonstrated an important troubleshooting principle:

> An NSG can permit a flow while routing independently prevents the packet from reaching the destination.

The temporary blackhole route was used only for testing and removed afterward.

### Evidence

![UDR routing test](screenshots/06-routing-udr.png)

---

## VNet Peering

`VNET-Management` was connected to `VNET-CloudSecurity-Lab` using Azure VNet Peering.

This provides private connectivity between the management environment and server resources over the Azure backbone without requiring public IP communication between the workloads.

Azure Network Watcher confirmed the routing decision:

```text
Next Hop Type: VirtualNetworkPeering
```

Cross-VNet HTTP connectivity was then validated from `VM-Management` to `VM-Server`.

### Evidence

![VNet peering validation](screenshots/07-vnet-peering-validation.png)

---

## Validation

The design was tested using both operating-system-level commands and Azure-native diagnostics.

### Client security validation

From `VM-Client`, HTTP access to `VM-Server` succeeded on TCP/80 while TCP/22 was blocked.

![Client security validation](screenshots/03-client-security-validation.png)

### Azure Network Watcher

The following tools were used during validation and troubleshooting:

- **IP Flow Verify** — validated whether NSG rules allowed or denied a specific flow.
- **Effective Security Rules** — reviewed the security policy effectively applied to the workload.
- **Next Hop** — validated Azure's routing decision.
- **Connection Troubleshoot** — tested end-to-end connectivity and helped isolate failure points.
- **Network Topology** — reviewed resource relationships and network layout.

#### HTTP allowed

![IP Flow Verify - HTTP allowed](screenshots/04-ip-flow-http-allowed.png)

#### SSH denied

![IP Flow Verify - SSH denied](screenshots/05-ip-flow-ssh-denied.png)

---

## Troubleshooting Case Study

One of the most useful tests occurred during cross-VNet validation.

`VM-Management` attempted to connect to the web service on `VM-Server` and received:

```text
connect to 10.0.10.4 port 80 failed: Connection refused
```

Instead of immediately changing the NSG, the network path was validated first.

The investigation confirmed:

- VNet Peering was established.
- Azure selected `VirtualNetworkPeering` as the next hop.
- The server-side inbound security path permitted TCP/80.
- The route to the destination was valid.

Because the connection was **refused** rather than **timed out**, troubleshooting moved to the application layer.

On `VM-Server`:

```bash
sudo ss -lntp | grep ':80'
```

returned no listener.

The temporary Python HTTP service had stopped after the VM was restarted. After restarting the service and verifying that TCP/80 was listening, the connection was tested again.

Result:

```text
HTTP/1.0 200 OK
```

### Root Cause

The Azure network path was functioning correctly. The failure occurred because the application service was not listening on TCP/80.

### Key Lesson

A connectivity failure should be isolated systematically across layers:

```text
Security policy → Routing → Peering → Host → Application
```

Changing firewall or NSG rules before verifying the application could have introduced unnecessary access without resolving the actual issue.

---

## Security Design Decisions

- Segmented client, server, and management workloads.
- Kept workload communication on private Azure networking.
- Restricted administrative protocols from the client network.
- Used a dedicated management network for privileged access.
- Applied NSGs at the subnet level.
- Tested routing behavior independently from NSG behavior.
- Used explicit connectivity tests instead of assuming configuration was correct.
- Used Azure Network Watcher to validate and troubleshoot network behavior.

---

## Project Evidence

| Evidence | What it demonstrates |
|---|---|
| [Architecture diagram](architecture/azure-network-security-architecture.png) | Overall Azure security design |
| [VNet and subnet configuration](screenshots/01-vnet-subnets.png) | Network segmentation |
| [NSG rules](screenshots/02-nsg-server-rules.png) | Least-privilege traffic control |
| [Client security validation](screenshots/03-client-security-validation.png) | HTTP allowed and SSH blocked |
| [IP Flow Verify - HTTP](screenshots/04-ip-flow-http-allowed.png) | Azure-native allow validation |
| [IP Flow Verify - SSH](screenshots/05-ip-flow-ssh-denied.png) | Azure-native deny validation |
| [UDR test](screenshots/06-routing-udr.png) | Routing behavior and blackhole testing |
| [VNet Peering validation](screenshots/07-vnet-peering-validation.png) | Private cross-VNet connectivity |

---

## Skills Demonstrated

`Azure Networking` · `Network Security Groups` · `Network Segmentation` · `TCP/IP` · `User Defined Routes` · `VNet Peering` · `Network Watcher` · `Linux Networking` · `Least Privilege` · `Cloud Security` · `Network Troubleshooting`

---

## Documentation

Detailed implementation and validation notes are available in [`docs/implementation-notes.md`](docs/implementation-notes.md).
