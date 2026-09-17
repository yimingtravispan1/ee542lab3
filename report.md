# EE 542 Lab 3 Report

## Part 1 — AWS Network Performance Experiments

### 1. AWS Network Topology

#### 1.1 VPC and Subnets

| Purpose | Subnet |
|---|---|
| Client experimental network | `10.0.1.0/24` |
| Server experimental network | `10.0.2.0/24` |
| Client SSH/EIP network | `10.0.3.0/24` |
| Server SSH/EIP network | `10.0.4.0/24` |

#### 1.2 EC2 Instances

| Node | Interface | Private IP | Purpose |
|---|---|---:|---|
| Client | experimental interface | `10.0.1.91` | Client–VyOS traffic |
| Client | SSH interface | `10.0.3.39` | Elastic IP / SSH |
| VyOS | eth0 | `10.0.1.173` | Client-side routing |
| VyOS | eth1 | `10.0.2.221` | Server-side routing |
| Server | experimental interface | `10.0.2.177` | Server–VyOS traffic |
| Server | SSH interface | `10.0.4.136` | Elastic IP / SSH |

---

### 2. AWS Architecture and Concepts

A VPC is an isolated virtual network, and subnets divide its address space into separate network segments. An Internet Gateway connects the VPC to the Internet. Route tables determine where subnet traffic is forwarded. Security groups are stateful filters for instances and ENIs, while Network ACLs are stateless filters applied to subnets. An ENI is a virtual network interface with private IP addresses and security groups. An Elastic IP is a public address mapped by AWS to a private IP on an ENI. The Client and Server used separate NICs for experimental traffic and SSH/Internet access. AWS source/destination checking requires an instance to be the source or destination of traffic passing through its ENI. It was disabled on VyOS because the router forwards packets between other hosts.

---

### 3. Routing Configuration

Client traffic to the Server network was routed through VyOS:

```bash
sudo ip route add 10.0.2.0/24 via 10.0.1.173 dev ens5
```

Server traffic to the Client network was routed through VyOS.

```bash
sudo ip route add 10.0.1.0/24 via 10.0.2.221 dev ens5
```

---

### 4. Connectivity Verification

Client-to-Server connectivity was tested using:

```bash
ping 10.0.2.177
```

Traffic forwarding through VyOS was verified with:

```bash
sudo tcpdump -ni eth0 icmp
sudo tcpdump -ni eth1 icmp
```

![Client-to-Server ping result](images/part1/client-server-ping.png)

![ICMP traffic captured on VyOS eth0](images/part1/vyos-eth0-tcpdump.png)

![ICMP traffic captured on VyOS eth1](images/part1/vyos-eth1-tcpdump.png)

**Result:** 13/13 pings succeeded (0% loss); RTT min/avg/max = 0.410/0.498/1.092 ms. Both VyOS interfaces captured the ICMP requests and replies, confirming forwarding.

---

### 5. Baseline Performance

Server:

```bash
iperf3 -s
```

Client UDP test:

```bash
iperf3 -u -c 10.0.2.177 -b 500M
```

Client TCP test:

```bash
iperf3 -c 10.0.2.177
```

![UDP baseline result](images/part1/baseline-udp.png)

![TCP baseline result](images/part1/baseline-tcp.png)

| Test | Throughput / Bandwidth | Notes |
|---|---:|---|
| UDP baseline | 500 Mbit/s (configured) | 596 MBytes transferred; 2/69,834 datagrams lost (0.0029%); jitter = 0.026 ms |
| TCP baseline | 4.96 Gbit/s | 5.78 GBytes transferred; 103 retransmissions |

The maximum throughput measured was 4.96 Gbit/s for TCP and 500 Mbit/s for UDP. The UDP result matched the configured `-b 500M` sending rate.

---

### 6. 100 ms Delay

A 100 ms delay was applied to `ens5` on both the Client and Server:

```bash
# Run on both VMs
sudo tc qdisc add dev ens5 root netem delay 100ms
```

![UDP result with 100 ms delay](images/part1/delay_100ms_udp.png)

![TCP result with 100 ms delay](images/part1/delay_100ms_tcp.png)

| Test | Result with 100 ms delay |
|---|---|
| UDP | 490 Mbit/s received; 0% loss; 0.019 ms jitter |
| TCP | 97.7 Mbit/s received; 0 retransmissions |

