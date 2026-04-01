# Ceph RGW Benchmarking with Elbencho - Complete Guide

This guide provides correct elbencho commands for benchmarking Ceph RGW with SSL/TLS endpoints.

## Cluster Information

```
Cluster ID: ab1893fe-17bd-11f1-995f-043201489a60
OSDs: 17 (up and in)
Total Capacity: 25 TiB
Used: 645 GiB
Available: 24 TiB
Current Objects: 1.38M objects (209 GiB)
RGW Daemons: 7 active (3 hosts)
RGW Endpoints:
  - https://10.64.1.14:8000
  - https://10.64.1.15:8000
  - https://10.64.1.16:8000
```

## Prerequisites

### 1. Install Elbencho

```bash
# Download and install elbencho
wget https://github.com/breuner/elbencho/releases/latest/download/elbencho
chmod +x elbencho
sudo mv elbencho /usr/local/bin/

# Or build from source
git clone https://github.com/breuner/elbencho.git
cd elbencho
make -j$(nproc)
sudo make install

# Verify installation
elbencho --version
```

### 2. Setup SSL Certificate

```bash
# Add cephadm CA certificate to system trust store
sudo cp /root/benchmark/cephadm_root_ca.pem /etc/pki/ca-trust/source/anchors/cephadm-root-ca.crt
sudo update-ca-trust

# Verify certificate is trusted
openssl s_client -connect 10.64.1.14:8000 -CAfile /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem
```

### 3. Setup Environment Variables

```bash
# Set S3 credentials
export S3_ENDPOINT="https://10.64.1.14:8000"
export S3_KEY="<your-access-key>"
export S3_SECRET="<your-secret-key>"

# Or use AWS environment variables
export AWS_ACCESS_KEY_ID="<your-access-key>"
export AWS_SECRET_ACCESS_KEY="<your-secret-key>"
```

## Elbencho Command Syntax

### Basic Format

```bash
elbencho --s3endpoints <endpoint> --s3key <key> --s3secret <secret> \
    [OPTIONS] BUCKET_NAME
```

### Key Options

- `-d` or `--mkdirs`: Create buckets
- `-w` or `--write`: Write/upload objects
- `-r` or `--read`: Read/download objects
- `--stat`: Read object status attributes
- `-F` or `--delfiles`: Delete objects
- `-D` or `--deldirs`: Delete buckets
- `-t` or `--threads`: Number of I/O worker threads
- `-n` or `--dirs`: Number of directories (object key prefixes) per thread
- `-N` or `--files`: Number of objects per thread per directory
- `-s` or `--size`: Object size
- `-b` or `--block`: Block size for uploads/downloads
- `--lat`: Show latency statistics
- `--s3fastget`: Send downloads directly to /dev/null (faster)

### Object Count Calculation

```
Total Objects = threads × directories × files
Example: -t 32 -n 10 -N 100 = 32 × 10 × 100 = 32,000 objects
```

## Benchmark Tests

### Test 1: Create Bucket

```bash
elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -d elbencho-test
```

### Test 2: Small Objects (4KB) - High IOPS

```bash
# Write 160,000 small objects (16 threads × 100 dirs × 100 files)
elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -w -t 16 -n 100 -N 100 -s 4k -b 4k \
    --lat \
    elbencho-test

# Expected: Tests small object IOPS performance
# Total data: 160,000 × 4KB = 640 MB
```

### Test 3: Medium Objects (1MB) - Balanced

```bash
# Write 32,000 medium objects (32 threads × 10 dirs × 100 files)
elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -w -t 32 -n 10 -N 100 -s 1m -b 1m \
    --lat \
    elbencho-test

# Expected: Tests balanced throughput
# Total data: 32,000 × 1MB = 32 GB
```

### Test 4: Large Objects (10MB) - Throughput

```bash
# Write 3,200 large objects (32 threads × 5 dirs × 20 files)
elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -w -t 32 -n 5 -N 20 -s 10m -b 5m \
    --lat \
    elbencho-test

# Expected: Tests maximum throughput
# Total data: 3,200 × 10MB = 32 GB
```

### Test 5: Very Large Objects (100MB) - Multipart Upload

```bash
# Write 320 very large objects (32 threads × 2 dirs × 5 files)
elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -w -t 32 -n 2 -N 5 -s 100m -b 16m \
    --lat \
    elbencho-test

# Expected: Tests multipart upload performance
# Total data: 320 × 100MB = 32 GB
```

### Test 6: Read Performance - Small Objects

```bash
# Read 160,000 small objects
elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -r -t 16 -n 100 -N 100 -s 4k -b 4k \
    --lat \
    --s3fastget \
    elbencho-test
```

### Test 7: Read Performance - Medium Objects

```bash
# Read 32,000 medium objects
elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -r -t 32 -n 10 -N 100 -s 1m -b 1m \
    --lat \
    --s3fastget \
    elbencho-test
```

### Test 8: Multi-Endpoint Load Balancing

