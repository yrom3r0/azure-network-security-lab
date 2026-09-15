![Azure](https://img.shields.io/badge/Microsoft-Azure-0078D4?logo=microsoftazure&logoColor=white)
![Cloud Security](https://img.shields.io/badge/Cloud-Security-blue)
![Networking](https://img.shields.io/badge/Network-Security-success)
![Linux](https://img.shields.io/badge/Linux-Ubuntu-E95420?logo=ubuntu&logoColor=white)

# Azure Network Security & Segmentation Lab

> A hands-on Azure security project built around a simple question: **how do you let people reach the services they need without giving them access to everything else?**

## Why I built this

I wanted to build something closer to a real network-security problem than a basic “create a VNet and a VM” lab.

The scenario is a small organization moving internal workloads to Azure. Regular users need access to an application, but they should not be able to administer the servers behind it. At the same time, IT needs a separate and controlled path for management access.

So I built the environment around three ideas:

- **Users should get only the access they actually need.**
- **Administrative access should come from a separate management network.**
- **Every rule should be tested instead of assumed to work.**

The result is a segmented Azure environment using VNets, subnets, NSGs, custom routes, VNet Peering, Network Watcher, and Linux networking tools.

---

## Architecture

The design uses one VNet for normal workloads and a second VNet for management access.

<p align="center">
  <a href="architecture/azure-network-security-architecture.png">
    <img src="architecture/azure-network-security-architecture.png" width="1000" alt="Azure Network Security Architecture">
  </a>
</p>

<p align="center"><em>Click the diagram to open the full-size architecture.</em></p>

### Network layout

| Component | CIDR / IP | What it is used for |
|---|---|---|
| `VNET-CloudSecurity-Lab` | `10.0.0.0/16` | Main workload network |
| `Subnet-Servers` | `10.0.10.0/24` | Server and application workloads |
| `Subnet-Clients` | `10.0.20.0/24` | Standard user/client workloads |
| `VNET-Management` | `10.1.0.0/16` | Separate administrative network |
| `Subnet-Management` | `10.1.10.0/24` | IT/Security management workloads |
| `VM-Server` | `10.0.10.4` | Ubuntu web/application server |
| `VM-Client` | `10.0.20.4` | Standard client machine |
| `VM-Management` | `10.1.10.4` | Administrative workstation |

[![VNet and subnet configuration](screenshots/01-vnet-subnets.png)](screenshots/01-vnet-subnets.png)

---

## What should be allowed?

The idea was to keep the policy easy to understand.

A normal client should be able to open the application over HTTP, but it should **not** be able to SSH into the server.

The management network is different: it is the trusted administrative path, so it can reach the server over SSH and can also test the web service when needed.

### Client → Server

| Traffic | Result | Reason |
|---|---|---|
| HTTP `TCP/80` | **Allowed** | Users need the application |
| SSH `TCP/22` | **Blocked** | Standard clients should not administer servers |
| Unneeded traffic | **Restricted** | Reduce unnecessary lateral access |

### Management → Server

| Traffic | Result | Reason |
|---|---|---|
| SSH `TCP/22` | **Allowed** | Administrative access |
| HTTP `TCP/80` | **Allowed** | Testing and troubleshooting |
| Unneeded traffic | **Restricted** | Management should not mean unrestricted access |

---

## Enforcing the policy with NSGs

I applied Network Security Groups at the subnet level, with `NSG-Servers` protecting the server subnet.

The important part was not just creating allow and deny rules, but making the rules match the role of each network:

- Client subnet → application traffic only.
- Management subnet → approved administrative traffic.
- Everything else → restricted unless there is a reason to allow it.

[![NSG server rules](screenshots/02-nsg-server-rules.png)](screenshots/02-nsg-server-rules.png)

---

## Testing it like a real user

Before relying on Azure diagnostics, I tested the environment from the machines themselves.

From `VM-Client`:

- HTTP to `VM-Server` worked.
- SSH to `VM-Server` was blocked.

That gave me the behavior I wanted: the user can reach the service, but not the administrative interface behind it.

[![Client security validation](screenshots/03-client-security-validation.png)](screenshots/03-client-security-validation.png)

Then I checked the same flows with **Azure Network Watcher**.

### HTTP — allowed

IP Flow Verify confirmed that TCP/80 matched the intended allow rule.

[![IP Flow Verify HTTP allowed](screenshots/04-ip-flow-http-allowed.png)](screenshots/04-ip-flow-http-allowed.png)

### SSH — denied

The same test confirmed that SSH from the client network was denied.

[![IP Flow Verify SSH denied](screenshots/05-ip-flow-ssh-denied.png)](screenshots/05-ip-flow-ssh-denied.png)

---

## What happens when routing breaks?

I also wanted to test something that is easy to miss during troubleshooting: **an NSG can allow traffic and the connection can still fail because of routing.**

To prove that, I created a temporary route table called `RT-Clients` and added this User Defined Route:

```text
Name:        Block-Server-Subnet
Destination: 10.0.10.0/24
Next hop:    None
```

That route intentionally sent traffic to a black hole.

In other words, even if the security rule said “allow,” the packet still had no valid path to the server.

[![UDR routing test](screenshots/06-routing-udr.png)](screenshots/06-routing-udr.png)

I removed the route after the test so normal connectivity was restored.

---

## Adding a separate management network

Instead of placing administrators in the same network as regular users, I created `VNET-Management` and connected it to the workload VNet through **VNet Peering**.

This gave the management VM a private path to the server without exposing the workloads with public IPs.

Network Watcher confirmed the routing decision:

```text
Source:        10.1.10.4
Destination:   10.0.10.4
Next Hop Type: VirtualNetworkPeering
Route:         System Route
```

[![VNet peering validation](screenshots/07-vnet-peering-validation.png)](screenshots/07-vnet-peering-validation.png)

---

## The troubleshooting moment that made the lab worth it

The most useful part of the project came when something actually failed.

From `VM-Management`, I tried to reach the web service on `VM-Server` and got:

```text
connect to 10.0.10.4 port 80 failed: Connection refused
```

My first thought could have been “the firewall is blocking it,” but instead I checked the path one layer at a time.

I verified that:

- VNet Peering was working.
- Azure selected `VirtualNetworkPeering` as the next hop.
- The NSG allowed TCP/80.
- The route to the server was valid.

At that point the network looked healthy, so I checked the server itself:

```bash
sudo ss -lntp | grep ':80'
```

Nothing was listening on port 80.

The Python HTTP service had stopped after the VM was restarted.

I restarted the service, tested again, and got:

```text
HTTP/1.0 200 OK
```

The issue was not Azure networking at all — it was the application.

That was a good reminder that network troubleshooting is usually faster when you work through the layers instead of changing firewall rules until something starts working.

```text
Security policy → Routing → Peering → Host → Application
```

---

## What I used to validate the environment

During the lab I used:

- **IP Flow Verify** to see which NSG rule allowed or denied a connection.
- **Effective Security Rules** to check the policy actually applied to a workload.
- **Next Hop** to understand Azure's routing decision.
- **Connection Troubleshoot** to test connectivity end to end.
- **Network Topology** to review how the resources were connected.
- **curl, TCP tests, and Linux socket checks** to validate what was happening from the operating system itself.

---

## What I learned

The biggest takeaway was that a connection problem does not automatically mean “firewall issue.”

Security rules, routes, peering, the operating system, and the application can all fail independently. Testing each layer made it much easier to find the real cause without opening access unnecessarily.

A few things this lab reinforced for me:

- Least privilege is easier to manage when networks have clear roles.
- A management network should provide controlled access, not unlimited trust.
- `Connection refused` and `connection timed out` tell very different stories.
- Routing and security need to be tested separately.
- Azure-native tools are useful, but they are much stronger when combined with real traffic tests from the VM.

---

## What this project demonstrates

This lab brings together several skills in one small environment:

- Azure network design and segmentation
- Network Security Groups
- Least-privilege access
- TCP/IP troubleshooting
- User Defined Routes
- VNet Peering
- Azure Network Watcher
- Linux networking
- Application-layer troubleshooting
- Security validation and documentation

---

## Evidence

| Evidence | What it shows |
|---|---|
| [Architecture diagram](architecture/azure-network-security-architecture.png) | Overall design |
| [VNet and subnet configuration](screenshots/01-vnet-subnets.png) | Network segmentation |
| [NSG rules](screenshots/02-nsg-server-rules.png) | Traffic-control policy |
| [Client security validation](screenshots/03-client-security-validation.png) | HTTP allowed and SSH blocked |
| [IP Flow Verify — HTTP](screenshots/04-ip-flow-http-allowed.png) | Azure confirms the allow rule |
| [IP Flow Verify — SSH](screenshots/05-ip-flow-ssh-denied.png) | Azure confirms the deny rule |
| [UDR blackhole test](screenshots/06-routing-udr.png) | Routing failure scenario |
| [VNet Peering validation](screenshots/07-vnet-peering-validation.png) | Private cross-VNet routing |

---

## Next steps

A few ways I could extend the environment later:

- Azure Bastion for controlled administrative access
- Just-in-Time VM access
- Azure Firewall or an NVA for centralized inspection
- NSG flow logs and deeper traffic visibility
- Point-to-Site or Site-to-Site VPN connectivity

For this version, I intentionally kept the scope focused on **segmentation, access control, routing, private connectivity, and troubleshooting**.

---

## Technologies

`Microsoft Azure` · `Azure Virtual Networks` · `Network Security Groups` · `Network Segmentation` · `TCP/IP` · `User Defined Routes` · `VNet Peering` · `Azure Network Watcher` · `Ubuntu Linux` · `SSH` · `HTTP` · `Cloud Security` · `Network Troubleshooting`

---

## Documentation

Detailed implementation notes are available in [`docs/implementation-notes.md`](docs/implementation-notes.md).