UDP stayed near its configured rate. Applying 100 ms in both directions added about 200 ms to the RTT and reduced TCP throughput.

---

### 7. 10% Packet Loss

The `ens5` interface on both the Client and Server was configured with 10% packet loss:

```bash
# Run on both VMs
sudo tc qdisc change dev ens5 root netem delay 0ms loss 10%
```

![UDP result with 10% packet loss](images/part1/loss_10pct_udp.png)

![TCP result with 10% packet loss](images/part1/loss_10pct_tcp.png)

| Test | Result with 10% loss |
|---|---|
| UDP | 450 Mbit/s received; 7,013/69,834 lost (10%); 0.010 ms jitter |
| TCP | 5.96 Mbit/s received; 108 retransmissions |

UDP maintained its sending rate but lost packets. TCP throughput fell sharply because both data packets and ACKs were subject to loss.

---

### 8. Kernel Log and MTU

Kernel messages were monitored during the tests:

```bash
sudo dmesg -wH &
```

No relevant kernel errors were observed.

Interface MTU was checked using:

```bash
ip link show ens5
```

**MTU:** `9001` bytes on `ens5`. This jumbo MTU carries more data per packet, reducing packet rate and header/CPU overhead, which can improve throughput. If the path does not support it, packets may be fragmented or dropped. The MTU was inspected but not varied in this experiment.

---

### 9. 100 Mbit/s Rate Limiting

The previous qdiscs were removed from `ens5` on both the Client and Server:

```bash
# Run on both VMs
sudo tc qdisc del dev ens5 root
```

A Token Bucket Filter was then configured on the Client:

```bash
sudo tc qdisc add dev ens5 root tbf rate 100mbit latency 1ms burst 9015
```

`rate 100mbit` sets the average rate limit, `burst 9015` allows a short 9,015-byte burst, and `latency 1ms` limits how long packets may wait in the queue.

![UDP result with 100 Mbit/s TBF](images/part1/tbf_100mbps_udp.png)

![TCP result with 100 Mbit/s TBF](images/part1/tbf_100mbps_tcp.png)


| Test | Receiver throughput | Other result |
|---|---:|---|
| TCP | 96.9 Mbit/s | 412 retransmissions |
| UDP | 99.3 Mbit/s | 55,954/69,828 lost (80%); 0.399 ms jitter |

The 100 Mbit/s TBF capped both flows near 100 Mbit/s. The UDP sender still transmitted at 500 Mbit/s, so excess packets were dropped.

---

### 10. ethtool Test

The following command was tested:

```bash
sudo ethtool -s ens5 speed 10
```

**Observed error/result:**  
`netlink error: Operation not supported`

**Explanation:** AWS exposes a virtual NIC; its link speed is managed by the platform and cannot be changed with `ethtool` from the guest OS.

---

### 11. Experiments on VyOS

The previous `tc` configurations were removed from the Client and Server.

The experiments were repeated with the same impairment values on VyOS `eth1`, which was not associated with an Elastic IP. Unlike the two-VM tests, this affected only traffic leaving `eth1`.

![UDP result with 100 ms delay on VyOS eth1](images/part1/vyos_eth1_delay_100ms_udp.png)

![TCP result with 100 ms delay on VyOS eth1](images/part1/vyos_eth1_delay_100ms_tcp.png)

![UDP result with 10% packet loss on VyOS eth1](images/part1/vyos_eth1_loss_10pct_udp.png)

![TCP result with 10% packet loss on VyOS eth1](images/part1/vyos_eth1_loss_10pct_tcp.png)

![UDP result with 100 Mbit/s TBF on VyOS eth1](images/part1/vyos_eth1_tbf_100mbps_udp.png)

![TCP result with 100 Mbit/s TBF on VyOS eth1](images/part1/vyos_eth1_tbf_100mbps_tcp.png)

| Configuration | UDP receiver result | TCP receiver result |
|---|---|---|
| 100 ms delay | 494 Mbit/s; 119/69,834 lost (0.17%); 0.015 ms jitter | 265 Mbit/s; 0 retransmissions |
| 10% loss | 450 Mbit/s; 7,013/69,833 lost (10%); 0.030 ms jitter | 23.8 Mbit/s; 392 retransmissions |
| 100 Mbit/s TBF | 97.7 Mbit/s; 56,186/69,829 lost (80%); 0.385 ms jitter | 90.3 Mbit/s; 1,780 retransmissions |