```bash
# Use all 3 RGW endpoints for load distribution
elbencho --s3endpoints https://10.64.1.14:8000,https://10.64.1.15:8000,https://10.64.1.16:8000 \
    --s3key $S3_KEY --s3secret $S3_SECRET \
    -w -t 64 -n 10 -N 100 -s 1m -b 1m \
    --lat \
    elbencho-test-multiep

# Expected: Better performance with load distribution
# Total: 64,000 objects across 3 endpoints
```

### Test 9: Shared Upload (Multiple Threads per Object)

```bash
# Upload 10 large files (1GB each) using 8 threads per file
elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -w -t 8 -s 1g -b 16m \
    --lat \
    elbencho-test/largefile{1..10}

# Expected: Tests concurrent multipart upload
# Total data: 10 × 1GB = 10 GB
```

### Test 10: Metadata Operations (Stat)

```bash
# Test object stat operations
elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    --stat -t 32 -n 10 -N 100 \
    --lat \
    elbencho-test

# Expected: Tests metadata read performance
```

### Test 11: Mixed Workload

```bash
# Create multiple buckets and test concurrently
for i in {1..5}; do
    elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
        -w -t 16 -n 10 -N 100 -s 1m -b 1m \
        --lat \
        elbencho-test-$i &
done
wait

echo "Mixed workload test completed"
```

## Cleanup Commands

### Delete Objects

```bash
# Delete all objects from Test 2 (small objects)
elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -F -t 16 -n 100 -N 100 \
    elbencho-test

# Delete all objects from Test 3 (medium objects)
elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -F -t 32 -n 10 -N 100 \
    elbencho-test

# Delete all objects from Test 4 (large objects)
elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -F -t 32 -n 5 -N 20 \
    elbencho-test
```

### Delete Bucket

```bash
# Delete bucket (must be empty first)
elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -D elbencho-test
```

## Complete Benchmark Script

```bash
#!/bin/bash
# complete_rgw_benchmark.sh

set -e

# Configuration
S3_ENDPOINT="https://10.64.1.14:8000"
S3_KEY="<your-access-key>"
S3_SECRET="<your-secret-key>"
BUCKET="elbencho-benchmark"
RESULTS_DIR="/root/benchmark/results"

# Create results directory
mkdir -p $RESULTS_DIR
cd $RESULTS_DIR

# Log function
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a benchmark.log
}

log "=== Starting Comprehensive RGW Benchmark ==="
log "Cluster Status:"
ceph -s | tee -a benchmark.log

# Create bucket
log "=== Creating bucket: $BUCKET ==="
elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -d $BUCKET

# Test 1: Small objects (4KB)
log "=== Test 1: Small objects (4KB) - 160,000 objects ==="
elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -w -t 16 -n 100 -N 100 -s 4k -b 4k --lat $BUCKET \
    2>&1 | tee -a test1_4k_write.log

# Test 2: Medium objects (1MB)
log "=== Test 2: Medium objects (1MB) - 32,000 objects ==="
elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -w -t 32 -n 10 -N 100 -s 1m -b 1m --lat $BUCKET \
    2>&1 | tee -a test2_1m_write.log

# Test 3: Large objects (10MB)
log "=== Test 3: Large objects (10MB) - 3,200 objects ==="
elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -w -t 32 -n 5 -N 20 -s 10m -b 5m --lat $BUCKET \
    2>&1 | tee -a test3_10m_write.log

# Test 4: Read performance (1MB objects)
log "=== Test 4: Read performance (1MB) ==="
elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -r -t 32 -n 10 -N 100 -s 1m -b 1m --lat --s3fastget $BUCKET \
    2>&1 | tee -a test4_1m_read.log

# Test 5: Multi-endpoint test
log "=== Test 5: Multi-endpoint load balancing ==="
elbencho --s3endpoints https://10.64.1.14:8000,https://10.64.1.15:8000,https://10.64.1.16:8000 \
    --s3key $S3_KEY --s3secret $S3_SECRET \
    -w -t 64 -n 10 -N 100 -s 1m -b 1m --lat ${BUCKET}-multiep \
    2>&1 | tee -a test5_multiep_write.log

# Cleanup
log "=== Cleanup: Deleting objects ==="
elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -F -t 32 -n 115 -N 100 $BUCKET

elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -F -t 64 -n 10 -N 100 ${BUCKET}-multiep

log "=== Cleanup: Deleting buckets ==="
elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -D $BUCKET

elbencho --s3endpoints $S3_ENDPOINT --s3key $S3_KEY --s3secret $S3_SECRET \
    -D ${BUCKET}-multiep

log "=== Benchmark Complete ==="
log "Final Cluster Status:"
ceph -s | tee -a benchmark.log

log "Results saved in: $RESULTS_DIR"
```

## Cluster Capacity Analysis

### Current State

```
Total Capacity: 25 TiB (27,487,790,694,400 bytes)
Used: 645 GiB (692,674,641,920 bytes)
Available: 24 TiB (26,388,279,066,624 bytes)
Current Objects: 1.38M objects
Current Data: 209 GiB
Replication: 3x (default)
```

