# Networking Fundamentals and Protocols

Language: English | [中文](../后端知识库/06-网络基础与协议.md)

---

## Table of Contents and Learning Path

### Part I: Fundamentals
1. [Layering and Data Delivery](#1-layering-and-data-delivery)
2. [IP, Link, Routing, and NAT](#2-ip-link-routing-and-nat)

### Part II: Transport
3. [TCP In Depth](#3-tcp-in-depth)
4. [UDP and Transport Selection](#4-udp-and-transport-selection)

### Part III: Application Protocols
5. [HTTP](#5-http)
6. [HTTPS and TLS](#6-https-and-tls)
7. [Real-Time Communication and Video Transport](#7-real-time-communication)
8. [DNS and Domain Resolution](#8-dns-and-domain-resolution)

### Part IV: Traffic Entry and Data-Center Networks
9. [Load Balancing, Proxies, CDN, and Global Traffic](#9-load-balancing-proxies-cdn-and-global-traffic)
10. [IDC Data-Center Networking: From Large L2 to Spine-Leaf and SDN](#10-idc-data-center-networking-from-large-l2-to-spine-leaf-and-sdn)

### Part V: Hosts, Containers, and Modern Protocols
11. [Linux, Container Networking, and Security Boundaries](#11-linux-container-networking-and-security-boundaries)
12. [QUIC and HTTP/3](#12-quic-and-http3)
13. [TCP BBR Congestion Control](#13-tcp-bbr-congestion-control)

### Part VI: Operations and Review
14. [Network Troubleshooting](#14-network-troubleshooting)
15. [Game Networking Index](#15-game-networking-index)
16. [Interview Self-Check](#16-interview-self-check)

---

## Common Terms and Acronyms

| Term / Acronym | Full Name | Meaning |
|---------|-----------|---------|
| **OSI** | Open Systems Interconnection | Seven-layer network model |
| **TCP** | Transmission Control Protocol | Reliable connection-oriented transport |
| **UDP** | User Datagram Protocol | Connectionless low-overhead transport |
| **IP** | Internet Protocol | Network-layer addressing and routing |
| **HTTP** | HyperText Transfer Protocol | Web application protocol |
| **HTTPS** | HTTP Secure / HTTP over TLS | HTTP protected by TLS |
| **SSL** | Secure Sockets Layer | Predecessor of TLS; obsolete |
| **TLS** | Transport Layer Security | Modern transport security protocol |
| **RTT** | Round-Trip Time | Time for a packet and its response to make a round trip |
| **DNS** | Domain Name System | Distributed naming system for domains |
| **CDN** | Content Delivery Network | Distributed content delivery and caching |
| **GSLB** | Global Server Load Balancing | Global traffic steering across regions/sites |
| **Anycast** | Anycast | Multiple sites advertise the same IP; routing selects a nearby/best site |
| **BGP** | Border Gateway Protocol | Inter-domain routing protocol of the Internet |
| **TTL** | Time To Live | Cache lifetime or packet lifetime/hop limit |
| **CA** | Certificate Authority | Entity that issues certificates |
| **mTLS** | mutual TLS | Both client and server authenticate with certificates |
| **AEAD** | Authenticated Encryption with Associated Data | Encryption mode with integrity authentication |
| **PSK** | Pre-Shared Key | Keying material used for TLS resumption |
| **QUIC** | Quick UDP Internet Connections | Modern UDP-based transport used by HTTP/3 |
| **BBR** | Bottleneck Bandwidth and Round-trip propagation time | Congestion control algorithm |
| **RTMP** | Real-Time Messaging Protocol | Legacy TCP-based live ingest protocol |
| **HLS** | HTTP Live Streaming | HTTP segment-based live/VOD delivery |
| **SRT** | Secure Reliable Transport | UDP-based low-latency media transport |
| **FEC** | Forward Error Correction | Recover loss using redundant data |
| **ARQ** | Automatic Repeat reQuest | Receiver-driven retransmission |
| **IDC** | Internet Data Center | Data-center facility for compute/network/storage |
| **ToR** | Top of Rack | Rack-level access switch |
| **ECMP** | Equal-Cost Multi-Path | Use multiple equal routes in parallel |
| **M-LAG** | Multi-Chassis Link Aggregation | Link aggregation across multiple switches |
| **LACP** | Link Aggregation Control Protocol | Negotiates aggregated physical links |
| **STP** | Spanning Tree Protocol | L2 loop-prevention protocol |
| **VLAN** | Virtual LAN | L2 broadcast-domain isolation |
| **SDN** | Software-Defined Networking | Separation of control and forwarding planes |
| **VXLAN** | Virtual eXtensible LAN | Overlay tunnel encapsulating L2 frames in UDP/IP |
| **VTEP** | VXLAN Tunnel Endpoint | VXLAN encapsulation/decapsulation endpoint |
| **EVPN** | Ethernet VPN | BGP-based overlay reachability control plane |
| **TRILL** | Transparent Interconnection of Lots of Links | L2 multipathing with IS-IS control plane |
| **RBridge** | Routing Bridge | TRILL device that forwards L2 frames with routing-like behavior |
| **IS-IS** | Intermediate System to Intermediate System | Link-state routing protocol used by TRILL and some underlays |
| **OSPF** | Open Shortest Path First | Common link-state IGP routing protocol |
| **RSTP** | Rapid Spanning Tree Protocol | Faster-converging STP variant |
| **MSTP** | Multiple Spanning Tree Protocol | Maps multiple VLANs to spanning-tree instances |
| **SVI** | Switched Virtual Interface | L3 gateway interface for a VLAN on a switch |
| **ASIC** | Application-Specific Integrated Circuit | Hardware chip for line-rate forwarding |
| **FIB** | Forwarding Information Base | Data-plane forwarding table |
| **DCI** | Data Center Interconnect | Network interconnection between data centers |
| **Clos** | Clos Network | Multi-stage topology commonly implemented as Spine-Leaf |
| **NFV** | Network Functions Virtualization | Software-based network functions on commodity infrastructure |
| **VNF** | Virtualized Network Function | A network function delivered as a VM or software process |
| **CNF** | Cloud-native Network Function | A containerized/cloud-native network function |
| **DPDK** | Data Plane Development Kit | User-space packet I/O and data-plane acceleration framework |
| **SPDK** | Storage Performance Development Kit | User-space high-performance storage framework |
| **SmartNIC** | Smart Network Interface Card | NIC capable of offloading infrastructure processing |
| **DPU** | Data Processing Unit | Programmable processor for infrastructure data-plane services |
| **OVS** | Open vSwitch | Software switch commonly used for VMs, containers, and NFV |
| **VPP** | Vector Packet Processing | Batch/vector packet-processing framework |
| **SR-IOV** | Single Root I/O Virtualization | Splits a PCIe device into multiple virtual functions |
| **XDP** | eXpress Data Path | Early packet-processing path in Linux |
| **RDMA** | Remote Direct Memory Access | Remote memory access with reduced CPU/kernel involvement |
| **RoCE** | RDMA over Converged Ethernet | RDMA carried over Ethernet |
| **P4** | Programming Protocol-independent Packet Processors | Language/model for programmable switch data planes |
| **NUMA** | Non-Uniform Memory Access | Different costs for local and remote memory on multi-socket systems |
| **PMD** | Poll Mode Driver | Driver that polls NIC queues in user-space data planes |
| **NFVI** | NFV Infrastructure | Compute, network, storage, and acceleration resources for VNFs/CNFs |
| **virtio** | Virtual I/O | Common paravirtualized device interface for VMs |
| **VF** | Virtual Function | NIC virtual function exposed by SR-IOV |
| **VPN** | Virtual Private Network | Logical private network over a public/shared network |
| **IPsec** | Internet Protocol Security | IP-layer authentication, integrity, and encryption |
| **IKE** | Internet Key Exchange | Negotiates IPsec peers, algorithms, and session keys |
| **GRE** | Generic Routing Encapsulation | Encapsulates one network-layer protocol inside another IP packet |
| **MPLS** | Multiprotocol Label Switching | Label-based forwarding in provider/enterprise backbones |
| **SD-WAN** | Software-Defined Wide Area Network | Policy-driven control of multiple WAN links |
| **NAT-T** | NAT Traversal | UDP encapsulation that lets IPsec cross NAT |
| **CPE** | Customer Premises Equipment | Customer-side router/security device for a provider or cloud link |
| **VRF** | Virtual Routing and Forwarding | Multiple isolated routing tables on one device |
| **BFD** | Bidirectional Forwarding Detection | Fast detection of forwarding or neighbor failure |

---

## 1. Layering and Data Delivery

### 1.1 OSI Seven-Layer Model

| Layer | Name | Protocols / Devices | Purpose |
|-------|------|---------------------|---------|
| 7 | **Application** | HTTP, FTP, DNS | Application interface |
| 6 | **Presentation** | Encryption, compression | Data format conversion |
| 5 | **Session** | Session management | Establish and manage sessions |
| 4 | **Transport** | TCP, UDP | End-to-end communication |
| 3 | **Network** | IP, ICMP, routers | Routing |
| 2 | **Data Link** | MAC, switches | Local network communication |
| 1 | **Physical** | NIC, fiber | Bit transmission |

### 1.2 TCP/IP Four-Layer Model

| TCP/IP Layer | OSI Mapping | Protocols |
|--------------|-------------|-----------|
| **Application** | Layers 7-5 | HTTP, DNS, FTP, SMTP |
| **Transport** | Layer 4 | TCP, UDP |
| **Internet** | Layer 3 | IP, ICMP, ARP |
| **Link** | Layers 2-1 | Ethernet, Wi-Fi |

For backend interviews, the TCP/IP model is usually more practical. Most troubleshooting starts from DNS, then connection establishment, transport quality, and finally application protocol semantics.

### 1.3 Encapsulation and One Request's Journey

Think of one HTTPS request as sending an **encrypted registered letter**.

| Everyday idea | Network concept |
|---------------|-----------------|
| Company name | Domain |
| Phone book / map | DNS → IP |
| House number | IP address |
| Which window | Port (`443` for HTTPS) |
| Call before talking | TCP handshake |
| Agree on a lockbox code | TLS handshake |
| Put letter in envelopes | Encapsulation: HTTP → TCP → IP → Ethernet |
| Building lobby rewrite room → street address | NAT |
| Courier hubs | Routers |
| Reception / security / triage | CDN / firewall / LB / gateway |
| Employee who does the work | Application process |

Encapsulation:

```text
HTTP GET /users
-> TCP header (ports, seq)
-> IP header (addresses, TTL)
-> Ethernet header (next-hop MAC)
-> bits on the wire
```

#### Full path for `https://api.example.com/users`

Assume home PC `10.0.0.8`, router/NAT public IP `198.51.100.8`, DNS returns entry `203.0.113.10`, then CDN/LB/gateway → app `10.20.0.15:8080`.

```text
0. The app wants GET /users, but only has a name and intent.
   Before HTTP can be sent, it still needs DNS, TCP, and TLS.
1. DNS works like a phone book:
   cache → recursive resolver → A/AAAA = 203.0.113.10
   This is often the company gate/CDN entry, not the final Pod.
2. Local routing decides "who gets this parcel first":
   destination is off-subnet → default gateway; ARP/NDP learns gateway MAC.
   Destination IP stays 203.0.113.10; Ethernet MAC is only for this hop.
3. The client chooses a temporary "window number" (ephemeral port):
   (10.0.0.8, 52341, 203.0.113.10, 443)
4. Home NAT rewrites the internal room number to the public doorplate:
   10.0.0.8:52341 → 198.51.100.8:40001, with conntrack for return traffic.
5. Internet routers forward like courier hubs:
   longest-prefix route, TTL--, next-hop L2 rewrite; Anycast may hit nearest POP.
6. The request reaches the company gate:
   CDN/WAF → L4 LB → L7 gateway. L4 may pass through or proxy TCP;
   TCP, then TLS, then HTTP parsing happen in that order.
7. TCP is the phone call being connected:
   SYN → SYN-ACK → ACK → ESTABLISHED. The pipe is up, but not yet secure/business.
8. TLS 1.3 is identity check plus agreeing on a lockbox key:
   ClientHello(SNI, key_share, ALPN, cipher suites)
   ← ServerHello + key_share + {Certificate, CertificateVerify, Finished}
   client verifies chain/SAN/expiry; both derive AEAD traffic secrets (~1-RTT)
9. HTTP finally goes inside encrypted TLS records:
   HTTP/1.1 text or HTTP/2 frames → gateway routes → app handles → response.
   HTTP/3 is negotiated by ALPN inside QUIC's TLS handshake over UDP;
   an existing TCP connection is not converted into UDP.
10. The reply returns through the same layers:
    NAT reverses, TCP reassembles, TLS decrypts, UI updates; close or keep-alive.
```

The analogies are only memory aids. A strong answer still names the real artifacts: four-tuple, SNAT/conntrack, SYN queue, SNI, CertificateVerify, ALPN, L4 vs L7.

Troubleshoot by stage: DNS → route → port/firewall → TCP → TLS → gateway/HTTP → application.

---

## 2. IP, Link, Routing, and NAT

### 2.1 IPv4, CIDR, and Subnets

CIDR (Classless Inter-Domain Routing) uses `/prefix-length`. Hosts are on the same subnet when `IP AND netmask` matches.

```text
192.168.1.10/24 and 192.168.1.20/24 -> same subnet
192.168.1.10/24 and 192.168.2.20/24 -> route via gateway
```

Private ranges: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.

```bash
ip addr
ip route
ip route get 8.8.8.8
```

### 2.2 IPv6 and Dual Stack

IPv6 uses 128-bit addresses, NDP instead of ARP, and DNS AAAA records. Dual-stack clients often use Happy Eyeballs to avoid a broken IPv4 or IPv6 path delaying the connection.

### 2.3 MAC, ARP/NDP, and Routing

IP identifies the final destination; MAC identifies the next hop on the local link. Same-subnet traffic ARPs for the peer; remote traffic ARPs for the default gateway. Routing uses longest-prefix match, then the default route.

```bash
ip neigh
ip route get 203.0.113.10
ping -c 4 203.0.113.10
mtr -rw 203.0.113.10
```

ICMP carries more than ping: Destination Unreachable, Time Exceeded, and Fragmentation Needed are important for troubleshooting.

### 2.4 MTU, MSS, and PMTUD

MTU is the maximum IP packet size on a link (Ethernet commonly 1500). TCP MSS is application payload per segment (commonly 1460 for IPv4/TCP without options). PMTUD discovers path MTU. Small requests working while uploads/large responses time out often indicates an MTU black hole.

### 2.5 Ports, Four-Tuple, NAT, and Conntrack

Flows are identified by `(src IP, src port, dst IP, dst port)`. Servers listen on stable ports; clients normally use ephemeral source ports.

NAT may rewrite source (SNAT) or destination (DNAT). Linux conntrack stores state for NAT and stateful firewalling; a full conntrack table causes seemingly random new-connection failures.

```bash
sysctl net.netfilter.nf_conntrack_count
sysctl net.netfilter.nf_conntrack_max
conntrack -S
```

### 2.6 Dedicated Links, VPNs, and WAN Interconnection

The earlier sections explain how one host reaches another network through a gateway. Enterprises and cloud platforms also need to connect offices, branches, IDCs, cloud VPCs, and partners while deciding whether the connection needs predictable bandwidth, private routing, encryption, and failover.

#### What Problem Do Dedicated Links and VPNs Solve?

Suppose IDC A must reach cloud VPC B:

```text
IDC A: 10.1.0.0/16  -----  cloud VPC B: 10.2.0.0/16
```

Using the public Internet directly is awkward: private prefixes are not automatically routed there, NAT complicates return paths, public latency/loss/jitter are variable, and database or management traffic needs a clear security boundary.

| Option | What it really is | Main benefit | Main cost |
|--------|-------------------|--------------|-----------|
| **Internet VPN** | encrypted tunnel over public IP | cheap, fast, flexible | shared Internet quality, MTU/NAT issues |
| **Carrier private circuit** | managed isolated/logically isolated WAN transport | more predictable path and bandwidth | expensive and slow to provision |
| **Cloud private link/interconnect** | dedicated access from enterprise edge to cloud network | stable VPC/multi-cloud connectivity | provider and routing dependencies |
| **SD-WAN** | policy control over multiple WAN links | centralized steering and failover | more controller/device/policy complexity |

A private circuit is not automatically end-to-end encrypted; an IPsec VPN is encrypted but still depends on its underlay. Sensitive traffic may use IPsec even over a private circuit.

#### What Is a Private Circuit?

A private circuit is a managed WAN service, not necessarily a literal cable from one server room to another:

```text
enterprise IDC -- customer CPE -- provider access -- MPLS/IP backbone -- peer/cloud CPE
```

Common forms are:

- **L2 circuit**: both ends appear to share a Layer-2 link; flexible, but the customer owns VLAN/ARP/STP or gateway design and may enlarge the broadcast/failure domain.
- **L3 VPN/MPLS circuit**: provider/cloud edge maintains VRFs and Layer-3 reachability; sites exchange prefixes through BGP or static routes.
- **Cloud interconnect**: customer CPE connects to a cloud provider edge, which attaches the circuit to a VPC, transit gateway, or cloud network.

SLA normally constrains availability, latency, loss, or bandwidth; it does not remove the need for application timeouts, retries, and failover.

#### VPN: Build a Logical Network over an Existing One

VPN (Virtual Private Network) creates a controlled overlay tunnel over an untrusted or shared underlay:

```text
inner packet: 10.1.0.10 -> 10.2.0.20
  -> VPN gateway encapsulates and encrypts
outer packet: public IP A -> public IP B
  -> Internet routers forward only the outer packet
  -> VPN gateway decrypts and decapsulates
inner packet continues to 10.2.0.20
```

Think of a private circuit as a managed freight route and a VPN as a locked container carried over ordinary roads. The first makes path quality easier to contract; the second makes confidentiality and deployment flexibility easier.

A working VPN must answer three separate questions: who may establish the tunnel, whether packets are confidential and tamper-resistant, and which prefixes/policies use the tunnel. “Tunnel established” does not prove that routing, return paths, firewalls, DNS, and MTU are correct.

#### Common VPN Technologies

| Technology | Characteristics | Typical use | Main concerns |
|------------|-----------------|-------------|---------------|
| **IPsec site-to-site** | Layer-3 encrypted tunnel, often IKEv2 | IDC-to-cloud, branch-to-HQ | IKE/ESP, routing, NAT-T, MTU |
| **SSL/TLS VPN** | client/application access over TLS | remote employees | identity, permissions, full vs split tunnel |
| **WireGuard** | compact modern UDP encrypted tunnel | remote access, lightweight sites | keys and `AllowedIPs` encode identity/routing intent |
| **GRE** | generic encapsulation without encryption | dynamic routing or multi-protocol transport | commonly combined with IPsec |
| **VXLAN** | data-center tenant overlay encapsulation | IDC/cloud internal networks | not a public security VPN |
| **MPLS L3VPN** | provider VRF/label forwarding | multi-branch enterprise WAN | isolation is not end-to-end encryption |

**GRE over IPsec** is useful when GRE carries OSPF, BGP, multicast, or complex inner protocols and IPsec provides encryption. The combination is more flexible but consumes more MTU.

#### What Happens During an IPsec Setup?

With common IKEv2 + ESP tunnel mode:

```text
site A gateway                                  site B gateway
  | IKE_SA_INIT: algorithms, DH parameters, nonces       |
  |----------------------------------------------------->|
  | IKE_AUTH: identity, certificate/PSK, subnets, CHILD_SA|
  |<-----------------------------------------------------|
  | ESP encrypted traffic: 10.1.0.0/16 <-> 10.2.0.0/16    |
```

- **IKE_SA_INIT** negotiates algorithms and Diffie-Hellman material.
- **IKE_AUTH** authenticates peers and confirms protected inner networks.
- **CHILD_SA** creates the security association and traffic keys.
- **ESP** protects the actual IP packets with encryption, integrity, sequence numbers, and replay protection.

If NAT is present, NAT-T usually carries ESP inside UDP 4500; IKE commonly uses UDP 500. The outer packet being UDP does not make the tunnel ordinary unauthenticated UDP.

##### DH Key Agreement: How Two Peers Derive the Same Key Without Sending It

DH (Diffie-Hellman) is easiest to misunderstand if you call it “encryption.” It is not a symmetric cipher and it does not directly encrypt HTTP or IPsec packets. It is a **key agreement** method: both sides exchange public material over the network, yet independently derive the same shared secret.

Start with the intuition:

```text
public material: one public base color, visible to everyone
client secret: its private red recipe
server secret: its private blue recipe

client: public base + red recipe -> orange mixture, sent to server
server: public base + blue recipe -> green mixture, sent to client

client: green mixture + red recipe -> final color
server: orange mixture + blue recipe -> the same final color
```

An observer can see the public base, orange mixture, and green mixture, but cannot efficiently separate out the private red or blue recipe. If the “mixing” operation is easy forward and hard backward, both peers can arrive at the same final result. That final result corresponds to the shared secret used to derive traffic keys.

Map the analogy back to finite-field DH:

| Analogy | DH object | Sent publicly? |
|---------|-----------|----------------|
| public base color | large prime `p` and generator `g` | yes |
| client secret red recipe | client private random `a` | no |
| server secret blue recipe | server private random `b` | no |
| orange mixture | `A = g^a mod p` | yes |
| green mixture | `B = g^b mod p` | yes |
| final color | `K = g^(ab) mod p` | no, each side derives it locally |

Classic finite-field DH:

```text
public parameters: large prime p and generator g
client chooses private random a
server chooses private random b

client computes A = g^a mod p and sends A
server computes B = g^b mod p and sends B

client: K = B^a mod p = (g^b)^a mod p = g^(ab) mod p
server: K = A^b mod p = (g^a)^b mod p = g^(ab) mod p
```

The wire contains `p`, `g`, `A`, and `B`, but not `a`, `b`, or `K`:

```text
Client                         Server
  private a                      private b
  A = g^a mod p  -------------->
                 <-------------- B = g^b mod p
  K = B^a mod p                  K = A^b mod p
                  same K
```

The hard part for an eavesdropper is reversing `A = g^a mod p` to recover `a`, which is the discrete-log problem. With strong groups or safe elliptic curves, that is computationally infeasible in practice.

So the paint analogy is only for remembering three facts: **public material can be sent, private randomness is never sent, and the shared secret is derived locally by both sides**. The security comes from one-way modular-exponentiation or elliptic-curve operations, not from literal colors.

**ECDH** applies the same idea to elliptic curves. **ECDHE** adds “ephemeral”: fresh private values per handshake, providing forward secrecy. TLS 1.3 `key_share` is ECDHE public material; it is not a private key or an AEAD traffic key.

DH alone does not authenticate the peer. An attacker can perform one DH exchange with the client and another with the server. Certificates and `CertificateVerify` bind the handshake to the server identity and prevent that man-in-the-middle attack:

```text
ECDHE: agree on a temporary shared secret
certificate/signature: prove the server owns the trusted certificate private key
AEAD: efficiently encrypt the actual HTTP data
```

#### IKE SA, CHILD SA, and IPsec SA

An SA (Security Association) is not a physical circuit. It is a set of **direction-specific security parameters**: traffic selectors, key, algorithm, SPI, sequence number, replay window, and lifetime.

| Object | Role | Directly carries business IP packets? |
|--------|------|---------------------------------------|
| **IKE SA** | protects IKE control messages and manages later SAs | no |
| **CHILD SA** | an IKEv2-negotiated business security association, used for initial traffic or rekey | indirectly |
| **IPsec SA** | the one-way data-plane key and parameters used by ESP/AH | yes |

A bidirectional tunnel normally has at least two opposite-direction IPsec SAs:

```text
SA-A->B: A outbound to B, key K1, SPI 1001
SA-B->A: B outbound to A, key K2, SPI 2001
```

Do not imagine one bidirectional shared connection. IPsec normally maintains separate keys, SPIs, sequence numbers, and lifetimes per direction.

Gateways commonly maintain two relevant tables:

- **SPD (Security Policy Database)** decides whether matching inner traffic is `PROTECT`, `BYPASS`, or `DISCARD`.
- **SAD (Security Association Database)** maps an SA/SPI to keys, algorithms, mode, sequence state, and replay window.

For `10.1.0.0/16 -> 10.2.0.0/16`, the outbound data path is:

```text
1. host sends an ordinary IP packet to gateway A
2. gateway A checks SPD: this selector requires PROTECT
3. it selects SA-A->B and looks up the key/algorithm/SPI in SAD
4. ESP encrypts the inner IP packet and adds SPI, sequence number, and integrity tag
5. an outer IP header is added: gateway A -> gateway B
6. ordinary routers forward only by the outer destination
7. gateway B finds the opposite-direction SA by SPI, checks replay/integrity, and decrypts
8. it removes ESP/outer headers and routes the original packet to 10.2.0.20
```

The return direction uses a different SA. Decryption does not automatically create a return route; the peer still needs a route and matching policy.

#### ESP Tunnel-Mode Packet Layout

The common site-to-site form is tunnel mode:

```text
outer Ethernet
  -> outer IP: VPN gateway A -> VPN gateway B
    -> ESP header: SPI + sequence number
      -> encrypted: original inner IP header + TCP/UDP + payload + padding
        -> ESP trailer / authentication tag
```

The endpoints expose the gateway addresses to the transit network. The protected content is the complete inner IP packet, so intermediate routers cannot see tenant host addresses and ports, although timing, size, and gateway metadata may remain visible.

Transport mode protects the transport-layer portion while retaining the original IP header and is more common in host-to-host arrangements. Do not equate IPsec tunnel mode with any UDP tunnel: IPsec defines the security processing and SA; UDP may only be its outer carrier under NAT-T.

#### SA Lifetime, Rekey, and Failure Symptoms

SAs usually have both time and byte/packet lifetimes. Before expiry, peers rekey through IKE; old and new SAs may briefly coexist so in-flight packets are not abruptly invalidated.

| State/failure | Typical symptom |
|---------------|-----------------|
| IKE SA down | cannot negotiate or refresh CHILD SAs |
| CHILD/IPsec SA down | control channel may appear alive, but business packets fail |
| unknown SPI | peer sends packets using stale SA; local gateway drops or renegotiates |
| replay-window failure | packets are rejected as duplicates or excessively late |
| policy mismatch | tunnel is “up,” but selected prefixes or ports do not pass |
| stale rekey state | one-way traffic, packet loss, or repeated renegotiation |

Troubleshooting should inspect IKE state, CHILD/IPsec SAs, SPD/SAD, routes, firewall counters, and ESP/NAT-T captures separately. A green VPN status alone is not proof that business traffic works.

#### Full Tunnel, Split Tunnel, and High Availability

```text
full tunnel : all traffic -> VPN -> enterprise egress -> Internet or private network
split tunnel: private prefixes -> VPN; ordinary Internet -> local egress
```

Full tunnel simplifies audit and centralized security but consumes enterprise egress and adds latency. Split tunnel improves user experience but lets local Internet traffic bypass enterprise controls.

Production connectivity should include dual CPEs, dual cloud gateways, dynamic BGP routing, BFD/health checks, and a tested Internet VPN or 4G/5G backup. Active-active designs must account for ECMP, NAT/session state, and asymmetric paths.

SD-WAN is not a new encryption protocol. It is a control and orchestration approach that manages MPLS, Internet, 4G/5G, and cloud circuits, selecting paths according to latency, loss, jitter, cost, and application policy. It may carry IPsec or GRE overlays.

#### Dedicated Circuit, VPN, Overlay, and DCI

| Concept | Main question | Encrypted by default? | Bandwidth guaranteed by default? |
|---------|---------------|-----------------------|----------------------------------|
| private circuit | what managed WAN transport connects sites? | not necessarily | usually an SLA question |
| IPsec VPN | how do we create an encrypted tunnel over IP? | yes | no; depends on underlay |
| GRE | how do we carry inner protocols or routing? | no | no |
| MPLS L3VPN | how does a provider isolate customer WANs? | not necessarily end-to-end | usually has a provider SLA |
| VXLAN overlay | how does an IDC/cloud virtualize tenant networks? | usually no | depends on underlay |
| DCI | how do multiple data centers connect or replicate? | depends on design | depends on transport |

In one sentence: **a private circuit is a transport service, a VPN is a tunnel/security mechanism, an overlay is a virtual network, SD-WAN controls multiple WAN paths, and DCI is the cross-data-center use case.**

---

## 3. TCP In Depth

### 3.1 TCP vs UDP

| Dimension | TCP | UDP |
|-----------|-----|-----|
| **Connection** | Connection-oriented, three-way handshake | Connectionless |
| **Reliability** | Reliable, ACK and retransmission | Best effort |
| **Ordering** | Ordered byte stream | No ordering guarantee |
| **Speed** | Higher overhead | Lower overhead |
| **Header size** | 20 bytes minimum | 8 bytes |
| **Use cases** | HTTP, FTP, email, RPC | DNS, video, games, QUIC |

TCP optimizes reliable ordered delivery. UDP optimizes simplicity and low latency. Protocol selection should start from business requirements, not from "which one is faster".

### 3.2 TCP Three-Way Handshake

```text
Client                         Server
  |                              |
  |  1. SYN seq=x                |
  |----------------------------->|
  |                              |
  |  2. SYN-ACK seq=y ack=x+1    |
  |<-----------------------------|
  |                              |
  |  3. ACK seq=x+1 ack=y+1      |
  |----------------------------->|
  |                              |
  |  Connection established      |
```

Why three handshakes?

- The first SYN proves the client can send.
- The SYN-ACK proves the server can receive and send.
- The final ACK proves the client can receive, so the server can safely enter `ESTABLISHED`.

Two handshakes are not enough because the server cannot confirm that the client received the server's sequence number. Four handshakes are unnecessary because SYN and ACK can be combined in the second packet.

**State transitions**:

```text
Client: CLOSED -> SYN_SENT -> ESTABLISHED
Server: CLOSED -> LISTEN -> SYN_RECV -> ESTABLISHED
```

### 3.2.1 Kernel-Level Handshake Notes

**SYN Flood, SYN queue, and syncookies**

**Half-open / SYN queue**: after the server receives SYN and sends SYN-ACK but has not yet got the client's final ACK, the socket is `SYN_RECV`. Those entries sit in the SYN queue and consume kernel memory. Distinct from the **accept queue** (completed handshakes waiting for `accept()`).

```bash
cat /proc/sys/net/ipv4/tcp_max_syn_backlog   # SYN queue size hint
cat /proc/sys/net/ipv4/tcp_syncookies
# 0=off, 1=use cookies when SYN queue is exhausted (default), 2=always cookies
```

**SYN Flood**: attacker sends many SYNs (often spoofed) and never completes the third ACK. The SYN queue fills; legitimate SYNs fail. It is a memory/queue exhaustion attack.

| Path | Without syncookie (or queue not full) | With syncookie |
|------|----------------------------------------|----------------|
| On SYN | Allocate SYN_RECV entry in queue | Encode state into SYN-ACK seq (cookie); little/no queue slot |
| On final ACK | Match queued state → ESTABLISHED | Verify cookie → ESTABLISHED |
| Under flood | Queue full → good clients blocked | Fake SYNs cost CPU hash, not queue slots |

Trade-off: cookie mode historically limited some TCP option negotiation; modern kernels improved this. Production usually keeps `tcp_syncookies=1`.

**Listen backlog and accept queue**:

```bash
# listen(fd, backlog) limits the queue of connections that completed handshake
# but have not yet been accepted by the application.
# Effective value = min(backlog, somaxconn)
cat /proc/sys/net/core/somaxconn

# Inspect listen queue usage
ss -lnt
# LISTEN Recv-Q Send-Q Local:Port
#        queued max

# Overflow counters
netstat -s | grep "listen queue"
# or
nstat | grep ListenOverflows
```

**SYN/SYN-ACK retry counts**:

```bash
cat /proc/sys/net/ipv4/tcp_syn_retries
cat /proc/sys/net/ipv4/tcp_synack_retries
```

**TCP Fast Open** allows application data inside SYN for repeat connections:

```bash
echo 3 > /proc/sys/net/ipv4/tcp_fastopen
# 0=off, 1=client, 2=server, 3=both

# First connection: client requests a cookie.
# Later connection: client sends SYN + cookie + data, saving one RTT.
```

### 3.3 TCP Four-Way Termination

```text
Client                         Server
  |                              |
  |  1. FIN seq=u                |
  |----------------------------->|
  |                              |
  |  2. ACK ack=u+1              |
  |<-----------------------------|
  |                              |
  |  3. FIN seq=w                |
  |<-----------------------------|
  |                              |
  |  4. ACK ack=w+1              |
  |----------------------------->|
  |                              |
  |  Connection closed           |
```

The active closer enters `TIME_WAIT`. In Linux, it typically waits 60 seconds. Conceptually, TIME_WAIT is often explained as `2MSL`, where MSL is Maximum Segment Lifetime.

### 3.3.1 Why TIME_WAIT Is 2MSL

TIME_WAIT exists for two reasons.

**Reason 1: make sure the final ACK can be retransmitted if needed**

```text
If the final ACK is lost:
1. The peer retransmits FIN.
2. The active closer in TIME_WAIT can still receive FIN.
3. It sends ACK again.

2MSL = 1MSL for the retransmitted FIN to arrive + 1MSL for ACK to return.
```

**Reason 2: prevent old packets from interfering with a new connection**

```text
Connection A:
client:1234 -> server:80

Connection A closes.
A delayed packet from A may still exist in the network.

If Connection B immediately reuses the same four-tuple:
client:1234 -> server:80

The delayed packet may be mistaken as part of B.
TIME_WAIT waits long enough for old packets to disappear.
```

### 3.3.2 Too Many TIME_WAIT Connections

**Ephemeral ports are not the fixed service port.** A TCP connection is a four-tuple `(src_ip, src_port, dst_ip, dst_port)`. When you `connect` to `example.com:443`, **443 is fixed on the server**; the client's **source port** is chosen by the kernel from `ip_local_port_range` (e.g. 32768–60999)—a new short connection often gets a new ephemeral port. App code normally does not hardcode the source port.

Port exhaustion hits when many short connections to the **same** destination leave source ports in TIME_WAIT (~60s on Linux) and the ephemeral pool runs out → `Cannot assign requested address`.

Problems:

- Ephemeral port exhaustion on clients that create many short connections.
- Extra memory and CPU overhead for connection state and timers.

Common diagnosis:

```bash
ss -s | grep TIME-WAIT
netstat -an | grep TIME_WAIT | wc -l
ss -tapn | grep TIME-WAIT
```

Practical mitigations:

```bash
# Reuse TIME_WAIT ports for outbound client connections.
sysctl -w net.ipv4.tcp_tw_reuse=1

# Shorten FIN_WAIT_2 timeout.
sysctl -w net.ipv4.tcp_fin_timeout=30

# Increase ephemeral port range.
sysctl -w net.ipv4.ip_local_port_range="1024 65535"
```

Best fix: use connection pools and keep-alive instead of frequent short connections.

```go
http.Transport{
    MaxIdleConns:        100,
    MaxIdleConnsPerHost: 10,
    IdleConnTimeout:     90 * time.Second,
}
```

`tcp_tw_reuse` is much safer than the removed `tcp_tw_recycle`. `tcp_tw_recycle` depended on monotonically increasing TCP timestamps from the same peer IP, which breaks badly behind NAT because multiple clients may share one public IP with different timestamp sequences.

### 3.3.3 CLOSE_WAIT and Other Closing States

| State | Why It Stays | Timeout | Fix |
|-------|--------------|---------|-----|
| **FIN_WAIT1** | Peer did not ACK FIN | TCP retransmission timer | Tune orphan retries only if necessary |
| **FIN_WAIT2** | Peer did not send FIN | `tcp_fin_timeout` | Reduce timeout if needed |
| **CLOSE_WAIT** | Local application did not close socket | No kernel timeout | Fix application code |
| **LAST_ACK** | Peer did not ACK final FIN | TCP retransmission timer | Same as FIN_WAIT1 |
| **TIME_WAIT** | Wait for old packets to disappear | Fixed wait in Linux | Reuse, pooling, reduce short connections |

If you see many `CLOSE_WAIT`, it is almost always an application bug, such as not closing response bodies or connections on error paths.

### 3.4 Sliding Window and Flow Control ⭐⭐⭐

Without a window, TCP would send one packet and wait for its ACK before sending the next one. That wastes bandwidth on high-latency networks.

```text
Stop-and-wait:
send 1 segment -> wait for ACK -> send next segment

Sliding window:
send N segments continuously -> slide window as ACKs arrive
```

The effective send window is:

```text
send_window = min(cwnd, rwnd)

cwnd: congestion window, controlled by sender based on network feedback.
rwnd: receive window, advertised by receiver based on available buffer.
```

**Zero window probe**:

```text
1. Receiver buffer is full -> advertises rwnd=0.
2. Sender stops sending.
3. Receiver later frees buffer and sends window update.
4. If the update ACK is lost, both sides may deadlock.

Solution:
Sender starts a persist timer and periodically sends a 1-byte probe.
Receiver replies with current rwnd.
```

### 3.4.1 Flow Control vs Congestion Control

| Dimension | Flow Control | Congestion Control |
|-----------|--------------|--------------------|
| **Problem** | Do not overwhelm the receiver | Do not overwhelm the network |
| **Variable** | `rwnd` | `cwnd` |
| **Decided by** | Receiver | Sender |
| **Signal** | TCP Window field | Loss, duplicate ACKs, RTT, bandwidth signals |
| **Relationship** | Actual send window = min(cwnd, rwnd) | Works together with flow control |

### 3.4.2 Nagle Algorithm and Delayed ACK

Nagle reduces tiny TCP packets:

```text
If there is unacknowledged data:
  buffer small writes until enough data is collected or ACK arrives.
If no unacknowledged data:
  send immediately.
```

Delayed ACK reduces ACK packets:

```text
The receiver waits briefly before sending ACK, often up to around 200ms.
```

Together they can hurt interactive workloads:

```text
Sender sends a small packet and waits because of Nagle.
Receiver delays ACK.
Result: every small message may pay extra latency.
```

Disable Nagle for interactive protocols:

```c
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &on, sizeof(on));
```

Use cases: games, SSH, real-time control messages.

### 3.5 TCP Congestion Control

Classic algorithms:

1. **Slow Start**: `cwnd` grows exponentially from a small initial value.
2. **Congestion Avoidance**: after reaching `ssthresh`, `cwnd` grows linearly.
3. **Fast Retransmit**: retransmit after 3 duplicate ACKs, without waiting for timeout.
4. **Fast Recovery**: after fast retransmit, reduce `cwnd` but do not fall back to 1 MSS unless timeout happens.

```text
Slow Start:
cwnd: 1 -> 2 -> 4 -> 8 -> 16 ...

Congestion Avoidance:
cwnd: 16 -> 17 -> 18 -> 19 ...
```

Why exponential first and linear later?

- Exponential growth quickly discovers available bandwidth.
- Linear growth cautiously probes capacity near the congestion point.

**Fast retransmit example**:

```text
Sender sends: 1000, 1500, 2000, 2500, 3000
Segment 1500 is lost.

Receiver ACKs:
ACK=1500
ACK=1500 duplicate 1
ACK=1500 duplicate 2
ACK=1500 duplicate 3

Sender retransmits seq=1500 immediately.
```

**Algorithm evolution**:

| Algorithm | Key Idea | Notes |
|-----------|----------|-------|
| Tahoe | Loss -> `cwnd=1` | Conservative |
| Reno | Fast recovery on 3 duplicate ACKs | Better performance |
| NewReno | Handles multiple losses better | Widely used conceptually |
| CUBIC | Cubic window growth | Linux default for many years |
| BBR | Model bandwidth and RTT, not primarily loss | Good for high-latency/lossy paths |

Check and change the congestion control algorithm:

```bash
sysctl net.ipv4.tcp_congestion_control
sysctl net.ipv4.tcp_available_congestion_control
sysctl -w net.ipv4.tcp_congestion_control=bbr
```

### 3.6 TCP Packet Sticking and Splitting

TCP is a byte-stream protocol. It does not preserve application message boundaries. This causes "packet sticking" or "packet splitting" at the application layer.

Common framing solutions:

1. Fixed-length messages.
2. Delimiters such as `\n`.
3. Length-prefixed frames, usually preferred.

```go
func Encode(data []byte) []byte {
    length := uint32(len(data))
    buf := make([]byte, 4+len(data))
    binary.BigEndian.PutUint32(buf[:4], length)
    copy(buf[4:], data)
    return buf
}

func Decode(buf []byte) ([]byte, int) {
    if len(buf) < 4 {
        return nil, 0
    }
    length := binary.BigEndian.Uint32(buf[:4])
    if len(buf) < int(4+length) {
        return nil, 0
    }
    data := buf[4 : 4+length]
    return data, int(4 + length)
}
```

### 3.7 Socket Buffers and TCP Performance

Socket buffer size strongly affects throughput on high-latency links.

```bash
# Global socket buffers
cat /proc/sys/net/core/rmem_default
cat /proc/sys/net/core/rmem_max
cat /proc/sys/net/core/wmem_default
cat /proc/sys/net/core/wmem_max

# TCP-specific buffers: min default max
cat /proc/sys/net/ipv4/tcp_rmem
cat /proc/sys/net/ipv4/tcp_wmem
```

Bandwidth-delay product:

```text
BDP = bandwidth * RTT

Example:
1Gbps = 125MB/s
RTT = 400ms
BDP = 125MB/s * 0.4s = 50MB
```

If socket buffers are much smaller than BDP, the sender cannot keep enough data in flight to fill the network pipe.

Tuning example:

```bash
# 1Gbps, 200ms RTT -> BDP around 25MB
echo "4096 131072 52428800" > /proc/sys/net/ipv4/tcp_wmem
echo "4096 131072 104857600" > /proc/sys/net/ipv4/tcp_rmem

cat /proc/sys/net/ipv4/tcp_moderate_rcvbuf
```

Tuning priority:

1. Low RTT under 5ms: buffer tuning usually matters little.
2. Medium RTT 5-50ms: moderate tuning may help.
3. High RTT over 50ms: calculate BDP and tune buffers.
4. High RTT plus loss: combine buffer tuning with BBR evaluation.

---

## 4. UDP and Transport Selection

### 4.1 UDP Characteristics

Advantages:

- Connectionless, no handshake.
- Low overhead and low latency.
- No built-in ACK or retransmission.
- Supports broadcast and multicast.

Limitations:

- Packet loss.
- Reordering.
- No flow control.
- No congestion control unless implemented by the application protocol.

### 4.2 Why Real-Time Systems Often Use UDP

Real-time games prefer recent state over guaranteed delivery of old state.

| Scenario | Protocol | Reason |
|----------|----------|--------|
| Login, payment | TCP | Must be reliable |
| Chat | TCP or reliable channel | Must be reliable |
| Movement, attack input | UDP | Real-time priority |
| Match result | TCP or reliable channel | Must be reliable |

For position updates, retransmitting stale packets can make the game feel worse. It is often better to drop old state and use the latest update.

### 4.3 Reliable Transport over UDP

**KCP**:

- Reliable protocol over UDP.
- Uses ARQ, fast retransmit, and fast recovery.
- Trades bandwidth for lower latency.

```go
import "github.com/xtaci/kcp-go"

listener, _ := kcp.ListenWithOptions(":8080", nil, 10, 3)
for {
    conn, _ := listener.AcceptKCP()
    go handleKCPConn(conn)
}

conn, _ := kcp.DialWithOptions("127.0.0.1:8080", nil, 10, 3)
conn.Write([]byte("Hello KCP"))
```

**QUIC**:

- Built on UDP.
- 1-RTT and 0-RTT handshakes.
- Multiplexed streams without TCP-level head-of-line blocking.
- Connection migration across network changes.

---

## 5. HTTP

### 5.1 Request and Response

```http
GET /api/users/1001 HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: application/json
Connection: keep-alive
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 123
Connection: keep-alive

{"id":1001,"name":"Alice"}
```

### 5.2 HTTP Methods

| Method | Meaning | Idempotent? | Safe? |
|--------|---------|-------------|-------|
| **GET** | Retrieve resource | Yes | Yes |
| **POST** | Create or submit for processing | No | No |
| **PUT** | Full replacement | Yes | No |
| **PATCH** | Partial update | Usually no | No |
| **DELETE** | Delete resource | Yes | No |

The primary difference between methods is semantics, not whether they can carry parameters.

### 5.3 Status Codes and Common Headers

| Class | Examples |
|-------|----------|
| 2xx success | 200, 201, 204 |
| 3xx redirect/cache | 301/308 permanent, 302/307 temporary, 304 cached |
| 4xx client/request | 400, 401 unauthenticated, 403 unauthorized, 404, 409, 429 |
| 5xx server/gateway | 500, 502 invalid upstream, 503 unavailable, 504 upstream timeout |

Common headers: `Host`, `Content-Type`, `Accept`, `Authorization`, `Cookie`, `Set-Cookie`, `Content-Length`, `Range`, `Forwarded`/`X-Forwarded-For`, and `Traceparent`.

### 5.4 HTTP/1.1 vs HTTP/2 vs HTTP/3

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| **Transport** | TCP | TCP | QUIC over UDP |
| **Multiplexing** | No | Yes | Yes |
| **Header compression** | No | HPACK | QPACK |
| **Head-of-line blocking** | Severe | Still exists at TCP layer | Solved at QUIC stream level |

HTTP/2 improves connection reuse and multiplexing, but TCP-level packet loss can still block all streams. HTTP/3 moves multiplexing into QUIC so loss on one stream does not block unrelated streams.

### 5.5 HTTP Caching ⭐⭐⭐

**Strong cache**: no request is sent to the server.

```text
Cache-Control has higher priority than Expires.

Common directives:
- max-age=3600
- no-cache       # must revalidate before reuse
- no-store       # do not store at all
- public         # shared caches may store
- private        # browser cache only
- s-maxage=3600  # shared cache TTL
```

**Validation cache**: the browser asks the server whether the resource changed.

```text
ETag / If-None-Match:
- More precise; usually content hash or version.
- Server returns 304 Not Modified if unchanged.

Last-Modified / If-Modified-Since:
- Uses file modification time.
- Less precise because timestamp resolution may be one second.
```

Recommended strategies:

```text
HTML:
  Cache-Control: no-cache

Hashed CSS/JS/assets:
  Cache-Control: public, max-age=31536000

Sensitive API responses:
  Cache-Control: no-store

CDN assets:
  Cache-Control: public, s-maxage=86400, max-age=3600
```

### 5.6 Cookie, Session, Token, and CORS

Cookie is a browser-managed transport/storage mechanism; Session stores state server-side; Token/JWT carries credentials for verification. Secure cookies should normally use `Secure`, `HttpOnly`, and an appropriate `SameSite`.

CORS (Cross-Origin Resource Sharing) is a browser policy. Complex cross-origin requests send an `OPTIONS` preflight. Credentialed CORS must use an explicit origin, not `Access-Control-Allow-Origin: *`.

---

## 6. HTTPS and TLS

### 6.1 What HTTPS/TLS Provides

**HTTPS (HTTP Secure / HTTP over TLS) = HTTP over TLS (Transport Layer Security)**. TLS runs on TCP before HTTP.

**SSL vs TLS**:

| Name | Status | Relationship |
|------|--------|--------------|
| **SSL 2.0 / 3.0** (Secure Sockets Layer) | Obsolete | Early Netscape protocol |
| **TLS 1.0** (Transport Layer Security) | Obsolete | Standardized successor based on SSL 3.0 |
| **TLS 1.1** | Obsolete | Small improvement over 1.0 |
| **TLS 1.2** | Still widely used | Modern compatibility baseline |
| **TLS 1.3** | Preferred | Faster, safer, simpler handshake |

So TLS versions before 1.0 were **not** called TLS; they were SSL. TLS is the standardized successor to SSL. “SSL certificate” is mostly a legacy phrase; HTTPS today normally uses a TLS/X.509 certificate.

| Property | Meaning |
|----------|---------|
| Confidentiality | Eavesdroppers cannot read plaintext |
| Integrity | Tampering is detectable (AEAD / MAC) |
| Authentication | Usually the **server** via certificates; client certs optional (mTLS, mutual TLS) |

Ship **TLS 1.2 + 1.3**; **1.0/1.1 are obsolete**. Learn **1.3 as default**, use **1.2 as contrast**.

### 6.2 Why TLS 1.3 — Compare with 1.2?

Yes, compare. TLS 1.3’s “special” points are almost all relative to 1.2:

| | TLS 1.2 | TLS 1.3 |
|--|---------|---------|
| Full handshake | Typically **2-RTT (Round-Trip Time)** before app data | Typically **1-RTT** |
| Key share timing | Separate key-exchange flight after Hello/cert | **Key Share in ClientHello**; server finishes in first reply |
| Encryption of handshake | Cert often cleartext in classic flows | Most post-ServerHello handshake msgs encrypted |
| Crypto cleanup | Legacy RSA key transport, CBC, RC4, … | AEAD + ECDHE; many weak suites removed |
| Forward secrecy | Optional | **Required** (ephemeral key exchange) |
| Resumption | Session ID/ticket (usually 1-RTT) | PSK (Pre-Shared Key) / **0-RTT** (replay risk) |

Three takeaways for 1.3: **faster**, **safer defaults**, **encrypts earlier**.

### 6.3 TLS 1.2 Handshake (contrast: why 2-RTT)

Modern ECDHE full handshake (simplified):

```text
Client                                              Server
  |  ClientHello                                     |
  |------------------------------------------------->|
  |  ServerHello + Certificate + ServerKeyExchange   |
  |  + ServerHelloDone  (server ephemeral public key)|
  |<-------------------------------------------------|
  |  ClientKeyExchange + CCS + Finished              |
  |------------------------------------------------->|
  |  CCS + Finished                                  |
  |<-------------------------------------------------|
  |  Encrypted HTTP                                  |
```

After the first round-trip, the client typically has not yet sent its key-exchange material. A second flight finishes KE → **2-RTT**. Older **RSA key transport** was even more ordered: the client needed the certificate first to encrypt the premaster secret (and lacked PFS). So 1.2’s extra RTT is mainly **legacy message ordering**, not “sending a public key early is unsafe.”

### 6.4 TLS 1.3 Handshake (main path: why 1-RTT)

**Misconception:** “1.3 can put key material in the first flight because it fixed a risk that blocked 1.2.”
**No.** `key_share` carries an **ephemeral ECDHE public key**, which is meant to be public. The latency win comes from **redesigning the handshake** so the client may *speculate* a group and send `key_share` in `ClientHello`. The risky feature is **0-RTT early application data** (replay), which is unrelated to sending `key_share` — see §6.5.

What changed vs 1.2:

1. Removed RSA key transport; key establishment is (EC)DHE / PSK with mandatory PFS.
2. Client may include `key_share` immediately (e.g. x25519) instead of waiting for `ServerKeyExchange`.
3. Server returns its `key_share` in the first response flight; most post-`ServerHello` handshake messages are encrypted.
4. Wrong group guess → **HelloRetryRequest (HRR)** and an extra RTT.

```text
Client                                              Server
  |  ClientHello (+ key_share, groups, SNI, ALPN…)   |
  |------------------------------------------------->|
  |  ServerHello (+ key_share)                       |
  |  {EncryptedExtensions, Certificate,              |
  |   CertificateVerify, Finished}                   |
  |<-------------------------------------------------|
  |  {Finished}                                      |
  |------------------------------------------------->|
  |  Encrypted HTTP   ← ~1-RTT from ClientHello      |
```

`{...}` = encrypted with handshake keys after ServerHello. Both sides derive the same secrets from the ECDHE shared secret (via HKDF), then AEAD traffic keys. Certificate chain + `CertificateVerify` still prove the server holds the cert private key.

The general DH/ECDHE key-agreement explanation is in the IPsec subsection above. TLS 1.3 `key_share` carries ephemeral ECDHE public material; certificates and `CertificateVerify` authenticate the server.

Contrast:

```text
1.2 ECDHE: server reveals ephemeral key first → client replies → ~2-RTT
1.3:       client guesses key_share first → server agrees → ~1-RTT
           (HRR if the guess is wrong)
```

### 6.5 Resumption and 0-RTT

| Mode | Latency | Note |
|------|---------|------|
| Full handshake | ~1-RTT (1.3) | First visit / expired ticket |
| PSK 1-RTT resume | 1-RTT | Cheaper crypto |
| **0-RTT early data** | 0-RTT | Fast; **replayable** — only for idempotent reads, never payments |

### 6.6 Practice

- Prefer 1.3, keep 1.2 for older clients; disable 1.0/1.1.
- Serve a complete certificate chain.
- TLS 1.3 post-hello is ciphertext in captures; use SSLKEYLOGFILE to decrypt.
- HTTP/3 embeds TLS 1.3 inside QUIC (see §11); do not equate it with TCP + separate TLS record layer.

### 6.7 PKI Operations

PKI (Public Key Infrastructure) chains a server certificate through an intermediate CA to a trusted root. Clients verify chain signatures, SAN hostname, validity dates, and revocation state.

Key operational concepts: SNI selects the certificate for a hostname; ALPN negotiates `h2`/`http/1.1`; OCSP Stapling carries certificate status; HSTS forces future HTTPS; mTLS authenticates both peers.

```bash
openssl s_client -connect api.example.com:443 \
  -servername api.example.com -showcerts -alpn h2
```

---

## 7. Real-Time Communication

### 7.1 WebSocket Handshake

```http
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
```

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

After the upgrade, WebSocket provides full-duplex communication over a long-lived connection.

### 7.2 Go Example

```go
import "github.com/gorilla/websocket"

var upgrader = websocket.Upgrader{
    CheckOrigin: func(r *http.Request) bool { return true },
}

func handleWebSocket(w http.ResponseWriter, r *http.Request) {
    conn, _ := upgrader.Upgrade(w, r, nil)
    defer conn.Close()

    for {
        messageType, message, err := conn.ReadMessage()
        if err != nil {
            break
        }
        conn.WriteMessage(messageType, message)
    }
}
```

### 7.3 Heartbeat

```go
func keepAlive(conn *websocket.Conn) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()

    for range ticker.C {
        conn.WriteMessage(websocket.PingMessage, []byte{})
    }
}
```

Application-level heartbeat is often still needed because TCP keepalive is usually too slow for business-level liveness.

### 7.4 WebSocket vs SSE vs Long Polling vs gRPC

| Option | Direction | Best for |
|--------|-----------|----------|
| Short polling | Client periodically requests | Simple low-frequency state |
| Long polling | Server holds HTTP request | Compatibility-first messaging |
| SSE | Server -> client stream | Notifications, logs, AI streaming |
| WebSocket | Full duplex | Chat, collaboration, real-time control |
| gRPC streaming | Uni/bidirectional over HTTP/2 | Typed service-to-service streaming |

All long-lived options need heartbeats, reconnect, backpressure, idempotency, and connection limits.

### 7.5 Video Transport Pipeline: Ingest, Transcode, Delivery, Interaction

Do not compare RTMP, HLS, SRT, and WebRTC as if they all solve the same problem. They usually sit at different points in the live-video pipeline:

```text
camera / broadcaster / encoder
  -> 1. ingest: send the live stream into the platform
  -> 2. transcode / repackage: produce multi-bitrate and multi-protocol outputs
  -> 3. delivery / playback: viewers fetch from CDN or edge nodes
  -> 4. interaction: two-way low-latency sessions such as calls or cloud gaming
```

#### 1. Ingest: From Capture Endpoint to Origin

Ingest asks how one continuous media stream reliably enters the platform. This leg is usually one-to-one or low fan-out; it is not the part serving millions of viewers directly.

| Protocol | Transport | Typical position | Notes |
|----------|-----------|------------------|-------|
| **RTMP** | TCP | OBS/encoder -> live origin | Mature compatibility; still common as an ingest protocol |
| **SRT** | UDP + ARQ/FEC | remote production, weak-network backhaul | Tunable retransmission and jitter resistance |
| **RIST** | UDP + reliability enhancements | broadcast/professional contribution | Standardized reliable contribution transport |
| WebRTC | UDP + RTP/SRTP | ultra-low-latency capture or co-hosting | Low latency, but NAT traversal and server architecture are more complex |

RTMP has not disappeared because OBS, hardware encoders, and live platforms still support it well. Its role has narrowed: it is good for ingest compatibility, but poor as a browser-native or massive CDN playback protocol.

#### 2. Transcoding and Repackaging

After ingest, the origin or media pipeline often transforms the stream:

```text
raw live stream
  -> demux / decode when needed
  -> transcode: 1080p / 720p / 480p adaptive bitrate ladder
  -> repackage: HLS, DASH, LL-HLS, fMP4, CMAF
  -> segment / generate manifests
  -> push to CDN or serve CDN origin fetch
```

Transcoding makes the stream fit different devices and bandwidth levels. Repackaging decides how the stream will be delivered. Codec, container, and transport are different layers: H.264/H.265/AV1, FLV/MP4/fMP4/TS, and RTMP/HTTP/QUIC/RTP should not be collapsed into one concept.

#### 3. Delivery and Playback: CDN-Friendly Fan-Out

Viewer playback asks how to support huge concurrency, nearby access, caching, and graceful degradation. Large-scale live and VOD delivery therefore commonly uses HTTP segment protocols.

| Protocol | Transport | Typical latency | Why it scales |
|----------|-----------|-----------------|---------------|
| **HLS** | HTTP/TCP, or HTTP/3 over QUIC | seconds to tens of seconds | `.m3u8` plus segments; mature browser/mobile/CDN ecosystem |
| **DASH** | HTTP/TCP, or HTTP/3 over QUIC | seconds to tens of seconds | standardized adaptive bitrate playback |
| **LL-HLS / LL-DASH** | HTTP chunks / partial segments | around 1-3 seconds | balances CDN friendliness with lower latency |
| HTTP/3 delivery | QUIC/UDP | depends on segment and player | reduces TCP HOL blocking and improves mobile path changes |

CDNs like HLS/DASH because segments can be cached, prefetched, coalesced, protected from origin storms, and distributed through multiple layers. The trade-off is latency: segment-based delivery buys stability and scale.

#### 4. Interactive Low Latency

Co-hosting, video meetings, online education, and cloud gaming value current audio/video more than perfect ordered delivery of every old frame.

WebRTC is common here:

```text
signaling: WebSocket / HTTPS
media: RTP/RTCP over UDP + SRTP encryption
traversal: ICE + STUN/TURN
weak network: NACK/PLI, FEC, jitter buffer, adaptive bitrate
```

Real-time media often prefers UDP-based stacks because media is time-sensitive: a B/P frame lost two seconds ago may be less useful than skipped. TCP's ordered byte stream can block later data behind one lost segment. On top of UDP, the media stack can retransmit only important data, drop expired frames, use FEC for small loss, and absorb jitter with adaptive buffering.

UDP is not magic. Enterprise firewalls may restrict it, NAT mappings may expire quickly, and congested networks still need congestion control, FEC, retransmission, and player-buffer trade-offs. Many production systems keep TCP/TLS or TURN relay fallbacks.

Summary:

```text
ingest: RTMP remains common; SRT/RIST are stronger for professional weak-network contribution
transcode: one stream becomes multiple bitrates, packages, and playback protocols
delivery: HLS/DASH/LL-HLS use CDN for massive audience fan-out
interaction: WebRTC/RTP suits sub-second two-way real-time sessions
```

### 7.6 Live-Streaming Stall Diagnosis

Sports/live events have a long path:

```text
venue signal -> ingest -> origin/transcode/segment
-> CDN mid-tier/edge -> ISP/last mile -> Wi-Fi/mobile -> player decode
```

Scope is the first clue:

| Symptom | Likely area |
|---------|-------------|
| Only one user stalls | Wi-Fi/mobile, device decode, player buffer, local DNS choice |
| One region/ISP stalls | CDN edge overload, ISP peering, traffic steering mistake |
| Nationwide/global stall | source signal, ingest, transcode, origin, CDN core/backbone |
| Stall at match start/hot moments | flash crowd, hot segment miss, origin shield/backfill pressure |

CDN is essential for large live events: origin cannot serve millions of viewers directly. Live CDN relies on tree-like distribution, segment caching, traffic steering, origin-shielding, multi-CDN, and sometimes P2P-CDN. The hard trade-off is latency vs smoothness: larger buffer is stable but delayed; smaller buffer is low-latency but fragile.

---

## 8. DNS and Domain Resolution

DNS (Domain Name System) maps human-readable names to IP addresses and other resource records. It is distributed, hierarchical, and cache-heavy.

### 8.1 Resolution Flow

```text
Browser
-> browser cache
-> OS cache / hosts
-> recursive resolver (ISP / enterprise / public DNS)
-> root DNS
-> TLD DNS (.com / .cn)
-> authoritative DNS for the zone
-> response cached by TTL
```

**Recursive vs iterative lookup**:

| Type | Who keeps asking | Example |
|------|------------------|---------|
| Recursive | Resolver returns the final answer for the client | Client asks local DNS for `example.com` |
| Iterative | Server returns the next place to ask | Resolver asks root, then TLD, then authoritative DNS |

### 8.2 DNS Record Types

| Type | Meaning | Example |
|------|---------|---------|
| **A** | IPv4 address | example.com -> 192.168.1.1 |
| **AAAA** | IPv6 address | example.com -> 2001:db8::1 |
| **CNAME** | Alias | www.example.com -> example.com |
| **MX** | Mail server | example.com -> mail.example.com |
| **TXT** | Text record | SPF, DKIM, domain verification |
| **NS** | Authoritative name server | example.com -> ns1.provider.com |
| **SOA** | Start of authority | zone serial and refresh policy |
| **SRV** | Service discovery | `_sip._tcp.example.com` |

### 8.3 TTL and Change Propagation

TTL (Time To Live) controls how long a resolver may cache a record.

```text
www.example.com A 1.2.3.4 TTL=300
-> recursive resolver may return cached 1.2.3.4 for 300 seconds
```

DNS changes do not become global immediately because browser, OS, recursive resolver, and ISP caches may all still hold old answers. Some ISP resolvers also clamp minimum/maximum TTL.

**Can a client do anything besides waiting for TTL expiry?**

For the whole internet: mostly no. There is no standard way to force-flush every recursive resolver worldwide. Old answers usually remain until their TTLs expire.

For one machine / one app: yes, with limited scope.

| Action | Effect | Limit |
|--------|--------|-------|
| Clear browser DNS cache / restart browser | Drops browser-local cache | OS + recursive caches remain |
| Flush OS DNS cache | Host re-queries its resolver | Resolver may still return the old IP |
| Temporarily switch to a public DNS | May see the new answer sooner | Only if that resolver is not still caching the old value |
| Temporary `hosts` override | Immediate, bypasses DNS | Local-only; easy to forget |
| App-managed HTTPDNS | App can shorten/invalidate its own cache | Does not help browsers or other clients |
| Flush an enterprise recursive DNS you control | Helps users of that resolver | Cannot flush home ISP resolvers |

Migration practice: lower TTL first, wait for old TTL expiry, switch records while keeping the old IP serving through the previous TTL window, then raise TTL again. Local flush is for troubleshooting, not a global cutover plan.

### 8.4 GSLB

GSLB (Global Server Load Balancing) often uses DNS to steer users to different regions, ISPs, or data centers.

```text
East China Telecom user -> Shanghai Telecom endpoint
South China Mobile user -> Guangzhou Mobile endpoint
Overseas user           -> Singapore endpoint
```

Capabilities:

- geo / ISP-aware routing
- health-check based failover
- weighted rollout and migration
- cross-region disaster recovery

Limits:

- DNS usually sees the recursive resolver IP, not always the real client IP.
- TTL and resolver behavior make failover non-instant.
- DNS chooses an entry point; it does not measure per-connection quality.

### 8.5 Anycast

Anycast is a BGP (Border Gateway Protocol) routing technique, not a DNS record type. Multiple sites advertise the same IP prefix; routing sends clients to the nearest/best reachable site.

```text
Same IP: 203.0.113.53
Beijing, Shanghai, and Singapore all advertise it
Client traffic goes to the best route selected by BGP
```

Common uses: public DNS (`8.8.8.8`, `1.1.1.1`), CDN edge entry, DDoS scrubbing, global service entry.

| | GSLB | Anycast |
|--|------|---------|
| Layer | DNS/application steering | BGP/network routing |
| Client answer | Different IPs possible | Same IP |
| Failover | Affected by TTL/cache | Affected by BGP convergence |
| Control | Fine-grained by domain/region/weight | Route-policy driven |

They are often combined: DNS/GSLB chooses a domain entry or region, and the returned IP may itself be Anycast.

### 8.6 DNS Troubleshooting

```bash
dig www.example.com
dig @8.8.8.8 www.example.com
dig +trace www.example.com
dig www.example.com CNAME
dig example.com NS
```

Compare local resolver vs public resolver vs authoritative trace. Regional failures often involve GSLB policy, resolver cache, ECS (EDNS Client Subnet), or Anycast routing.

### 8.7 Private and Encrypted DNS

Split-horizon DNS returns different answers inside/outside a network. DNSSEC authenticates DNS data but does not encrypt queries. DoT (DNS over TLS) and DoH (DNS over HTTPS) encrypt client-to-resolver traffic. HTTPDNS lets applications query over HTTP(S), useful on mobile networks but requires application-managed caching, SNI, and fallback.

Private zones, conditional forwarding, and CoreDNS search/forwarding rules are common causes of cloud/Kubernetes name-resolution incidents.

---

## 9. Load Balancing, Proxies, CDN, and Global Traffic

### 9.1 Proxies and Gateways

Forward proxies represent clients; reverse proxies represent servers. API gateways add authentication, rate limiting, routing, protocol translation, and observability. Trust `Forwarded`/`X-Forwarded-For` only from controlled proxies.

### 9.2 L4 vs L7 Load Balancing

| | L4 | L7 |
|--|----|----|
| Routes by | IP, port, TCP/UDP | Host, path, headers, cookies |
| Understands HTTP | No | Yes |
| Examples | LVS/IPVS, cloud network LB | Nginx, Envoy, HAProxy |

Operational requirements include health checks, connection draining, explicit timeout stages, weighted rollout, and avoiding sticky sessions where possible.

### 9.3 CDN and Origin Fetch

```text
User -> DNS/GSLB or Anycast -> CDN POP
                               |- cache hit: respond
                               `- miss: fetch from origin
```

Design cache keys carefully; align origin Host/SNI; prevent cache-expiry stampedes; use versioned asset names rather than relying on instant global purge.

### 9.4 CDN, Edge Computing, and Private CDN

CDN moves content closer; edge computing moves logic closer.

| | CDN | Edge computing |
|--|-----|----------------|
| Main job | Cache and distribute static/streaming content | Execute code near users |
| Data | images, assets, packages, video segments | auth, edge functions, A/B, ads, AI inference |
| Benefit | lower latency, lower origin bandwidth | local decisions, less backhaul, faster interaction |

For live streaming, CDN serves video segments near viewers; edge compute may do token validation, anti-hotlink checks, lightweight ad decisions, or simple stream processing.

Public commercial CDN should not usually be used as a shortcut for private cross-IDC synchronization. It moves internal data through public third-party edge nodes and complicates security, compliance, firewall, and origin paths. For private cross-DC distribution, prefer private CDN/cache nodes, object-storage replication, message-driven async sync, dedicated links, private load balancing, or cloud interconnect.

### 9.5 Global Entry Stack

```text
DNS/GSLB -> region/site
Anycast  -> nearby route to shared IP
CDN/WAF  -> edge cache/protection
Edge compute -> nearby auth/light logic (optional)
L4 LB    -> backend connection
L7 gateway -> service route
```

## 10. IDC Data-Center Networking: From Large L2 to Spine-Leaf and SDN

IDC networking is not one protocol. It is an evolution from physical links and switches to routing fabrics and virtual-network control planes. The useful mental model is:

```text
traditional large L2: VLAN + STP + stacking
  -> transition: TRILL / RBridge / vendor L2 fabrics
  -> modern mainstream: L3 Clos / Spine-Leaf + ECMP
  -> cloud networks: VXLAN Overlay + EVPN/SDN control plane
```

Early designs tried to make many switches look like one large switch. Modern designs usually keep the physical network as a stable L3 underlay, then build the L2/L3 networks applications want on top through overlays.

### 10.1 Switch Control Plane and Forwarding Plane

Switches and routers are usually described as two planes:

| Plane | Job | Typical components |
|-------|-----|--------------------|
| **Control plane** | compute topology, protocols, and tables | CPU, network OS, BGP/OSPF/STP/LACP processes |
| **Forwarding / data plane** | forward packets at line rate from installed tables | ASIC, MAC table, FIB, ACL/TCAM |

The control plane is the brain that computes the map; the forwarding plane is the fast sorting machine that uses the map. Production traffic normally hits ASIC tables, not the device CPU per packet.

L2 switching primarily looks at MAC addresses:

```text
receive Ethernet frame -> lookup MAC table -> forward out learned port
unknown unicast / broadcast -> flood inside the VLAN
```

L3 switching adds routing in the switching ASIC:

```text
receive IP packet -> lookup FIB -> rewrite next-hop MAC -> line-rate forwarding
```

An SVI (Switched Virtual Interface) is commonly the gateway interface for a VLAN:

```text
VLAN 10: 192.168.10.0/24
SVI vlan10: 192.168.10.1  # default gateway for that VLAN
```

Moving from L2 to L3 is not just adding a small router. The key change is pushing gateways and routing down to access/leaf switches so east-west traffic no longer has to detour through distant aggregation/core layers.

### 10.2 Traditional Large L2: VLAN, STP, and Three-Tier Trees

Traditional data centers often used access -> aggregation -> core:

```text
server -> access / ToR switch -> aggregation switch -> core switch
```

Why this existed:

- Port density and switching silicon could not put every server directly on core devices.
- VLANs needed to stretch across racks for the same application or subnet.
- Early VM mobility expected IP addresses to remain stable after migration, pushing networks toward large L2 domains.

The L2 problem is loops. Ethernet frames have no TTL, so broadcast or unknown-unicast frames can circulate indefinitely and create a broadcast storm.

STP (Spanning Tree Protocol) computes a loop-free tree using BPDUs:

```text
1. elect a Root Bridge, often by operator-configured priority
2. each switch chooses its shortest path to the root
3. redundant links enter Blocking/Discarding state to break loops
```

STP does not understand architecture roles such as core, aggregation, or access. It only uses Bridge ID, path cost, and port roles. Core devices often become the root because engineers plan priorities that way.

Costs of STP:

- Many redundant links sit blocked instead of carrying traffic.
- Convergence can be slow; RSTP/MSTP improve this but keep the spanning-tree model.
- Larger L2 broadcast domains enlarge the blast radius of loops, ARP storms, and unknown-unicast flooding.

### 10.3 TRILL and RBridge: Transitional L2 Multipathing

TRILL (Transparent Interconnection of Lots of Links) tried to fix STP's main pain: keep L2 semantics, but avoid blocking all redundant links.

TRILL devices are called RBridges (Routing Bridges):

```text
to hosts: they look like L2 bridges forwarding Ethernet frames
inside the RBridge fabric: they act like routers using link-state shortest paths
```

Basic flow:

1. RBridges run IS-IS to learn topology.
2. The ingress RBridge encapsulates the original Ethernet frame with a TRILL header and marks the egress RBridge.
3. Intermediate RBridges forward by shortest path instead of an STP tree.
4. The TRILL header includes Hop Count, similar to TTL, to avoid infinite loops.
5. The egress RBridge decapsulates and delivers the original frame.

```text
original Ethernet frame
  -> ingress RBridge adds TRILL encapsulation
  -> RBridge fabric forwards along IS-IS shortest paths
  -> egress RBridge decapsulates
  -> destination host
```

| Dimension | STP large L2 | TRILL / RBridge |
|-----------|--------------|-----------------|
| redundant links | some blocked | multiple paths usable |
| control plane | spanning tree | IS-IS link state |
| forwarding | MAC learning on a tree | encapsulated shortest-path forwarding |
| loop protection | block links | Hop Count + link-state topology |
| goal | avoid L2 loops | keep L2 semantics and improve link utilization |

TRILL did not become the dominant cloud data-center architecture because cloud networks needed much larger tenant isolation, simpler L3 underlays, and an ecosystem that moved toward VXLAN/EVPN. TRILL is not a wrong idea; it is an important transition between STP-era L2 and modern L3-underlay/overlay designs.

### 10.4 Bonding, LACP, Stacking, and M-LAG

Server access starts with redundancy: survive a NIC, cable, or switch failure.

```text
server
  |-- NIC0 -> Leaf/Access Switch A
  |-- NIC1 -> Leaf/Access Switch B
```

Bonding is the Linux/server-side term; switch-side terms include LACP, Port-Channel, and Eth-Trunk.

| Mode | Purpose | Notes |
|------|---------|-------|
| active-backup | primary/standby failover | simplest and stable; little switch dependency |
| 802.3ad / LACP | aggregate links and hash flows | requires switch support; one flow usually stays on one member |
| balance-xor | fixed-policy hashing | verify switch compatibility |
| balance-alb/tlb | host-side load balancing | validate carefully in production |

Two 10G NICs in LACP do not make one TCP connection 20G. To avoid reordering, the same flow usually hashes to one member link; aggregate capacity appears across many flows.

Physical stacking makes multiple switches one logical switch. It simplifies management and cross-switch aggregation and historically reduced STP blocking, but it couples control planes tightly. Stack split, supervisor failure, version bugs, or upgrades can affect the whole stack.

M-LAG (also vPC/MC-LAG in vendor terminology) keeps switch control planes independent while coordinating the data plane:

```text
control plane: A and B each run their own OS, BGP, LACP, management processes
data plane: peer-link / keepalive synchronize required state and appear as one LACP peer to the server
```

| Dimension | Physical stacking | M-LAG |
|-----------|-------------------|-------|
| control plane | merged / tightly coupled | independent |
| data plane | looks like one switch | coordinated forwarding |
| blast radius | whole stack may be affected | one switch failure often costs partial bandwidth |
| upgrade | often higher risk | easier rolling maintenance |
| fit | small/medium simplicity | large IDC fault isolation |

Stacking is "merge into one." M-LAG is "separate brains, coordinated behavior toward the server." Modern IDCs often de-stack to reduce blast radius, vendor coupling, and upgrade risk.

### 10.5 Modern L3 Clos / Spine-Leaf and ECMP

A Clos network is a multi-stage topology; the common data-center form is Spine-Leaf:

```text
          Spine1   Spine2   Spine3   Spine4
             |       |       |       |
Leaf1 -------+-------+-------+-------+
Leaf2 -------+-------+-------+-------+
Leaf3 -------+-------+-------+-------+

servers attach to Leafs; every Leaf connects to every Spine
```

Leafs are usually ToR switches. Spines do not connect to servers; they provide equal paths between Leafs.

Modern Spine-Leaf is typically a pure L3 underlay:

```text
Leaf/Spine interfaces are L3 IP links
Leafs and Spines run BGP/OSPF/IS-IS
multiple equal next hops to a VTEP/prefix -> ECMP hashing
```

Why it replaces STP/stacking:

- IP packets have TTL, so routing failures do not loop forever like L2 broadcasts.
- Routing protocols include loop-prevention logic.
- ECMP uses parallel links actively instead of blocking them.
- A single Spine or Leaf failure normally reduces bandwidth or affects one rack, not an entire L2 domain.

Spine-Leaf does not require SDN. You can run a manually configured BGP/OSPF L3 fabric. SDN adds automation, tenant isolation, overlay mapping, policy orchestration, and lifecycle integration.

### 10.6 Underlay, Overlay, VXLAN, and VTEP

Underlay is the real physical network. Overlay is the virtual network built on top.

```text
Underlay: L3 IP Leaf/Spine fabric with BGP/OSPF + ECMP
Overlay : VXLAN-created virtual L2/L3 tenant networks
```

VXLAN encapsulates an original Ethernet frame inside UDP/IP:

```text
outer Ethernet
  outer IP: VTEP A physical/loopback IP -> VTEP B physical/loopback IP
    outer UDP: dst port 4789
      VXLAN header: VNI
        original Ethernet frame: VM A -> VM B
```

VXLAN uses UDP 4789 as a tunnel carrier, but the underlay is still L3. Intermediate spines route by outer destination IP; they do not care about tenant VM IP/MAC or VNI.

VTEP (VXLAN Tunnel Endpoint) encapsulates and decapsulates:

| Type | Location | Notes |
|------|----------|-------|
| hardware VTEP | Leaf switch | ASIC line-rate encapsulation/decapsulation |
| software VTEP | host / hypervisor / CNI | Open vSwitch, Linux tunnels, eBPF datapaths |

Flow example:

```text
VM A sends original Ethernet frame
  -> VTEP A looks up that VM B's MAC/IP is behind VTEP B
  -> encapsulate outer src = VTEP A IP, dst = VTEP B IP, UDP 4789, VNI = tenant network
  -> underlay routes by outer IP through Spine-Leaf + ECMP
  -> VTEP B decapsulates VXLAN
  -> original frame reaches VM B
```

VXLAN lets workloads behave as if they share an L2 network while the physical fabric remains routed L3. VNI expands isolation far beyond VLAN's 4096 limit.

### 10.7 EVPN-BGP: Overlay Control Plane

VXLAN is only data-plane encapsulation. The question is: how does VTEP A know which VTEP owns VM B?

Simple approaches are static mappings or flood-and-learn. At scale, both hurt: manual mapping is impossible and flooding brings large-L2 behavior back into the overlay.

EVPN-BGP publishes reachability:

```text
VTEP / controller publishes through BGP EVPN:
  which MAC/IP is behind which VTEP
  which VTEPs participate in a VNI
  which prefixes/gateways are reachable
```

| Type | Purpose |
|------|---------|
| Type 2 | MAC/IP -> VTEP mapping |
| Type 3 | VTEP membership for a VNI, controlling flood scope |
| Type 5 | IP prefix routes for more L3-style overlay forwarding |

EVPN-BGP moves "who is where" from data-plane flooding to control-plane advertisements. It also supports distributed gateways and anycast gateways, so a VM can keep the same default gateway IP/MAC after migration.

### 10.8 SDN, OpenFlow, and BGP

SDN (Software-Defined Networking) is an architecture: separate control from forwarding and orchestrate networks through software. It is not the same as overlay, and not the same as OpenFlow.

```text
cloud platform creates or migrates a VM/Pod
  -> SDN controller observes lifecycle
  -> computes networks, subnets, security groups, routes, MAC/IP/VNI/VTEP mappings
  -> programs switches, host OVS/eBPF datapaths, or cloud gateways
  -> datapath forwards locally from installed tables; packets do not ask the controller every hop
```

SDN can automate both underlay and overlay:

| Scope | Examples |
|-------|----------|
| Underlay automation | switch interface IPs, BGP neighbors, routing policy, link-state collection |
| Overlay orchestration | VNI, VTEP mappings, security groups, virtual routers, tenant isolation |

OpenFlow and BGP differ in philosophy:

| Dimension | OpenFlow | BGP / EVPN-BGP |
|-----------|----------|----------------|
| control model | centralized controller pushes flow rules | distributed peers exchange reachability |
| granularity | fine-grained match/action on MAC/IP/port/VLAN | route reachability; EVPN adds MAC/IP -> VTEP |
| device role | more executor-like | runs protocol and computes local paths |
| common use | early SDN, labs, fine policy control | Internet routing, IDC underlay, VXLAN EVPN |
| risk | strong controller dependency, scale/stability challenges | mature and distributed, but policy design is complex |

Common modern combination:

```text
Underlay: BGP/OSPF/IS-IS + ECMP for stable physical IP reachability
Overlay : VXLAN data plane + EVPN-BGP or SDN controller for mappings
Ops     : controller/automation handles lifecycle, policy, observability, config consistency
```

If the controller is briefly unavailable, existing datapath entries often continue forwarding; new VM, route, or security-policy changes may pause.

### 10.9 NFV, DPDK, SPDK, and Infrastructure Data-Plane Acceleration

#### 10.9.1 A Complete Cloud Infrastructure Data Path

The earlier sections explain how the physical fabric connects, routes, and carries tenant overlays. Cloud platforms must also answer: after a tenant creates a VM or Pod, how does traffic pass through virtual switching, routing, firewalling, NAT, and load balancing before reaching the workload?

```text
tenant creates VPC / subnet / VM
  -> cloud controller records network, route, security, and service-chain state
  -> SDN control plane programs Leafs, VTEPs, hosts, and gateways
  -> NFV instances provide vRouter / vFirewall / vNAT / vLB
  -> OVS / VPP / DPDK / eBPF execute the host or gateway datapath
  -> SmartNIC / DPU may offload switching, VXLAN, crypto, and security
  -> workload VM / Pod receives the packet
```

A useful analogy is a logistics park: **SDN is the dispatch center; NFV turns customs, inspection, and sorting centers into software; OVS is the internal road system; DPDK is the fast lane that reduces toll-booth stops; SmartNIC/DPU moves some sorting to the station entrance; NUMA determines whether goods, workers, and loading docks are in the same park.**

#### 10.9.2 What NFV Really Means: From Buying Boxes to Installing Network Apps

“Network Functions Virtualization” is still abstract. A more useful first sentence is: **NFV takes network functions that previously required dedicated hardware appliances and runs them as software, VMs, or containers on general-purpose servers.**

Without NFV, many capabilities require appliance purchases:

| Need | Traditional approach | Essential limitation |
|------|----------------------|----------------------|
| Firewall | buy Palo Alto, Fortinet, or similar hardware firewalls | function is tied to a box |
| Load balancing | buy F5 or similar hardware LBs | capacity comes in appliance-sized chunks |
| Gateway/routing | buy Cisco/Juniper routers | scaling, migration, and automation are slow |
| DPI/IDS/NAT | buy specialized appliances | each function becomes another box family |

The core shift is:

```text
before: network function = fixed capability inside a dedicated hardware box
after : network function = software service on general-purpose x86/ARM servers

Like installing apps:
server + KVM/container + vFirewall / vRouter / vNAT / vLB
```

This does not mean dedicated hardware disappears. It means the **function** is decoupled from the **appliance**: firewalls, routers, NAT gateways, and load balancers can be created, destroyed, scaled, and upgraded more like VMs or Pods.

#### 10.9.3 NFV Is Not Bound to One Network Layer

NFV is not only an access-layer idea and not only “a virtual firewall.” It can appear wherever traffic needs network processing:

| Location | Common VNF/CNF examples | Purpose |
|----------|-------------------------|---------|
| **edge/egress layer** | vFirewall, vRouter, vNAT, DDoS scrubbing | process traffic entering or leaving a data center or VPC |
| **service/application front door** | vLB, WAF, API Gateway | distribute traffic, enforce L7 policy, and protect applications |
| **carrier/access network** | vCPE, 5G Core UPF/AMF | virtualize branch CPE and mobile-core network functions |
| **internal tenant network** | virtual router, virtual security gateway | provide VPC, subnet, and cross-AZ networking capabilities |

In short: **NFV is the virtualization/cloud-native idea applied to networking, not a fixed network tier.** If a traffic segment needs filtering, routing, NAT, load balancing, inspection, or scrubbing, that function may be NFV-based.

#### 10.9.4 SDN + NFV: SDN Draws the Path, NFV Processes the Traffic

A practical split:

```text
SDN: decides where traffic goes; lays paths, tunnels, routes, and flow rules
NFV: decides what traffic passes through; filters, transforms, inspects, NATs, and load-balances
```

Suppose outside traffic must pass through a virtual firewall, then a virtual load balancer, then a business VM:

```text
outside traffic
  -> vFirewall (NFV: enforce security policy)
  -> vLB       (NFV: choose a backend)
  -> VM C      (application)

SDN controller:
  programs routes/tunnels/flows so packets traverse those nodes in that order
```

This is Service Function Chaining (SFC):

```text
[ outside traffic ]
    -> [ vFirewall ]
    -> [ vLB ]
    -> [ target VM/Pod ]
```

Analogy: NFV provides the inspection machines, customs counters, and sorting stations as creatable and scalable function nodes. SDN draws roads between them and steers traffic. Without SDN, NFV nodes can exist but traffic may not automatically traverse them in the intended order. Without NFV, SDN can forward traffic but cannot elastically provide firewall, NAT, LB, DPI, or similar services.

| Dimension | SDN | NFV |
|-----------|-----|-----|
| Focus | routing and forwarding: where should traffic go? | network services: how should traffic be filtered, NATed, or load-balanced? |
| Replaces or reshapes | manual control-plane configuration | dedicated firewall, LB, DPI, NAT, and router appliances |
| Common carrier | switches, Leaf/Spine fabric, VTEPs, controllers | general-purpose servers, VMs, containers, virtual gateways |
| Typical technologies | OpenFlow, BGP-EVPN, VXLAN, controllers | KVM, containers, OVS, DPDK, SR-IOV, SmartNIC/DPU |
| Result | traffic can be steered automatically | network functions can be deployed and scaled like software services |

#### 10.9.5 Example: A Tenant VM Reaches the Internet

Suppose `10.10.1.20` accesses `203.0.113.8:443`. The cloud must enforce security policy, route the private address, perform SNAT, and associate return traffic with the connection:

```text
VM
  -> virtio NIC / vhost
  -> host OVS or eBPF datapath
  -> VXLAN VTEP: tenant overlay to physical underlay
  -> vFirewall: five-tuple, state, and policy checks
  -> vRouter: tenant route lookup
  -> vNAT: private address:port -> public address:port
  -> Leaf/Spine underlay
  -> Internet
```

The return packet uses the NAT connection-tracking entry for reverse translation, passes the firewall state check, and returns through the correct VTEP to the VM. Here, NFV is the set of function nodes on the path, SDN is the control plane that decides and distributes path/state, and VXLAN/EVPN is the mechanism that keeps tenant networks reachable across the physical fabric.

#### 10.9.6 NFV Components: NFVI, VNF/CNF, and Service Chains

```text
tenant/cloud orchestration:
  create VPC, NAT, LB, and firewall policy

NFV orchestration:
  choose function instances, scale, order, and failure policy

VNF / CNF:
  vFirewall, vRouter, vNAT, vLB, UPF

NFVI:
  servers, NICs, virtualization, network, storage, and accelerators
```

- **NFVI** is the compute, network, storage, and accelerator infrastructure that hosts network functions.
- **VNF** normally runs as a VM or process and is useful for migrating mature network software.
- **CNF** uses containers, microservices, declarative configuration, and cloud-native lifecycle management; elastic operation is attractive, but stateful network functions are harder to redesign.
- **Service Function Chaining** sends traffic through functions in a required order, like a parcel passing inspection, customs, and sorting before dispatch.

#### 10.9.7 NUMA: Why a Fast CPU Can Still Produce a Slow VNF

NUMA (Non-Uniform Memory Access) means a multi-socket server does not provide one completely uniform memory pool:

```text
NUMA Node 0                         NUMA Node 1
CPU 0/1 + local memory              CPU 2/3 + local memory
NIC queue 0 -- local PCIe           NIC queue 1 -- local PCIe
       \________ interconnect _________/
```

Local memory normally has lower latency and higher bandwidth. A high-rate VNF repeatedly touches NIC queues, DPDK mbuf pools, NAT/connection tables, and firewall rules. If the NIC is attached to Node 0 but worker threads and packet buffers live on Node 1, CPU utilization may look healthy while packet throughput collapses. It is like putting a warehouse, workers, and loading dock in separate parks.

Typical alignment:

```text
NIC PCIe on Node 0
  -> pin DPDK PMD threads to Node 0 CPUs
  -> allocate mbufs/hugepages from Node 0
  -> place related vCPUs, OVS/vhost threads, and state tables on Node 0
```

NUMA is not NFV-specific, but high packet rates and poll-mode data planes amplify remote-memory penalties. Normal web services may see jitter; a high-PPS firewall may see queue buildup and packet loss.

#### 10.9.8 DPDK, OVS, VPP, eBPF/XDP, and SR-IOV

A compatible VM path is:

```text
VM application -> guest TCP/IP -> virtio-net -> vhost boundary
  -> OVS/Linux bridge -> physical NIC
```

It preserves ordinary sockets and migration behavior, but packets may incur copies, virtualization transitions, interrupts, and kernel processing.

Common optimizations:

1. **OVS** connects VMs, Pods, and NICs and handles VLAN, VXLAN, ACL, and virtual-port forwarding.
2. **vhost-net/vhost-user** reduces overhead at the virtio boundary.
3. **DPDK** uses poll-mode drivers, preallocated mbuf pools, and batching to reduce interrupts, syscalls, and kernel-stack work.
4. **VPP** processes packet vectors in batches to reduce repeated per-packet instructions.
5. **eBPF/XDP** filters, forwards, samples, or load-balances early in the Linux path while retaining more kernel governance.
6. **SR-IOV** divides a physical NIC into virtual functions so a VM can access NIC queues more directly.

```text
normal: NIC -> interrupt/NAPI -> kernel stack -> socket
DPDK/VPP: NIC queue -> user-space PMD -> mbuf/vector -> network function
SR-IOV: NIC VF queue -> guest driver -> application
XDP: NIC/driver early path -> eBPF -> drop/forward/continue into kernel
```

These are not a simple performance ranking: DPDK trades general socket semantics and power for packet rate; SR-IOV trades virtual-switch visibility and migration flexibility for a shorter path; XDP trades some peak speed for easier kernel governance.

#### 10.9.9 SmartNIC and DPU: Moving Infrastructure Work Off the Host

When a host CPU runs business workloads while also processing virtual switching, VXLAN, crypto, storage, and policy, infrastructure work competes with application compute:

```text
VM/Pod
  -> DPU virtual switch / security policy
  -> DPU performs VXLAN, crypto, or routing offload
  -> physical NIC -> Leaf/Spine underlay
```

**SmartNIC** generally adds programmable processing, accelerators, or ARM cores to a NIC. A **DPU** emphasizes an independent CPU, memory, accelerator, and management boundary capable of running an infrastructure software stack.

```text
ordinary NIC: moves bits into the host
SmartNIC: a co-pilot for selected network tasks
DPU: a small infrastructure server beside the host
```

Typical offloads include virtual switching, VXLAN/Geneve, security groups, IPsec/TLS, NVMe-oF, and tenant isolation. The goal is not that every packet becomes faster; it is that host CPU capacity and latency become more predictable.

Offload introduces costs: host captures may no longer show the full path, hardware feature coverage and table sizes are limited, and the DPU needs its own firmware, drivers, control plane, upgrades, and state synchronization.

#### 10.9.10 The Unified Model

```text
cloud platform / Kubernetes / NFV orchestrator
  -> create VM, Pod, VPC, vFirewall, vNAT, vLB
  -> SDN computes routes, policies, VNI, VTEP, and service chains
  -> underlay: Leaf/Spine + BGP/ECMP
  -> overlay: VXLAN/EVPN
  -> host datapath: OVS / vSwitch / eBPF
  -> fast path: DPDK / VPP / SR-IOV
  -> locality: CPU pinning + NUMA
  -> hardware offload: SmartNIC / DPU
  -> VM/Pod / external network / storage
```

### 10.10 DCI and Cross-IDC L2 Stretch Boundaries

DCI (Data Center Interconnect) connects multiple data centers for disaster recovery, replication, service-to-service access, traffic steering, or a small amount of legacy L2 extension.

Be careful with cross-IDC L2 stretch:

| Risk | Explanation |
|------|-------------|
| latency | L2, storage, and cluster-heartbeat protocols may assume low RTT |
| broadcast domain | ARP, unknown unicast, and broadcast failures spread farther |
| MTU | VXLAN/MPLS/IPsec encapsulation consumes MTU and can create black holes |
| convergence | WAN flaps affect MAC/VTEP/route convergence |
| consistency | databases and storage do not become strongly consistent just because the network connects sites |

Recommended direction:

```text
prefer L7/API/message replication over forced L2 stretch
prefer object-storage replication or async sync over cross-city broadcast domains
if L2 stretch is unavoidable, narrow scope and monitor MTU, flooding, convergence, and isolation controls
```

---

## 11. Linux, Container Networking, and Security Boundaries

### 11.1 Linux Data Path, Containers, and Firewalls

```text
NIC -> driver/NAPI -> Ethernet/IP/TCP -> netfilter/conntrack
    -> socket queue -> epoll -> application
```

Containers commonly use a network namespace, veth pair, bridge/CNI, host routing, and optional overlay. Kubernetes Service commonly translates ClusterIP to Pod IP through iptables/IPVS/eBPF.

Security boundaries may stack: cloud security group -> nftables/iptables -> NetworkPolicy -> proxy/WAF -> application auth.

```bash
ip -s link
nstat
sar -n DEV,TCP,ETCP 1
ip netns list
nft list ruleset
iptables -t nat -L -n -v
```

---

## 12. QUIC and HTTP/3

### 12.1 Motivation

TCP has several pain points in modern networks:

| Pain Point | Explanation | Impact |
|------------|-------------|--------|
| **TCP head-of-line blocking** | One lost TCP segment blocks later bytes | Reduces HTTP/2 multiplexing benefit |
| **Handshake latency** | TCP handshake + TLS handshake | Slow on mobile and high-RTT networks |
| **Four-tuple binding** | Connection tied to src/dst IP and port | Wi-Fi to mobile switch breaks connection |

### 12.2 Core QUIC Features

**1-RTT and 0-RTT handshake**:

```text
First connection:
Client Initial: ClientHello + KeyShare -> Server
Server Initial: ServerHello + Cert     -> Client
Encrypted data can start after 1 RTT.

Resumed connection:
Client sends Initial + 0-RTT data using PSK.
```

0-RTT data has replay risk and should only be used for idempotent requests.

**Multiplexing without TCP-level HOL blocking**:

```text
HTTP/2 over TCP:
All streams share one ordered TCP byte stream.
Loss in one packet blocks all streams behind it.

HTTP/3 over QUIC:
Each QUIC stream has independent ordering.
Loss in one stream does not block other streams.
```

**Connection migration**:

```text
QUIC connection identity = Connection ID, not the four-tuple.

When mobile switches Wi-Fi -> 4G:
- TCP must reconnect.
- QUIC keeps the same Connection ID and validates the new path.
```

**Built-in TLS 1.3**:

- QUIC requires encryption.
- Handshake and transport negotiation are integrated.
- Many transport parameters are protected from middlebox modification.

### 12.3 QUIC vs TCP + TLS

| Dimension | TCP + TLS 1.3 | QUIC |
|-----------|---------------|------|
| **First handshake** | 2 RTT | 1 RTT |
| **Resumption** | 1-2 RTT | 0 RTT possible |
| **HOL blocking** | TCP-level HOL exists | Stream-level isolation |
| **Connection migration** | Not supported | Supported |
| **Congestion control** | Kernel implementation | User-space evolution is easier |
| **Compatibility** | Excellent | UDP may be blocked or rate limited |

### 12.4 Deployment Notes

```bash
# Nginx example
server {
    listen 443 quic reuseport;
    listen 443 ssl;
    http3 on;
    ssl_certificate     /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    add_header Alt-Svc 'h3=":443"; ma=86400';
}
```

Operational concerns:

- UDP 443 may be blocked by corporate networks or ISPs.
- Clients usually fall back to HTTP/2 through `Alt-Svc`.
- NAT mappings for UDP often expire faster than TCP mappings.
- Use tools such as `curl --http3`, browser network panels, Wireshark, qlog, and qvis for debugging.

---
## 13. TCP BBR Congestion Control

### 13.1 Problem with Loss-Based Congestion Control

Algorithms like Reno and CUBIC treat packet loss as congestion. In modern networks with large buffers, loss may only happen after queues are already full.

```text
Bufferbloat:
The sender keeps increasing rate until buffers fill.
RTT grows from propagation delay to propagation delay + queueing delay.
Users feel high latency even when bandwidth looks sufficient.
```

### 13.2 BBR Core Idea

BBR, Bottleneck Bandwidth and Round-trip propagation time, estimates two values:

```text
BtlBw: bottleneck bandwidth
  Estimated by delivered bytes / ACK interval.

RTprop: round-trip propagation time
  Estimated by the minimum RTT over a recent window.

Target inflight = BtlBw * RTprop
```

BBR tries to send at the bottleneck bandwidth while keeping inflight data near one BDP, maximizing throughput without persistently filling queues.

### 13.3 BBR States

```text
Startup -> Drain -> ProbeBW <-> ProbeRTT
```

- **Startup**: quickly probes bandwidth with aggressive growth.
- **Drain**: reduces sending rate to drain queues built during Startup.
- **ProbeBW**: steady-state bandwidth probing.
- **ProbeRTT**: briefly reduces inflight data to measure the real minimum RTT.

### 13.4 BBR vs CUBIC

| Dimension | CUBIC | BBR |
|-----------|-------|-----|
| **Congestion signal** | Loss | Bandwidth and RTT model |
| **Throughput under loss** | Drops significantly | More stable |
| **Latency** | Can fill buffers | Tries to control inflight data |
| **Fairness** | Fair with similar loss-based flows | BBRv1 may be unfair to CUBIC |
| **Best for** | General networks | High-latency, lossy, cross-region links |

Enable BBR on Linux:

```bash
uname -r
modprobe tcp_bbr
sysctl -w net.ipv4.tcp_congestion_control=bbr
sysctl net.ipv4.tcp_congestion_control

cat >> /etc/sysctl.conf << 'EOF'
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
EOF
sysctl -p
```

BBR works best with `fq`, which provides pacing support.

---
## 14. Network Troubleshooting

### 14.1 Common Commands

```bash
# Reachability and latency
ping example.com

# Route path
traceroute example.com

# Packet capture
tcpdump -i eth0 port 8080

# Connections
netstat -antp
ss -antp | grep ESTABLISHED

# Listen queues
ss -lnt

# TCP state summary
ss -s
```

### 14.2 Practical Troubleshooting Order

Use a fixed order to avoid guessing:

```text
1. DNS:
   Is the name resolved correctly? Is resolution slow or hijacked?

2. Connection establishment:
   Are SYN/SYN-ACK/ACK present? Is there SYN retransmission?

3. Transport quality:
   Are there retransmissions, duplicate ACKs, high RTT, or packet loss?

4. Application protocol:
   Is the request sent? When does the response leave the server?

5. Application runtime:
   Thread pool, connection pool, GC, locks, downstream dependencies.
```

For intermittent timeouts, compare packet timelines on both client and server. Packet capture is evidence; always correlate it with application logs, gateway metrics, kernel counters, and dependency metrics.

---
## 15. Game Networking Index

For deeper architecture and optimization details:

- [Game server architecture](../专项知识库/01-游戏后台专项.md#4-战斗系统设计)
- [Frame synchronization vs state synchronization](../专项知识库/01-游戏后台专项.md#41-帧同步-vs-状态同步)
- [Weak-network and scaling optimization](../专项知识库/01-游戏后台专项.md#6-性能优化与高峰应对)

---
## 16. Interview Self-Check

### Networking Troubleshooting Notes

- Do not only recite protocol definitions. Strong answers connect symptoms, layered diagnosis, packet evidence, and metrics.
- A useful troubleshooting sequence is DNS -> connection establishment -> transport quality -> application protocol semantics.
- Production issues are often not "cannot connect"; they are intermittent slowness, packet loss, retries, queueing, and tail latency spikes.

### Quick Questions

### Q1: How do the OSI seven-layer model and TCP/IP four-layer model map?

**Answer:** OSI is more fine-grained, while TCP/IP is closer to real implementations. OSI application, presentation, and session layers roughly map to TCP/IP application layer. Transport maps to transport. Network maps to internet. Data link and physical layers map to link.

### Q2: What is the real difference between GET and POST?

**Answer:** The key difference is semantics. GET retrieves resources and is safe and idempotent. POST submits data for processing or creation and is not inherently idempotent. The difference is not simply whether parameters can be passed in the URL or body.

### Q3: What do 301, 302, 307, and 308 mean?

**Answer:** 301 is permanent redirect. 302 is temporary redirect, and historically many clients changed the method to GET. 307 is temporary redirect while preserving the original method. 308 is permanent redirect while preserving the original method.

### Q4: What is the difference between Cookie, Session, and Token?

**Answer:** Cookie is a browser-side storage and transport mechanism. Session is server-side state. Token is an authentication credential. They can be combined, such as storing a session ID in a cookie or using a JWT token in an Authorization header.

### Q5: What is the difference between TCP and UDP?

**Answer:** TCP provides connection-oriented, reliable, ordered byte-stream delivery with flow control and congestion control. UDP is connectionless and best effort with lower overhead. The choice depends on whether reliability and ordering or low latency and real-time behavior matter more.

### Q6: What is the difference between HTTP Keep-Alive, TCP keepalive, and application heartbeat?

**Answer:** HTTP Keep-Alive is connection reuse at the HTTP layer. TCP keepalive detects dead peers at the kernel layer, usually with long intervals. Application heartbeat is business-level liveness detection and timeout control, often much faster and more semantically meaningful.

### Q7: Why do many RPC frameworks use HTTP/2 or gRPC?

**Answer:** HTTP/2 provides multiplexing and header compression over long-lived connections. gRPC adds Protobuf schema, efficient serialization, and strong interface contracts. This makes it suitable for service-to-service communication.

### Q8: How do 401 and 403 differ?

**Answer:** 401 means the request is not authenticated. 403 means the identity is known but not authorized to perform the action.

### Q9: What do 502, 503, and 504 usually mean at a gateway?

**Answer:** 502 means the upstream returned an invalid response. 503 means the service is unavailable or overloaded. 504 means the gateway timed out waiting for the upstream.

### Q9b: Rough stages when a browser opens an HTTPS URL?

**Answer:** DNS → local route/ARP → egress NAT → public path to CDN/LB/gateway → TCP handshake → TLS (cert verify + key agreement) → HTTP → gateway to app → response reverse path (NAT undo, TLS decrypt). Details: Q25 and §1.3.

### Deep-Dive Questions

### Q10: Explain each step of the TCP three-way handshake. Why not two or four?

**Answer:** SYN lets the server confirm the client's send ability. SYN-ACK lets the client confirm the server can receive and send. ACK lets the server confirm the client can receive. Two handshakes do not let the server know whether the client received the server's sequence number. Four are unnecessary because the server's SYN and ACK can be combined.

### Q11: What is SYN Flood? How does syncookie defend against it?

**Answer:** In SYN Flood, attackers send many SYN packets but never complete the final ACK, exhausting the half-open connection queue. Syncookie avoids storing SYN state immediately. It encodes connection information into a cookie in the SYN-ACK sequence number and validates it when the final ACK returns.

### Q12: Why does TIME_WAIT exist? How do you handle too many TIME_WAIT connections?

**Answer:** TIME_WAIT ensures the final ACK can be retransmitted and old packets cannot interfere with a new connection using the same four-tuple. Too many TIME_WAIT connections are usually caused by short connections. Use keep-alive, connection pools, `tcp_tw_reuse` for outbound clients, and a larger ephemeral port range if necessary.

### Q13: What does a large number of CLOSE_WAIT connections indicate?

**Answer:** It means the peer has closed the connection, but the local application has not called `close()`. This is an application bug, often caused by forgotten response body close or missing cleanup on error paths.

### Q14: What does the TCP sliding window do? What decides the send window?

**Answer:** Sliding window allows the sender to send multiple segments without waiting for every ACK, improving bandwidth utilization. The actual send window is `min(cwnd, rwnd)`, constrained by both congestion control and receiver flow control.

### Q15: What is zero window probe?

**Answer:** If the receiver advertises `rwnd=0`, the sender stops sending. If the later window update is lost, both sides may deadlock. Zero window probe sends tiny periodic probes so the receiver can report the current window.

### Q16: Why is slow start exponential and congestion avoidance linear?

**Answer:** Slow start grows exponentially to quickly discover available bandwidth. Once near the threshold, exponential growth becomes too aggressive, so congestion avoidance grows linearly to probe capacity more cautiously.

### Q17: What is the interaction problem between Nagle and delayed ACK?

**Answer:** Nagle may wait for ACK before sending another small packet, while delayed ACK waits before sending the ACK. Together they can add around hundreds of milliseconds of latency to small interactive messages. `TCP_NODELAY` disables Nagle for latency-sensitive applications.

### Q18: What is the difference between strong cache and validation cache in HTTP?

**Answer:** Strong cache uses local cache directly without sending a request, controlled by `Cache-Control` and `Expires`. Validation cache sends conditional requests with `ETag` or `Last-Modified`; unchanged resources return `304 Not Modified`.

### Q19: What are the main differences between HTTP/1.1, HTTP/2, and HTTP/3?

**Answer:** HTTP/1.1 has limited parallelism and head-of-line blocking. HTTP/2 adds multiplexing and HPACK but still suffers from TCP-level HOL blocking. HTTP/3 uses QUIC over UDP, avoids TCP-level HOL blocking, supports faster handshakes, and supports connection migration.

### Q20: What is BDP and why does it matter for high-latency networks?

**Answer:** BDP is bandwidth times RTT. It is the amount of data needed in flight to fully utilize the link. If socket buffers are smaller than BDP, the sender cannot fill the pipe, so bandwidth utilization remains low even when nominal bandwidth is high.

### Q21: How do L4 and L7 load balancers differ?

**Answer:** L4 load balancing routes by IP and port, with low overhead and high performance. L7 load balancing understands HTTP semantics such as host, path, headers, and cookies, enabling routing, canary release, auth, and observability, but with more processing cost.

### Q22: Why does "high bandwidth" not necessarily mean "fast requests"?

**Answer:** Bandwidth is capacity; RTT, queueing, retransmission, and application processing determine latency. Tail latency can remain high even with large bandwidth if the path has high RTT, packet loss, bufferbloat, or overloaded dependencies.

### Q23: How do you use packet capture to distinguish retransmission, reordering, and slow application processing?

**Answer:** Capture packets on both client and server. Look for duplicate ACKs and retransmissions to identify loss. Compare request arrival time and response send time on the server to identify application delay. Compare both timelines to locate whether the delay is before sending, in the network, or inside server processing.

### Q24: When is HTTP/3 clearly better than HTTP/2, and when is the benefit limited?

**Answer:** HTTP/3 helps most on lossy, high-RTT, mobile, or network-switching scenarios because it avoids TCP-level HOL blocking and supports connection migration. On low-loss, low-RTT internal networks, HTTP/2 may already be sufficient and HTTP/3 benefits may be limited.

### Q25: Walk through a full HTTPS request from client click to response (as detailed as possible).

**Answer:** Take `https://api.example.com/users`. Answer in time order and by layer; stress that the resolved IP is usually an **entry**, not the final Pod.

**0) Application intent**
Client prepares `GET /users`, `Host`, auth headers. For HTTPS it must finish DNS → TCP → TLS before sending HTTP.

**1) DNS**
Browser/OS/`hosts` cache → recursive resolver → root/TLD/authoritative as needed → `A/AAAA` (often CDN/GSLB/Anycast entry).

**2) Local routing + L2**
Off-subnet → default gateway; ARP/NDP for next-hop MAC; pick ephemeral port → four-tuple
`(10.0.0.8, 52341, 203.0.113.10, 443)`. Destination IP stays the entry; MAC is only this hop.

**3) Egress NAT / firewall**
SNAT `10.0.0.8:52341` → `198.51.100.8:40001`; conntrack for reverse path; stateful firewall allows the flow.

**4) Public forwarding**
Longest-prefix routing, TTL--, L2 rewrite each hop; Anycast may land at nearest POP → `203.0.113.10:443`.

**5) Entry chain**
`CDN/WAF → L4 LB → L7 gateway → app`. This is the component path, not the completed protocol order.
L4 distributes by connection and may either pass through TCP or proxy it with a new backend TCP connection. The L7 gateway works after it has a TCP byte stream, then may terminate TLS, and only after TLS/HTTP is parseable can it route by `Host`/path and set `X-Forwarded-For`.

**6) TCP handshake**
`SYN → SYN-ACK → ACK` → `ESTABLISHED` (MSS/window options). Pipe is up; not yet encrypted or business.

**7) TLS 1.3**
`ClientHello` (versions, ciphers, **SNI**, **key_share**, **ALPN**) → `ServerHello` + key_share + encrypted `Certificate` / `CertificateVerify` / `Finished` → verify chain, SAN, validity, prove private key → derive AEAD traffic secrets (~1-RTT). TLS 1.2 is typically ~2-RTT. If the gateway terminates TLS, cert validation is client↔gateway.

**8) HTTP**
Send HTTP/1.1 text or HTTP/2 frames inside TLS records → gateway routes → app auth/DB/cache → response. Stack: `HTTP → TLS Record → TCP → IP → frame`. HTTP/3 uses QUIC over UDP with its own TLS handshake and ALPN; it is not an upgrade of an already-established TCP connection.

**9) Return + teardown**
NAT reverses port mapping → TLS decrypt → UI. Short connection: FIN / `TIME_WAIT`; Keep-Alive / HTTP/2: reuse.

**Interview bonus:** order of stages; entry vs backend IP; TCP vs TLS vs HTTP; name SNI/ALPN/SNAT/L4 vs L7; map failures (bad DNS / SYN drop / cert error / 502·504) to layers. Full narrative: §1.3.

### Q26: After a DNS record changes, can clients do anything besides waiting for cache expiry?

**Answer:** For the global user base, mostly wait. Authoritative DNS has no standard way to flush every ISP recursive cache. Locally, a client can flush browser/OS caches, temporarily switch resolvers, use `hosts`, or invalidate app-managed HTTPDNS caches — but clearing the local cache does not force the recursive resolver to drop its old answer. Production cutovers should lower TTL first, keep old and new endpoints serving through the previous TTL window, then raise TTL. See §8.3.

### Q27: Is RTMP still common, and where do video protocols fit in the live pipeline?

**Answer:** RTMP is still common for ingest compatibility, especially OBS and hardware encoders pushing to a live origin. But modern video delivery should be described as a pipeline: RTMP/SRT/RIST ingest, transcoding and repackaging into multiple bitrates, HLS/DASH/LL-HLS delivery through CDN, and WebRTC/RTP for sub-second interaction. UDP-based media stacks are popular for real-time paths because late frames may be less valuable than skipped frames, and the application can combine FEC, selective retransmission, jitter buffers, congestion control, and adaptive bitrate without TCP head-of-line blocking.

### Q28: How do you diagnose whether live-streaming stalls come from the user, CDN, or origin?

**Answer:** First scope the blast radius. One user: Wi-Fi/mobile/device/player. One region/ISP: CDN edge, ISP peering, or traffic steering. Everyone: source signal, ingest, transcode, origin, CDN core, or backbone. Hot moments often expose flash crowds, hot segment misses, origin-shield pressure, or edge bandwidth exhaustion. CDN is usually mandatory for large events; the hard trade-off is low latency vs playback buffer.

### Q29: CDN vs edge computing, and should public CDN solve private cross-IDC sync?

**Answer:** CDN moves content closer; edge computing moves logic closer. Public CDN is for public user delivery and should not normally move private IDC data through public third-party edges. Private cross-DC distribution should use private CDN/cache nodes, object-storage replication, async sync, private links, private load balancing, or cloud interconnect.

### Q30: What do bonding, LACP, stacking, and M-LAG each solve?

**Answer:** Bonding is the server-side multi-NIC abstraction; LACP/Port-Channel/Eth-Trunk is the switch-side link aggregation. They provide link redundancy and aggregate throughput across many flows, but one TCP flow usually stays on one member link. Stacking merges multiple switches into one logical device, simplifying management but enlarging the failure domain. M-LAG keeps switch control planes independent while coordinating the data plane so a dual-homed server sees one LACP peer.

### Q31: How do traditional large L2, STP, TRILL/RBridge, and modern L3 Spine-Leaf relate?

**Answer:** Traditional large L2 stretches VLAN broadcast domains and uses STP to block redundant links, trading safety for wasted capacity and large blast radius. TRILL/RBridge is a transition: it keeps L2 semantics while using IS-IS shortest paths and Hop Count for L2 multipathing. Modern cloud data centers further decouple the system: the physical underlay is L3 Spine-Leaf with BGP/OSPF/IS-IS + ECMP, and tenant L2/L3 networks are provided by VXLAN/EVPN overlay.

### Q32: How do Spine-Leaf, Underlay/Overlay, VXLAN, VTEP, and EVPN relate?

**Answer:** Spine-Leaf is the physical topology. Underlay is the real L3 IP fabric. Overlay is the tenant network built on top. VXLAN is the data-plane encapsulation. VTEP encapsulates/decapsulates VXLAN. EVPN-BGP publishes MAC/IP/VNI/VTEP reachability to reduce flooding.

```text
Underlay: L3 Leaf/Spine IP network with BGP/OSPF + ECMP
VXLAN   : encapsulates original Ethernet frames into UDP/IP; outer destination is target VTEP
VTEP    : hardware Leaf or host software datapath that encapsulates/decapsulates
EVPN    : advertises MAC/IP/VNI/VTEP mappings
```

VXLAN uses UDP 4789 as the tunnel carrier, but the underlay remains L3. Intermediate spines route by outer IP, not by tenant VM MAC/IP.

### Q33: How do OpenFlow and BGP differ in SDN/data-center networking?

**Answer:** OpenFlow is a centralized match/action flow-table model where the switch is more executor-like. BGP is a distributed reachability protocol where devices exchange routes and compute local paths. Modern large data centers more often use BGP/EVPN-BGP for underlay and overlay reachability, with SDN controllers orchestrating lifecycle and policy rather than sitting in every packet's forwarding path.

### Q34: What are NFV, DPDK, SPDK, SmartNIC/DPU, OVS, VPP, eBPF, and RDMA?

**Answer:**

- **NFV** is an architecture/deployment model: firewalls, NAT, load balancers, and vRouters become software running on commodity servers, VMs, or containers.
- **VNF** is a VM/process-based virtual network function; **CNF** is a further containerized, cloud-native network function.
- **DPDK** accelerates network I/O through user-space PMDs, mempools, polling, and CPU/NUMA optimization.
- **SPDK** accelerates storage I/O through user-space NVMe drivers and asynchronous polling, reducing kernel storage-stack overhead.
- **SmartNIC/DPU** offloads virtual switching, VXLAN, crypto, storage, or security work to the NIC or an infrastructure processor.
- **OVS** is the host software switch; **VPP** is a high-performance user-space vectorized forwarding framework; **eBPF/XDP** is a programmable early Linux data path.
- **SR-IOV** gives VMs/containers more direct access to NIC virtual functions; **RDMA/RoCE** provides low-CPU remote memory/storage communication; **P4** describes programmable switch data-plane behavior.

They are different layers:

```text
SDN: controls and orchestrates network state
NFV: software/virtualized deployment model for network functions
VNF/CNF: concrete software network functions
DPDK: network I/O data-plane acceleration
SPDK: storage I/O data-plane acceleration
SmartNIC/DPU: hardware or independent-processor infrastructure offload
```

A typical design may use SDN/NFV to orchestrate a virtual firewall or vRouter, OVS/VPP plus DPDK to accelerate its packet path, and VXLAN/EVPN for network reachability. A storage service may use SPDK or RDMA/RoCE, while selected network, storage, and security functions are further offloaded to a SmartNIC/DPU. CPU pinning, NUMA, memory, NIC queues, firmware compatibility, lossless-network configuration, and observability then become part of the architecture rather than implementation details.

### Q35: What do private circuits, VPN, GRE, MPLS, VXLAN, and SD-WAN each solve?

**Answer:**

- A **private circuit** is managed WAN transport with path isolation/SLA expectations; it is not automatically end-to-end encrypted.
- A **VPN** creates a logical tunnel over an existing IP network; IPsec provides authentication, integrity, and encryption, but the public underlay still affects quality.
- **GRE** carries inner protocols or dynamic routing but does not encrypt; it is often combined with IPsec.
- **MPLS L3VPN** uses provider VRFs/labels to isolate customer Layer-3 WANs; isolation is not automatically end-to-end encryption.
- **VXLAN** is primarily a tenant overlay inside an IDC/cloud, not a public Internet security VPN.
- **SD-WAN** controls and orchestrates multiple WAN underlays such as circuits, Internet, and 4G/5G, selecting paths by policy and quality.

Production designs also need dual CPEs/gateways, BGP/BFD, return routes, session state, MTU/MSS validation, and a tested backup path. A VPN client showing `Connected` proves tunnel establishment, not that business routing and firewall policy are correct.

### Q36: If DH can establish a shared symmetric key without sending private keys, why does HTTPS still use expensive asymmetric cryptography?

**Answer:** This mixes three concepts:

1. **DH/ECDHE is itself a public-key cryptographic key-agreement method**, not a symmetric encryption algorithm. It helps derive a shared secret during the handshake; it does not continuously encrypt HTTP data.
2. HTTPS does **not** use RSA, ECDSA, or ECDHE to encrypt every HTTP packet. After the handshake, application data uses fast symmetric AEAD such as AES-GCM or ChaCha20-Poly1305.
3. Certificates and asymmetric signatures are still needed because DH only proves that both sides can derive the same secret; it does not prove who the peer is. The certificate chain and `CertificateVerify` prevent a man-in-the-middle from creating two separate DH exchanges.

Simplified TLS 1.3:

```text
ECDHE: derive an ephemeral shared secret
certificate/signature: prove the server owns the trusted certificate private key
HKDF: derive handshake and application traffic secrets
AES-GCM/ChaCha20-Poly1305: efficiently encrypt all subsequent HTTP data
```

The accurate statement is therefore: **HTTPS uses public-key cryptography for authentication and key agreement, then symmetric cryptography for bulk data.** The expensive public-key work is limited to the handshake and rekey events, buying authenticated server identity and forward secrecy.

### Open-Ended Design Questions

### D1: Design a globally deployed real-time communication system. What should the network layer consider?

**Reference approach:**

- Use UDP/RTP for real-time media and WebSocket or gRPC for signaling.
- Consider QUIC for reliable low-latency transport where appropriate.
- Use global traffic steering: Anycast, GSLB, nearby POPs, and backbone routing.
- Handle weak networks with FEC, jitter buffers, adaptive bitrate, and path switching.
- Support NAT traversal with STUN/TURN/ICE.
- Collect RTT, packet loss, jitter, bitrate, and reconnect metrics in real time.

### D2: TCP connections suddenly jump from 10,000 to 500,000 and the service becomes unavailable. How do you respond?

**Reference approach:**

- First stop the bleeding with connection limits, rate limits, or removing overloaded nodes from the load balancer.
- Use `ss -s` to inspect state distribution.
- Many `CLOSE_WAIT` entries indicate application connection leak.
- Many `TIME_WAIT` entries indicate short connections or active close pressure.
- Use pprof, jstack, logs, and connection pool metrics to locate leaks.
- Long-term fixes: connection pooling, proper timeouts, keep-alive, queue/backlog monitoring, and alerts.

---

## Summary

Backend and operations engineers should connect the full path:

1. **Addressing and delivery**: frames, IP, CIDR, ARP/NDP, routing, MTU, ports, NAT, and the underlay/overlay boundary of private circuits and VPNs.
2. **Transport**: TCP lifecycle, flow/congestion control, socket queues, and UDP trade-offs.
3. **Application protocols**: HTTP semantics/caching, TLS/PKI, real-time communication, and DNS.
4. **Traffic entry**: GSLB, Anycast, CDN, edge computing, L4 load balancers, and L7 gateways.
5. **Data-center networking**: bonding, stacking, M-LAG, Spine-Leaf, ECMP, VXLAN, EVPN, SDN, NFV, and high-performance data planes.
6. **Runtime path**: Linux network stack, conntrack, firewalls, container networking, and host data paths.
7. **Troubleshooting**: build evidence from DNS -> route -> connection -> TLS -> HTTP/RPC -> application.
