![Azure](https://img.shields.io/badge/Microsoft-Azure-0078D4?logo=microsoftazure&logoColor=white)
![Cloud Security](https://img.shields.io/badge/Cloud-Security-blue)
![Networking](https://img.shields.io/badge/Network-Security-success)
![Linux](https://img.shields.io/badge/Linux-Ubuntu-E95420?logo=ubuntu&logoColor=white)

# Azure Network Security & Segmentation Lab

> A practical Azure network-security case study focused on segmentation, least-privilege access, routing behavior, private connectivity, validation, and troubleshooting.

## Overview

A growing organization is migrating internal workloads to Microsoft Azure and needs a network architecture that separates standard users, server workloads, and administrative access.

The objective of this project was not simply to deploy Azure resources, but to translate security requirements into a working design, validate the controls, intentionally test failure scenarios, and troubleshoot connectivity across multiple layers.

The environment uses **Azure Virtual Networks, subnets, Network Security Groups (NSGs), User Defined Routes (UDRs), VNet Peering, Azure Network Watcher, and Linux networking tools**.

## Quick Navigation

- [Business Requirements](#business-requirements)
- [Architecture](#architecture)
- [Network Design](#network-design)
- [Security Controls](#security-controls)
- [Routing & UDR Testing](#routing--udr-testing)
- [VNet Peering](#vnet-peering)
- [Validation](#validation)
- [Troubleshooting Case Study](#troubleshooting-case-study)
- [Lessons Learned](#lessons-learned)
- [Results](#results)
- [Future Improvements](#future-improvements)
- [Technologies](#technologies)

---

## Business Requirements

The security team requested an Azure design that would:

- Separate client, server, and management workloads.
- Allow users to reach only the application services they need.
- Prevent standard client systems from using administrative protocols against servers.
- Restrict privileged server administration to the management network.
- Keep communication between Azure networks private.
- Reduce unnecessary lateral-movement paths.
- Validate security and routing behavior using Azure-native diagnostic tools.

The resulting design follows a **least-privilege network access model** rather than treating every internal subnet as equally trusted.

---

## Architecture

The environment uses two VNets: a primary workload VNet and a dedicated management VNet.

<p align="center">
  <a href="architecture/azure-network-security-architecture.png">
    <img src="architecture/azure-network-security-architecture.png" width="1000" alt="Azure Network Security Architecture">
  </a>
</p>

<p align="center"><em>Click the diagram to open the full-size architecture.</em></p>

## Network Design

| Component | CIDR / IP | Purpose |
|---|---|---|
| `VNET-CloudSecurity-Lab` | `10.0.0.0/16` | Main workload network |
| `Subnet-Servers` | `10.0.10.0/24` | Server/application workloads |
| `Subnet-Clients` | `10.0.20.0/24` | Standard client workloads |
| `VNET-Management` | `10.1.0.0/16` | Dedicated administration network |
| `Subnet-Management` | `10.1.10.0/24` | IT/Security management workloads |
| `VM-Server` | `10.0.10.4` | Linux application/web server |
| `VM-Client` | `10.0.20.4` | Standard client workload |
| `VM-Management` | `10.1.10.4` | Administrative workstation |

### Segmentation Evidence

[![VNet and subnet configuration](screenshots/01-vnet-subnets.png)](screenshots/01-vnet-subnets.png)

---

## Security Controls

### Traffic Policy

Application traffic and administrative traffic are handled as separate security classes.

#### Client → Server

| Service | Policy | Security Purpose |
|---|---|---|
| HTTP `TCP/80` | **Allow** | Required application access |
| SSH `TCP/22` | **Deny** | Prevent administrative access from standard clients |
| Other unnecessary traffic | **Deny / Restrict** | Reduce unnecessary east-west access |

#### Management → Server

| Service | Policy | Security Purpose |
|---|---|---|
| SSH `TCP/22` | **Allow** | Server administration from the management network |
| HTTP `TCP/80` | **Allow** | Application validation and troubleshooting |
| Other unnecessary traffic | **Deny / Restrict** | Keep management access controlled |

### Network Security Groups

NSGs were applied at the subnet level to enforce the policy between network segments.

`NSG-Servers` evaluates traffic based on source network, protocol, and destination port. This allows normal users to access the application while keeping privileged access on the dedicated management path.

[![NSG server rules](screenshots/02-nsg-server-rules.png)](screenshots/02-nsg-server-rules.png)

---

## Routing & UDR Testing

Azure system routes were reviewed first to understand the default routing behavior between subnets.

A custom route table, `RT-Clients`, was associated with `Subnet-Clients`.

To demonstrate route precedence and isolate routing failures, a temporary User Defined Route was configured:

```text
Name:        Block-Server-Subnet
Destination: 10.0.10.0/24
Next hop:    None
```

This intentionally created a **blackhole route** toward the server subnet.

The test demonstrated an important troubleshooting principle:

> A security policy can allow a flow while routing independently prevents the packet from reaching the destination.

The temporary blackhole route was used only for testing and removed afterward.

[![UDR routing test](screenshots/06-routing-udr.png)](screenshots/06-routing-udr.png)

---

## VNet Peering

`VNET-Management` was connected to `VNET-CloudSecurity-Lab` using **Azure VNet Peering**.

This provides private connectivity between the administrative environment and server resources without requiring public IP communication between the workloads.

Azure Network Watcher confirmed the routing decision:

```text
Source:        10.1.10.4
Destination:   10.0.10.4
Next Hop Type: VirtualNetworkPeering
Route:         System Route
```

[![VNet peering validation](screenshots/07-vnet-peering-validation.png)](screenshots/07-vnet-peering-validation.png)

---

## Validation

The environment was validated using both real traffic tests and Azure-native diagnostics.

### 1. Client Application Access

From `VM-Client`, HTTP access to `VM-Server` succeeded on TCP/80, while TCP/22 was blocked.

This confirms that normal users can reach the required application service without receiving administrative access to the server.

[![Client security validation](screenshots/03-client-security-validation.png)](screenshots/03-client-security-validation.png)

### 2. IP Flow Verify — HTTP Allowed

Azure Network Watcher confirmed that TCP/80 matched the intended allow rule.

[![IP Flow Verify HTTP allowed](screenshots/04-ip-flow-http-allowed.png)](screenshots/04-ip-flow-http-allowed.png)

### 3. IP Flow Verify — SSH Denied

The same diagnostic workflow confirmed that TCP/22 from the client network was denied.

[![IP Flow Verify SSH denied](screenshots/05-ip-flow-ssh-denied.png)](screenshots/05-ip-flow-ssh-denied.png)

### Azure Network Watcher Tools Used

- **IP Flow Verify** — identified whether a specific NSG rule allowed or denied a flow.
- **Effective Security Rules** — reviewed the actual security policy applied to a workload.
- **Next Hop** — validated Azure's routing decision.
- **Connection Troubleshoot** — tested end-to-end connectivity and helped isolate failure points.
- **Network Topology** — reviewed relationships between network resources.

---

## Troubleshooting Case Study

### Problem

During cross-VNet validation, `VM-Management` attempted to reach the HTTP service on `VM-Server` and received:

```text
connect to 10.0.10.4 port 80 failed: Connection refused
```

### Evidence & Investigation

Instead of immediately changing firewall or NSG rules, the path was validated layer by layer.

The investigation confirmed:

- **VNet Peering:** established and working.
- **Routing:** Azure selected `VirtualNetworkPeering` as the next hop.
- **Server inbound NSG:** TCP/80 was permitted.
- **Destination route:** valid.

Because the connection was **refused** rather than **timed out**, the network path appeared reachable and the investigation moved to the destination service.

### Root Cause

On `VM-Server`, the listener was checked with:

```bash
sudo ss -lntp | grep ':80'
```

No process was listening on TCP/80.

The temporary Python HTTP service had stopped after the VM was restarted.

### Resolution

The HTTP service was restarted and the listener was verified on `0.0.0.0:80`.

The test from `VM-Management` was repeated successfully:

```text
HTTP/1.0 200 OK
```

### Troubleshooting Flow

```text
Security Policy → Routing → VNet Peering → Host → Application
```

This prevented an unnecessary NSG change that would not have solved the actual problem and could have introduced additional exposure.

---

## Lessons Learned

This project reinforced several practical cloud-network troubleshooting principles:

- **Do not assume every connectivity failure is a firewall problem.**
- Validate security policy and routing independently.
- A successful route does not guarantee that an application is listening.
- `Connection refused` and `connection timed out` point toward different failure domains.
- Azure Network Watcher is most useful when its results are correlated with real application and operating-system tests.
- Management networks should provide controlled administrative access, not unrestricted trust.
- Temporary failure scenarios such as blackhole routes are useful for learning how Azure behaves when different layers fail.

---

## Results

The completed environment demonstrates:

- Segmented client, server, and management networks.
- Least-privilege client-to-server access.
- Dedicated administrative access from the management network.
- Private connectivity between VNets using Azure VNet Peering.
- Subnet-level NSG enforcement.
- Custom routing behavior using UDRs.
- Azure-native validation with Network Watcher.
- Practical TCP/IP and application-layer troubleshooting.

### Project Evidence

| Evidence | What it demonstrates |
|---|---|
| [Architecture diagram](architecture/azure-network-security-architecture.png) | Overall Azure security design |
| [VNet and subnet configuration](screenshots/01-vnet-subnets.png) | Network segmentation |
| [NSG rules](screenshots/02-nsg-server-rules.png) | Least-privilege traffic policy |
| [Client security validation](screenshots/03-client-security-validation.png) | HTTP allowed and SSH blocked |
| [IP Flow Verify — HTTP](screenshots/04-ip-flow-http-allowed.png) | Azure-native allow validation |
| [IP Flow Verify — SSH](screenshots/05-ip-flow-ssh-denied.png) | Azure-native deny validation |
| [UDR blackhole test](screenshots/06-routing-udr.png) | Custom routing behavior |
| [VNet Peering validation](screenshots/07-vnet-peering-validation.png) | Private cross-VNet routing |

---

## Future Improvements

Potential extensions to the same network-security design include:

- **Azure Bastion** for controlled administrative access without direct SSH exposure.
- **Just-in-Time VM access** for temporary administrative access.
- **Azure Firewall** or an NVA for centralized traffic inspection and policy enforcement.
- **NSG flow logging / traffic analytics** for deeper network visibility.
- **Point-to-Site or Site-to-Site VPN** for secure hybrid administrative connectivity.

---

## Technologies

`Microsoft Azure` · `Azure Virtual Networks` · `Network Security Groups` · `Network Segmentation` · `TCP/IP` · `User Defined Routes` · `VNet Peering` · `Azure Network Watcher` · `Ubuntu Linux` · `SSH` · `HTTP` · `Cloud Security` · `Network Troubleshooting`

---

## Documentation

Detailed implementation and validation notes are available in [`docs/implementation-notes.md`](docs/implementation-notes.md).
