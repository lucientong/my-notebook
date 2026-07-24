# Networking Fundamentals and Protocols

Language: English | [中文](../后端知识库/06-网络基础与协议.md)

---

## Table of Contents

### Protocol Fundamentals
1. [OSI Model and TCP/IP](#1-osi-model-and-tcpip)
2. [TCP In Depth](#2-tcp-in-depth)
3. [UDP and Game Networking](#3-udp-and-game-networking)
4. [HTTP](#4-http)
5. [HTTPS and TLS](#5-https-and-tls)

### Connectivity and Special Topics
6. [WebSocket and Real-Time Communication](#6-websocket-and-real-time-communication)
7. [Game Networking Notes](#7-game-networking-notes)
8. [DNS and Domain Resolution](#8-dns-and-domain-resolution)

### Advanced Topics
9. [QUIC and HTTP/3](#9-quic-and-http3)
10. [TCP BBR Congestion Control](#10-tcp-bbr-congestion-control)

### Troubleshooting and Interview Review
11. [Network Troubleshooting](#11-network-troubleshooting)
12. [Interview Self-Check](#12-interview-self-check)

---

## 1. OSI Model and TCP/IP

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

---

## 2. TCP In Depth

### 2.1 TCP vs UDP

| Dimension | TCP | UDP |
|-----------|-----|-----|
| **Connection** | Connection-oriented, three-way handshake | Connectionless |
| **Reliability** | Reliable, ACK and retransmission | Best effort |
| **Ordering** | Ordered byte stream | No ordering guarantee |
| **Speed** | Higher overhead | Lower overhead |
| **Header size** | 20 bytes minimum | 8 bytes |
| **Use cases** | HTTP, FTP, email, RPC | DNS, video, games, QUIC |

TCP optimizes reliable ordered delivery. UDP optimizes simplicity and low latency. Protocol selection should start from business requirements, not from "which one is faster".

### 2.2 TCP Three-Way Handshake

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

### 2.2.1 Kernel-Level Handshake Notes

**SYN Flood and syncookies**:

```bash
# Half-open connection queue limit
cat /proc/sys/net/ipv4/tcp_max_syn_backlog

# Syncookie switch
cat /proc/sys/net/ipv4/tcp_syncookies
# 0=off, 1=enable when SYN queue is exhausted, 2=always use cookies
```

With syncookies, the server does not store SYN state immediately. It encodes the four-tuple and timestamp into a cookie and puts it into the sequence number of the SYN-ACK. The client returns the cookie in the final ACK. If validation succeeds, the server establishes the connection. This converts memory exhaustion into CPU verification work.

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

### 2.3 TCP Four-Way Termination

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

### 2.3.1 Why TIME_WAIT Is 2MSL

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

### 2.3.2 Too Many TIME_WAIT Connections

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

### 2.3.3 CLOSE_WAIT and Other Closing States

| State | Why It Stays | Timeout | Fix |
|-------|--------------|---------|-----|
| **FIN_WAIT1** | Peer did not ACK FIN | TCP retransmission timer | Tune orphan retries only if necessary |
| **FIN_WAIT2** | Peer did not send FIN | `tcp_fin_timeout` | Reduce timeout if needed |
| **CLOSE_WAIT** | Local application did not close socket | No kernel timeout | Fix application code |
| **LAST_ACK** | Peer did not ACK final FIN | TCP retransmission timer | Same as FIN_WAIT1 |
| **TIME_WAIT** | Wait for old packets to disappear | Fixed wait in Linux | Reuse, pooling, reduce short connections |

If you see many `CLOSE_WAIT`, it is almost always an application bug, such as not closing response bodies or connections on error paths.

### 2.4 Sliding Window and Flow Control ⭐⭐⭐

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

### 2.4.1 Flow Control vs Congestion Control

| Dimension | Flow Control | Congestion Control |
|-----------|--------------|--------------------|
| **Problem** | Do not overwhelm the receiver | Do not overwhelm the network |
| **Variable** | `rwnd` | `cwnd` |
| **Decided by** | Receiver | Sender |
| **Signal** | TCP Window field | Loss, duplicate ACKs, RTT, bandwidth signals |
| **Relationship** | Actual send window = min(cwnd, rwnd) | Works together with flow control |

### 2.4.2 Nagle Algorithm and Delayed ACK

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

### 2.5 TCP Congestion Control

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

### 2.6 TCP Packet Sticking and Splitting

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

### 2.7 Socket Buffers and TCP Performance

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

## 3. UDP and Game Networking

### 3.1 UDP Characteristics

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

### 3.2 Why Games Often Use UDP

Real-time games prefer recent state over guaranteed delivery of old state.

| Scenario | Protocol | Reason |
|----------|----------|--------|
| Login, payment | TCP | Must be reliable |
| Chat | TCP or reliable channel | Must be reliable |
| Movement, attack input | UDP | Real-time priority |
| Match result | TCP or reliable channel | Must be reliable |

For position updates, retransmitting stale packets can make the game feel worse. It is often better to drop old state and use the latest update.

### 3.3 Reliable Transport over UDP

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

## 4. HTTP

### 4.1 Request and Response

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

### 4.2 HTTP Methods

| Method | Meaning | Idempotent? | Safe? |
|--------|---------|-------------|-------|
| **GET** | Retrieve resource | Yes | Yes |
| **POST** | Create or submit for processing | No | No |
| **PUT** | Full replacement | Yes | No |
| **PATCH** | Partial update | Usually no | No |
| **DELETE** | Delete resource | Yes | No |

The primary difference between methods is semantics, not whether they can carry parameters.

### 4.3 HTTP/1.1 vs HTTP/2 vs HTTP/3

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| **Transport** | TCP | TCP | QUIC over UDP |
| **Multiplexing** | No | Yes | Yes |
| **Header compression** | No | HPACK | QPACK |
| **Head-of-line blocking** | Severe | Still exists at TCP layer | Solved at QUIC stream level |

HTTP/2 improves connection reuse and multiplexing, but TCP-level packet loss can still block all streams. HTTP/3 moves multiplexing into QUIC so loss on one stream does not block unrelated streams.

### 4.4 HTTP Caching ⭐⭐⭐

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

---

## 5. HTTPS and TLS

HTTPS is HTTP over TLS. It provides confidentiality, integrity, and server authentication.

### 5.1 TLS 1.3 Handshake

```text
Client                                 Server
  |  ClientHello + KeyShare             |
  |------------------------------------>|
  |                                     |
  |  ServerHello + KeyShare + Cert      |
  |<------------------------------------|
  |                                     |
  |  Encrypted application data          |
```

Compared with TLS 1.2, TLS 1.3:

- Reduces handshake latency to 1 RTT.
- Removes many legacy insecure algorithms.
- Supports 0-RTT resumption, with replay risk.

0-RTT should only be used for idempotent requests because early data can be replayed.

---

## 6. WebSocket and Real-Time Communication

### 6.1 WebSocket Handshake

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

### 6.2 Go Example

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

### 6.3 Heartbeat

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

---

## 7. Game Networking Notes

For deeper architecture and optimization details, see the Chinese game backend guide:

- [Game server architecture](../专项知识库/01-游戏后台专项.md#4-战斗系统设计)
- [Frame synchronization vs state synchronization](../专项知识库/01-游戏后台专项.md#41-帧同步-vs-状态同步)
- [Weak network optimization](../专项知识库/01-游戏后台专项.md#6-性能优化与高峰应对)

Key concepts:

- Frame sync is common in RTS/MOBA-like deterministic simulations.
- State sync is common in FPS/MMO-like systems where the server owns authoritative state.
- Client prediction, interpolation, lag compensation, jitter buffer, and AOI are practical tools for perceived smoothness.

---

## 8. DNS and Domain Resolution

### 8.1 Resolution Flow

```text
Browser -> Browser cache -> OS cache -> Local DNS
        -> Root DNS -> TLD DNS -> Authoritative DNS
```

### 8.2 DNS Record Types

| Type | Meaning | Example |
|------|---------|---------|
| **A** | IPv4 address | example.com -> 192.168.1.1 |
| **AAAA** | IPv6 address | example.com -> 2001:db8::1 |
| **CNAME** | Alias | www.example.com -> example.com |
| **MX** | Mail server | example.com -> mail.example.com |

DNS optimization:

- Use appropriate TTLs.
- Use local DNS cache.
- Reduce unnecessary domain lookups.
- Use multiple authoritative DNS providers for resilience.
- Consider HTTPDNS in environments with ISP DNS hijacking.

---

## 9. QUIC and HTTP/3

### 9.1 Motivation

TCP has several pain points in modern networks:

| Pain Point | Explanation | Impact |
|------------|-------------|--------|
| **TCP head-of-line blocking** | One lost TCP segment blocks later bytes | Reduces HTTP/2 multiplexing benefit |
| **Handshake latency** | TCP handshake + TLS handshake | Slow on mobile and high-RTT networks |
| **Four-tuple binding** | Connection tied to src/dst IP and port | Wi-Fi to mobile switch breaks connection |

### 9.2 Core QUIC Features

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

### 9.3 QUIC vs TCP + TLS

| Dimension | TCP + TLS 1.3 | QUIC |
|-----------|---------------|------|
| **First handshake** | 2 RTT | 1 RTT |
| **Resumption** | 1-2 RTT | 0 RTT possible |
| **HOL blocking** | TCP-level HOL exists | Stream-level isolation |
| **Connection migration** | Not supported | Supported |
| **Congestion control** | Kernel implementation | User-space evolution is easier |
| **Compatibility** | Excellent | UDP may be blocked or rate limited |

### 9.4 Deployment Notes

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

## 10. TCP BBR Congestion Control

### 10.1 Problem with Loss-Based Congestion Control

Algorithms like Reno and CUBIC treat packet loss as congestion. In modern networks with large buffers, loss may only happen after queues are already full.

```text
Bufferbloat:
The sender keeps increasing rate until buffers fill.
RTT grows from propagation delay to propagation delay + queueing delay.
Users feel high latency even when bandwidth looks sufficient.
```

### 10.2 BBR Core Idea

BBR, Bottleneck Bandwidth and Round-trip propagation time, estimates two values:

```text
BtlBw: bottleneck bandwidth
  Estimated by delivered bytes / ACK interval.

RTprop: round-trip propagation time
  Estimated by the minimum RTT over a recent window.

Target inflight = BtlBw * RTprop
```

BBR tries to send at the bottleneck bandwidth while keeping inflight data near one BDP, maximizing throughput without persistently filling queues.

### 10.3 BBR States

```text
Startup -> Drain -> ProbeBW <-> ProbeRTT
```

- **Startup**: quickly probes bandwidth with aggressive growth.
- **Drain**: reduces sending rate to drain queues built during Startup.
- **ProbeBW**: steady-state bandwidth probing.
- **ProbeRTT**: briefly reduces inflight data to measure the real minimum RTT.

### 10.4 BBR vs CUBIC

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

## 11. Network Troubleshooting

### 11.1 Common Commands

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

### 11.2 Practical Troubleshooting Order

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

## Summary

Core networking topics for backend interviews:

1. **TCP/IP model**: understand what each layer is responsible for.
2. **TCP reliability**: handshake, termination, TIME_WAIT, sliding window, retransmission, congestion control.
3. **HTTP**: methods, status codes, keep-alive, caching, HTTP/2, HTTP/3.
4. **HTTPS/TLS**: certificate chain, key exchange, TLS 1.3 handshake.
5. **WebSocket**: full-duplex communication and heartbeat.
6. **DNS**: resolution path, TTL, record types, and performance.
7. **Troubleshooting**: DNS -> connection -> transport -> protocol -> application.

---

## 12. Interview Self-Check

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