### Usable Capacity Calculation

```bash
# With 3x replication
Available_Raw = 24 TiB
Overhead = 20% (metadata, indexes, etc.)
Usable_After_Overhead = 24 TiB × 0.80 = 19.2 TiB
Usable_With_Replication = 19.2 TiB / 3 = 6.4 TiB

# Result: ~6.4 TiB usable capacity
```

### Maximum Objects by Size

| Object Size | Max Objects | Storage Used | Objects/Bucket (1K buckets) | Objects/Bucket (5K buckets) |
|-------------|-------------|--------------|-----------------------------|-----------------------------|
| 4 KB        | 1.68 billion| 6.4 TiB      | 1.68 million               | 336 thousand                |
| 10 KB       | 671 million | 6.4 TiB      | 671 thousand               | 134 thousand                |
| 100 KB      | 67 million  | 6.4 TiB      | 67 thousand                | 13 thousand                 |
| 1 MB        | 6.7 million | 6.4 TiB      | 6.7 thousand               | 1.3 thousand                |
| 10 MB       | 671 thousand| 6.4 TiB      | 671                        | 134                         |
| 100 MB      | 67 thousand | 6.4 TiB      | 67                         | 13                          |
| 1 GB        | 6.7 thousand| 6.4 TiB      | 6.7                        | 1.3                         |

### Bucket Recommendations

```
Recommended Maximum Buckets: 2,000 - 5,000
Optimal Buckets: 2,000 - 3,000

Reasons:
- RGW performance degrades with >10,000 buckets
- Metadata overhead increases with bucket count
- Index sharding becomes complex
- Bucket listing operations slow down
```

### Performance Expectations

```bash
# Based on 17 OSDs and 7 RGW daemons

Write Throughput (aggregate): 500-1,000 MB/s
Read Throughput (aggregate): 800-1,500 MB/s
Small Object IOPS (4KB): 5,000-10,000 IOPS
Latency (1MB objects): 10-50 ms

# Per RGW daemon (7 daemons):
Write per daemon: 70-140 MB/s
Read per daemon: 110-210 MB/s
```

## Monitoring During Benchmarks

### Terminal 1: Cluster Status

```bash
watch -n 2 'ceph -s'
```

### Terminal 2: OSD Performance

```bash
watch -n 2 'ceph osd perf'
```

### Terminal 3: Pool Statistics

```bash
watch -n 2 'ceph df detail'
```

### Terminal 4: RGW Daemon Status

```bash
watch -n 2 'ceph orch ps --daemon-type rgw'
```

### Terminal 5: Network Monitoring

```bash
# Monitor network interface (replace eth0 with your interface)
watch -n 2 'ifstat -i eth0'
```

## Interpreting Results

### Good Performance Indicators

- ✅ Write throughput: >500 MB/s aggregate
- ✅ Read throughput: >800 MB/s aggregate
- ✅ Latency (1MB): <50ms average
- ✅ IOPS (4KB): >5,000
- ✅ No slow requests in `ceph -s`
- ✅ Balanced OSD usage

### Performance Issues

- ❌ Write throughput: <200 MB/s
- ❌ High latency: >100ms average
- ❌ Slow requests appearing
- ❌ Unbalanced OSD usage
- ❌ High CPU usage on RGW daemons

## Troubleshooting

### Issue: Low Throughput

```bash
# Check OSD performance
ceph osd perf

# Check for slow requests
ceph health detail

# Check RGW daemon logs
ceph orch logs rgw.ssl_rgw_qat_hw.ceph14.kdojpf
```

### Issue: SSL Certificate Errors

```bash
# Verify CA certificate is in trust store
ls -l /etc/pki/ca-trust/source/anchors/

# Test SSL connection
openssl s_client -connect 10.64.1.14:8000 -CApath /etc/pki/ca-trust/extracted/pem/
```

### Issue: Connection Refused

```bash
# Check RGW daemons are running
ceph orch ps --daemon-type rgw

# Check RGW ports
ss -tlnp | grep 8000

# Test connectivity
curl -v https://10.64.1.14:8000
```

## Best Practices

1. **Start Small**: Begin with small tests and gradually increase load
2. **Monitor Cluster**: Watch cluster health during benchmarks
3. **Use Multiple Endpoints**: Distribute load across all RGW daemons
4. **Cleanup**: Always cleanup test data after benchmarks
5. **Document Results**: Save logs and results for analysis
6. **Avoid Production Impact**: Run benchmarks during maintenance windows

## Summary

This guide provides:
- ✅ Correct elbencho syntax for your RGW setup
- ✅ Comprehensive benchmark tests
- ✅ Capacity calculations for your cluster
- ✅ Performance expectations
- ✅ Monitoring and troubleshooting tips

Your cluster can support:
- **2,000-5,000 buckets** (recommended)
- **6.7 million objects** (1MB average)
- **~6.4 TiB usable capacity** (with 3x replication)
- **500-1,000 MB/s throughput** (aggregate)