VyOS produced the same overall behavior: delay reduced TCP throughput, loss caused retransmissions, and TBF capped throughput. Results differed because only one egress interface was impaired.

---

### 12. Discussion Questions

#### How can DHCP be started or configured on VyOS?

For a persistent configuration, use `configure`, `set interfaces ethernet eth0 address dhcp`, `commit`, and `save`. A temporary DHCP client can be started with `sudo dhclient eth0`. In this experiment, no manual DHCP step was recorded because the VyOS interfaces already had their AWS-assigned private addresses.

#### Why does SSH disconnect if the IP/interface used by the Elastic IP is changed?

The Elastic IP is associated with one ENI/private IP. Changing that interface or IP breaks the existing SSH path, so the session disconnects.

#### How does AWS map a public Elastic IP to a private interface IP?

AWS maps the public Elastic IP to the ENI's private address using one-to-one NAT.

#### Why could the VMs not initially access the Internet, and how was it fixed?

The Internet route was available through the SSH/EIP interface, but DNS resolution was not configured correctly. A DNS server was added to `/etc/resolvconf/resolv.conf.d/base`, and the resolver configuration was regenerated:

```bash
# Add this line to /etc/resolvconf/resolv.conf.d/base
nameserver 8.8.8.8
sudo resolvconf -u
```

After DNS resolution worked, `sudo apt update` downloaded the latest package indexes. It did not install package upgrades.

#### Why can TCP and UDP show different throughput on the same link?

UDP sends at the configured rate without recovery. TCP adapts its rate and retransmits when packets are lost.

#### Why is changing NIC speed with `ethtool` not supported in AWS?

The NIC is virtual hardware controlled by AWS, not a physical adapter whose speed the guest can set.

---

### 13. Conclusion

The Client–VyOS–Server topology forwarded traffic successfully with 0% ping loss. Baseline TCP reached 4.96 Gbit/s; 100 ms delay and 10% loss greatly reduced TCP throughput, while TBF limited both TCP and UDP receiver throughput to about 100 Mbit/s.

# Part 2 — Fast and Reliable File Transfer in the Cloud

## 1. Test Setup

The sender transferred a 1 GiB file using the Lab 2 reliable-UDP program with an 8,192-packet window. MTU 1500 used a 1,400-byte payload and MTU 9001 used an 8,900-byte payload. `iperf3` and `ping` were run as reference measurements.

The three cases follow the Lab 2 target conditions: Case 1 uses 10 ms RTT, 1% loss per direction, and a 100 Mbit/s rate limit; Case 2 uses 200 ms RTT, 20% loss per direction, and a 100 Mbit/s rate limit; Case 3 uses 200 ms RTT, no random loss, 100 Mbit/s client/server limits, and an 80 Mbit/s router limit. UDP sending and file-transfer pacing were set to 95 Mbit/s for Cases 1–2 and 75 Mbit/s for Case 3. Ping loss is a round-trip measurement, whereas iperf3 UDP loss is measured in one direction.

## 2. Results

### Case 1 — Low-latency path (about 10–13 ms RTT)

| MTU | Ping loss / avg RTT | iperf3 UDP receiver | iperf3 TCP receiver | Reliable UDP transfer |
|---:|---|---|---|---|
| 1500 | 0% / 12.859 ms | 93.9 Mbit/s; 1.1% loss | 11.5 Mbit/s; 101 retransmissions | 97.527 s; 88.08 Mbit/s; 15,340 retransmissions |
| 9001 | 2.5% / 10.546 ms | 94.0 Mbit/s; 0.91% loss | 11.6 Mbit/s; 45 retransmissions | 91.288 s; 94.10 Mbit/s; 113 retransmissions |

Jumbo frames improved the reliable-UDP throughput by about 6 Mbit/s and substantially reduced retransmissions.

![Case 1 reliable UDP, MTU 1500](images/part2/case1_mtu1500_custom_transfer_sender.png)

![Case 1 reliable UDP, MTU 9001](images/part2/case1_mtu9001_custom_transfer_sender.png)

### Case 2 — High-delay, lossy path (about 200 ms RTT)

