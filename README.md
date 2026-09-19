# EE 542 Lab 3

Internet and Cloud Computing Lab 3 submission, covering AWS network performance experiments, cloud file-transfer tests, and Linux TCP congestion-control modifications.

## Contents

- [report.md](report.md) — complete experiment report and results.
- `images/` — screenshots and measurements for all three parts.
- `0001-ee542-linear-rto.patch` — enables linear RTO behavior for established TCP connections.
- `0002-bbr-lossy-cwnd-floor.patch` — adds a configurable BBR congestion-window floor for lossy links.
- `0001-EE542-improve-BBR-performance-on-lossy-high-RTT-link.patch` — improves BBR behavior on high-RTT, lossy links with cwnd, pacing-rate, and loss-threshold adjustments.

## Experiment Parts

1. AWS topology, routing, delay/loss/rate-limit experiments, and TCP/UDP performance analysis.
2. Fast and reliable file-transfer evaluation under different RTT, loss, and MTU settings.
3. Linux TCP/BBR modifications and performance comparison on lossy, high-latency paths.

See [report.md](report.md) for setup details, commands, measurements, and conclusions.
