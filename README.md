# Quantus 5060 Miner

This is a source fork of [Quantus-Network/quantus-miner](https://github.com/Quantus-Network/quantus-miner) at `c7838cbc86f7d74477da1771139f377a8e438072`, under its retained Apache-2.0 license. The fork tunes the CUDA launch target and fuses the existing Goldilocks multiply and row-sum implementation; the consensus hash is unchanged. It adds a transparent pool-mode project fee, tested on an RTX 5060 Ti. It has no fee for private-node mining. The project fee recipient is visible as `PROJECT_FEE_ADDRESS` in `crates/miner-cli/src/main.rs`.

The CUDA kernel is tuned for this 5060 Ti. Four adjacent 20-second runs measured 236.18 and 236.19 MH/s for the prior fork build versus 250.03 and 250.06 MH/s with the fused internal round. Median active GPU samples were 132.69/133.21 W and 134.41/134.73 W, respectively. All 10 CUDA tests passed, and Quanpool credited 13 accepted shares with zero stale or invalid in a short live run. See [the performance evidence](docs/5060ti-optimization.md). These are GPU-only samples; other cards may respond differently.

For a Quanpool-compatible token (`qz...` or `qz....worker`), pass `--pool-mode`. The miner sends 99 minutes of scheduled time to your payout address, then 1 minute to the project payout address. It closes the QUIC connection and changes the token at each boundary; connection/setup time is inside the fee minute, so actual credited work can be less than 1% and varies with pool difficulty and luck. If your payout address equals the project address, fee switching is omitted. The mode rejects tokens that do not look like Quantus payout addresses. A permanent project authentication failure stops mining instead of silently mining without the stated fee.

```bash
./target/release/quantus-miner serve --pool-mode \
  --node-addr YOUR_POOL_QUIC_ENDPOINT \
  --auth-token-file /private/path/pool-payout-token \
  --tls-cert-sha256-file /private/path/pool-cert-fingerprint \
  --cuda-gpu --gpu-devices 1 --cpu-workers 0
```

On 2026-10-04, a source-built fork connected to Quanpool with the local project wallet and the pool recorded 15 new accepted shares, zero stale, and zero invalid in a 35-second run. A separate 85-second integration build temporarily shortened the schedule to 15 seconds user / 5 seconds project, then Quanpool recorded 46 accepted test-user shares and 45 project-fee-worker shares, each with zero stale or invalid. The release source was restored to the 99-minute / 1-minute schedule and rebuilt. These short checks do not prove long-term uptime, a completed payout, or profitability.

## Upstream documentation

High-performance external mining service for Quantus Network with support for CPU, GPU, and hybrid CPU+GPU mining.

## Building

```bash
# CPU-only build (default)
cargo build -p miner-cli --release

# With GPU support (recommended)
cargo build -p miner-cli --release
```

The binary will be available at `target/release/quantus-miner`.

## Running

The node requires a shared auth token and TLS cert pin. Both live under the
node's chain config dir (`<base-path>/chains/<chain>/`):

- `miner-auth-token` — shared secret (**not** logged by the node; read this file)
- `miner-tls-cert-sha256` — SHA-256 of the miner TLS cert (also printed in node logs)

```bash
# Preferred: mount/read the node's chain config files
./target/release/quantus-miner serve \
  --node-addr 127.0.0.1:9833 \
  --auth-token-file /path/to/miner-auth-token \
  --tls-cert-sha256-file /path/to/miner-tls-cert-sha256 \
  --cpu-workers 4

# Or pass values directly (token from miner-auth-token; fingerprint also in node logs)
./target/release/quantus-miner serve \
  --node-addr 127.0.0.1:9833 \
  --auth-token <TOKEN> \
  --tls-cert-sha256 <FINGERPRINT> \
  --gpu-devices 1

# Hybrid CPU+GPU mining
./target/release/quantus-miner serve \
  --node-addr 127.0.0.1:9833 \
  --auth-token-file /path/to/miner-auth-token \
  --tls-cert-sha256-file /path/to/miner-tls-cert-sha256 \
  --cpu-workers 4 \
  --gpu-devices 1
```

## Configuration

| Argument | Environment Variable | Description | Default |
|----------|---------------------|-------------|---------|
| `--node-addr <ADDR>` | `MINER_NODE_ADDR` | Node address to connect to | `127.0.0.1:9833` |
| `--auth-token <TOKEN>` | `MINER_AUTH_TOKEN` | Shared secret from the node's `miner-auth-token` file (not logged) | required |
| `--auth-token-file <PATH>` | `MINER_AUTH_TOKEN_FILE` | Read the shared secret from a file (preferred) | — |
| `--pool-mode` | `MINER_POOL_MODE` | Quanpool-compatible payout token and transparent project fee | off |
| `--tls-cert-sha256 <HEX>` | `MINER_TLS_CERT_SHA256` | SHA-256 of the node's miner TLS cert (`miner-tls-cert-sha256` / node logs) | required |
| `--tls-cert-sha256-file <PATH>` | `MINER_TLS_CERT_SHA256_FILE` | Read the TLS cert fingerprint from a file | — |
| `--cpu-workers <N>` | `MINER_CPU_WORKERS` | Number of CPU worker threads | Auto-detect |
| `--gpu-devices <N>` | `MINER_GPU_DEVICES` | Number of GPU devices | Auto-detect |
| `--cuda-gpu` | `MINER_CUDA_GPU` | Use native CUDA instead of wgpu/Vulkan (NVIDIA) | off |
| `--gpu-batch-size <N>` | `MINER_GPU_BATCH_SIZE` | GPU batch size in nonces | 1000000 (32000000 with `--cuda-gpu`) |
| `--cpu-batch-size <N>` | `MINER_CPU_BATCH_SIZE` | CPU batch size in hashes | 10000 |
| `--gpu-throttle-ms <MS>` | `MINER_GPU_THROTTLE_MS` | Sleep duration (ms) between GPU batches | 0 |
| `--metrics-port <PORT>` | `MINER_METRICS_PORT` | Prometheus metrics port | 9900 |
| `--metrics-bind <IP>` | `MINER_METRICS_BIND` | Metrics listener IP; set `0.0.0.0` only when remote scraping is intended | `127.0.0.1` |

## GPU Mining

Two GPU paths:

- **wgpu** (default): Metal on macOS, Vulkan/DX12 elsewhere. Needs a graphics driver stack.
- **CUDA** (`--cuda-gpu`): native NVIDIA compute. Use this on Clore.ai and other CUDA-only boxes.

On NVIDIA Linux boxes that have CUDA but not Vulkan:

```bash
./target/release/quantus-miner benchmark --cuda-gpu --gpu-devices 1 --cpu-workers 0 --duration 10
```

### Setup

**Build with GPU support:**
```bash
cargo build -p miner-cli --release
```

**Platform requirements:**
- **macOS**: Works out-of-the-box
- **Linux**: Install GPU drivers (`nvidia-driver`, `mesa-vulkan-drivers`)
- **Windows**: Ensure recent graphics drivers are installed

### Performance Monitoring

- **macOS**: `sudo powermetrics --samplers gpu_power -i 1000`
- **Linux**: `nvidia-smi` (NVIDIA) or `radeontop` (AMD)  
- **Windows**: Task Manager GPU tab

## Examples

All `serve` examples need the auth token and TLS pin (files or inline values).

```bash
# CPU mining with 8 workers
./target/release/quantus-miner serve \
  --auth-token-file /path/to/miner-auth-token \
  --tls-cert-sha256-file /path/to/miner-tls-cert-sha256 \
  --cpu-workers 8

# Pure GPU mining (wgpu)
./target/release/quantus-miner serve \
  --auth-token-file /path/to/miner-auth-token \
  --tls-cert-sha256-file /path/to/miner-tls-cert-sha256 \
  --gpu-devices 1

# NVIDIA CUDA mining (no Vulkan)
./target/release/quantus-miner serve \
  --auth-token-file /path/to/miner-auth-token \
  --tls-cert-sha256-file /path/to/miner-tls-cert-sha256 \
  --cuda-gpu --gpu-devices 1 --cpu-workers 0

# GPU mining with throttle (reduce GPU utilization)
./target/release/quantus-miner serve \
  --auth-token-file /path/to/miner-auth-token \
  --tls-cert-sha256-file /path/to/miner-tls-cert-sha256 \
  --gpu-devices 1 --gpu-throttle-ms 50

# Hybrid mining: 4 CPU + 1 GPU workers
./target/release/quantus-miner serve \
  --auth-token-file /path/to/miner-auth-token \
  --tls-cert-sha256-file /path/to/miner-tls-cert-sha256 \
  --cpu-workers 4 --gpu-devices 1

# With verbose logging
RUST_LOG=debug ./target/release/quantus-miner serve \
  --auth-token-file /path/to/miner-auth-token \
  --tls-cert-sha256-file /path/to/miner-tls-cert-sha256 \
  --cpu-workers 2 --gpu-devices 1

# Production setup with metrics
./target/release/quantus-miner serve \
  --node-addr 127.0.0.1:9833 \
  --auth-token-file /path/to/miner-auth-token \
  --tls-cert-sha256-file /path/to/miner-tls-cert-sha256 \
  --cpu-workers 6 \
  --gpu-devices 1 \
  --metrics-port 9900
```

## Protocol

The miner uses a QUIC-based protocol for communication with the node:

- **Transport**: QUIC with TLS 1.3 (self-signed certificate, pinned by SHA-256)
- **Auth**: `Ready { token }` must match the node's `miner-auth-token`
- **ALPN**: `quantus-miner/2`
- **Port**: 9833 (default)
- **Messages**: `Ready` (miner→node), `NewJob` (node→miner), `JobResult` (miner→node)

For full protocol specification, see the node's `MINING.md`.

## Benchmarking

```bash
# Benchmark CPU performance
./target/release/quantus-miner benchmark --cpu-workers 8 --duration 30

# Benchmark GPU performance  
./target/release/quantus-miner benchmark --gpu-devices 1 --duration 30

# Benchmark hybrid performance
./target/release/quantus-miner benchmark --cpu-workers 4 --gpu-devices 1 --duration 30
```