| MTU | Ping loss / avg RTT | iperf3 UDP receiver | iperf3 TCP receiver | Reliable UDP transfer |
|---:|---|---|---|---|
| 1500 | 40% / 200.587 ms | 62.1 Mbit/s; 21% loss | 52.5 Kbit/s; 17 retransmissions | 152.192 s; 56.44 Mbit/s; 430,572 retransmissions |
| 9001 | 30% / 200.564 ms | 74.5 Mbit/s; 20% loss | 206 Kbit/s; 25 retransmissions | 146.011 s; 58.83 Mbit/s; 68,045 retransmissions |

Loss and long delay severely reduced TCP performance. The reliable-UDP program completed in both MTU settings; MTU 9001 was faster and required fewer retransmissions, although its FIN handshake timed out three times.

![Case 2 reliable UDP, MTU 1500](images/part2/case2_mtu1500_custom_transfer_sender.png)

![Case 2 reliable UDP, MTU 9001](images/part2/case2_mtu9001_custom_transfer_sender.png)

### Case 3 — High-delay, loss-free path (about 200 ms RTT)

| MTU | Ping loss / avg RTT | iperf3 UDP receiver | iperf3 TCP receiver | Reliable UDP transfer |
|---:|---|---|---|---|
| 1500 | 0% / 200.552 ms | 73.5 Mbit/s; 0% loss | 15.0 Mbit/s; 0 retransmissions | 120.838 s; 71.09 Mbit/s; 2 retransmissions |
| 9001 | 0% / 200.622 ms | 73.5 Mbit/s; 0% loss | 719 Kbit/s; 20 retransmissions | 116.318 s; 73.85 Mbit/s; 83 retransmissions |

With no measured UDP loss, receiver throughput was 73.5 Mbit/s, close to the configured 75 Mbit/s sending rate. The reliable-UDP transfer completed faster with MTU 9001. The 719 Kbit/s TCP result at MTU 9001 was caused by TBF queue behavior: jumbo packets consumed most of the available burst at once, causing extra queueing/drops and TCP retransmissions, which sharply reduced throughput.

![Case 3 reliable UDP, MTU 1500](images/part2/case3_mtu1500_custom_transfer_sender.png)

![Case 3 reliable UDP, MTU 9001](images/part2/case3_mtu9001_custom_transfer_sender.png)

## 3. Comparison with Lab 2

The following table compares reliable-UDP file transfers with the results in the Lab 2 report. Throughput is file goodput, calculated from file size divided by total transfer time.

| Case | MTU | Lab 2 time (s) | AWS time (s) | Lab 2 goodput (Mbit/s) | AWS goodput (Mbit/s) | Goodput change |
|---|---:|---:|---:|---:|---:|---:|
| 1 | 1500 | 97.539 | 97.527 | 88.066 | 88.078 | ~0% |
| 1 | 9001 | 94.692 | 91.288 | 90.714 | 94.097 | +3.7% |
| 2 | 1500 | 170.754 | 152.192 | 50.306 | 56.442 | +12.2% |
| 2 | 9001 | 158.518 | 146.011 | 54.189 | 58.831 | +8.6% |
| 3 | 1500 | 115.597 | 120.838 | 74.309 | 71.087 | −4.3% |
| 3 | 9001 | 111.600 | 116.318 | 76.971 | 73.849 | −4.1% |

AWS performance was similar or better in Case 1, improved by about 9–12% in Case 2, and was about 4% lower in Case 3. MTU 9001 shortened transfer time in every case in both environments. AWS Case 2 retransmissions decreased from 435,200 to 430,572 at MTU 1500 and from 85,230 to 68,045 at MTU 9001. These are individual runs, so the differences cannot be attributed to the platform alone.

Lab 2 verified all received files using MD5. In the provided AWS integrity check, both the original `data.bin` and the received `received.bin` had MD5 `65d30222f2668c9552382240d5c947a5`, confirming that the received file matched the original. The sender results also show that all data packets were acknowledged in every case.

## 4. Conclusion

The Lab 2 file-transfer program ran successfully on AWS and acknowledged all data packets in every case. It sustained 71–94 Mbit/s on the low-loss paths and remained at 56–59 Mbit/s on the high-delay, lossy path, where TCP fell to Kbit/s-level throughput. MTU 9001 generally reduced transfer time, but the benefit depends on the path condition.
