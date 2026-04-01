# Ceph RGW with Ingress and QAT - Deployment Scenarios

This document describes all supported (and unsupported) deployment scenarios for Ceph RGW with HAProxy Ingress and Intel QAT (QuickAssist Technology) acceleration.

---

## Table of Contents

1. [RGW HTTP Only](#1-rgw-http-only)
2. [RGW HTTP + QAT Compression](#2-rgw-http--qat-compression)
3. [RGW HTTPS Only (No Ingress, No QAT)](#3-rgw-https-only-no-ingress-no-qat)
4. [RGW HTTPS + QAT](#4-rgw-https--qat)
5. [Ingress Terminate + RGW HTTP (No QAT)](#5-ingress-terminate--rgw-http-no-qat)
6. [Ingress Terminate + QAT at Ingress (NOT SUPPORTED)](#6-ingress-terminate--qat-at-ingress-not-supported)
7. [Ingress Terminate + QAT at RGW](#7-ingress-terminate--qat-at-rgw)
8. [Ingress Terminate + QAT at RGW + QAT at Backend (NOT SUPPORTED)](#8-ingress-terminate--qat-at-rgw--qat-at-backend-not-supported)
9. [Ingress Passthrough + RGW HTTPS](#9-ingress-passthrough--rgw-https)
10. [Ingress Passthrough + RGW HTTPS + QAT at RGW](#10-ingress-passthrough--rgw-https--qat-at-rgw)
11. [Ingress Re-encrypt + RGW HTTPS](#11-ingress-re-encrypt--rgw-https)
12. [Ingress Re-encrypt + QAT at Ingress (NOT SUPPORTED)](#12-ingress-re-encrypt--qat-at-ingress-not-supported)
13. [Ingress Re-encrypt + QAT at RGW](#13-ingress-re-encrypt--qat-at-rgw)
14. [Ingress Re-encrypt + QAT at RGW + QAT at Ingress (NOT SUPPORTED)](#14-ingress-re-encrypt--qat-at-rgw--qat-at-ingress-not-supported)

---

## 1. RGW HTTP Only

**Traffic Flow:** `Client --HTTP--> RGW (port 80)`

The simplest deployment with no encryption, no load balancing, and no QAT acceleration.

- **Pros:** Simple, fast setup
- **Cons:** No encryption, no HA, no QAT acceleration

### RGW Spec

```yaml
service_type: rgw
service_id: simple.http
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  rgw_frontend_port: 80
  ssl: false
```

### Ingress Spec

_Not applicable - no ingress in this scenario._

---

## 2. RGW HTTP + QAT Compression

**Traffic Flow:** `Client --HTTP--> RGW (port 8001) with QAT compression`

RGW with QAT hardware-accelerated compression enabled but no SSL encryption. Two variants are shown: hardware QAT and software QAT.

- **Pros:** QAT compression, simple setup
- **Cons:** No encryption, no HA

### RGW Spec (Hardware QAT)

```yaml
service_type: rgw
service_id: nossl_rgw_qat_hw
service_name: rgw.nossl_rgw_qat_hw
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  qat:
    compression: hw
  rgw_exit_timeout_secs: 120
  rgw_frontend_port: 8001
```

### RGW Spec (Software QAT)

```yaml
service_type: rgw
service_id: nossl_rgw_qat_sw
service_name: rgw.nossl_rgw_qat_sw
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  qat:
    compression: sw
```

### Ingress Spec

_Not applicable - no ingress in this scenario._

---

## 3. RGW HTTPS Only (No Ingress, No QAT)

**Traffic Flow:** `Client --HTTPS--> RGW (port 8022)`

Standalone RGW with SSL/TLS enabled using cephadm-signed certificates. No ingress load balancer, no QAT acceleration.

### RGW Spec

```yaml
service_type: rgw
service_id: ssl_rgw_only
service_name: rgw.ssl_rgw_only
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  certificate_source: cephadm-signed
  generate_cert: true
  rgw_exit_timeout_secs: 120
  rgw_frontend_port: 8022
  ssl: true
```

### Ingress Spec

_Not applicable - no ingress in this scenario._

---

## 4. RGW HTTPS + QAT

**Traffic Flow:** `Client --HTTPS--> RGW (port 8022) with QAT`

RGW with both SSL/TLS encryption and QAT acceleration. Provides encrypted traffic with hardware-accelerated compression.

- **Pros:** Encrypted, QAT acceleration
- **Cons:** No HA, no load balancing

### RGW Spec

```yaml
service_type: rgw
service_id: ssl_rgw_qat_hw
service_name: rgw.ssl_rgw_qat_hw
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  certificate_source: cephadm-signed
  generate_cert: true
  qat:
    compression: hw
  rgw_exit_timeout_secs: 120
  rgw_frontend_port: 8023
  ssl: true
```

### RGW Spec (Software QAT Variant)

```yaml
service_type: rgw
service_id: ssl_rgw_qat_sw
service_name: rgw.ssl_rgw_qat_sw
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  certificate_source: cephadm-signed
  generate_cert: true
  qat:
    compression: sw
  rgw_exit_timeout_secs: 120
  rgw_frontend_port: 8024
  ssl: true
```

### Ingress Spec

_Not applicable - no ingress in this scenario._

---

## 5. Ingress Terminate + RGW HTTP (No QAT)

**Traffic Flow:** `Client --HTTPS--> Ingress (SSL terminate) --HTTP--> RGW (port 80)`

HAProxy terminates SSL from clients (HTTP mode, default). Backend RGW may or may not have SSL independently. By default, SSL terminates at HAProxy if passthrough parameter is not set.

- **Pros:** HA, load balancing, SSL termination at ingress
- **Cons:** Decrypted backend traffic between ingress and RGW

### RGW Spec

```yaml
service_type: rgw
service_id: rgw.nossl.one
service_name: rgw.rgw.nossl.one
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  rgw_frontend_port: 8070
  ssl: false
```

### Ingress Spec

```yaml
service_type: ingress
service_id: rgw.ssl_qat_ingress_rgw_nossl
service_name: ingress.qat_ingress_rgw_nossl
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  backend_service: rgw.nossl.one
  first_virtual_router_id: 51
  frontend_port: 443
  ssl: true
  monitor_port: 1983
  virtual_ip: 9.11.126.96/13
```

---

## 6. Ingress Terminate + QAT at Ingress (NOT SUPPORTED)

**Traffic Flow:** `Client --HTTPS--> Ingress (SSL terminate + QAT) --HTTP--> RGW`

> **NOT SUPPORTED NOW** - Due to HAProxy Image Build Issue

This scenario would use QAT for SSL acceleration at the ingress layer. It requires HAProxy compiled with QAT support. If HAProxy with QAT for SSL is available, it would offload SSL processing to QAT devices.

- **Pros (if supported):** HA, load balancing, QAT SSL
- **Cons:** Decrypted backend traffic

### RGW Spec

```yaml
service_type: rgw
service_id: rgw.nossl.one
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  rgw_frontend_port: 8071
  ssl: false
```

### Ingress Spec

```yaml
service_type: ingress
service_id: rgw.ssl_qat_ingress_rgw_nossl
service_name: ingress.qat_ingress_rgw_nossl
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  backend_service: rgw.nossl.one
  first_virtual_router_id: 54
  frontend_port: 444
  ssl: true
  monitor_port: 1984
  virtual_ip: 9.11.126.97/13
```

---

## 7. Ingress Terminate + QAT at RGW

**Traffic Flow:** `Client --HTTPS--> Ingress (SSL terminate) --HTTP--> RGW (QAT compression)`

SSL is terminated at the ingress (HAProxy). Backend RGW uses QAT for compression acceleration only (not for SSL).

- **Pros:** HA, load balancing, QAT compression
- **Cons:** Decrypted backend, no QAT for SSL

### RGW Spec

```yaml
service_type: rgw
service_id: rgw.nossl.two
service_name: rgw.rgw.nossl.two
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  qat:
    compression: hw
  rgw_frontend_port: 8095
  ssl: false
```

### Ingress Spec

```yaml
service_type: ingress
service_id: ingress.qat_ingress_rgw_nossl_two
service_name: ingress.qat_ingress_rgw_nossl_two
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  backend_service: rgw.nossl.two
  first_virtual_router_id: 52
  frontend_port: 445
  ssl: true
  monitor_port: 1986
  virtual_ip: 9.11.126.52/13
```

---

## 8. Ingress Terminate + QAT at RGW + QAT at Backend (NOT SUPPORTED)

**Traffic Flow:** `Client --HTTPS--> Ingress (SSL terminate + QAT) --HTTP--> RGW (QAT)`

> **NOT SUPPORTED NOW** - Due to HAProxy Image Build Issue

This scenario would have QAT at both the ingress (for SSL offloading) and at the RGW backend (for compression).

### RGW Spec

```yaml
service_type: rgw
service_id: rgw.nossl.three
service_name: rgw.rgw.nossl.three
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  qat:
    compression: hw
  rgw_frontend_port: 8081
  ssl: false
```

### Ingress Spec

```yaml
service_type: ingress
service_id: ingress.qat_ingress_rgw_nossl_three
service_name: ingress.qat_ingress_rgw_nossl_three
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  backend_service: rgw.nossl.three
  first_virtual_router_id: 55
  frontend_port: 446
  haproxy_qat_support: true
  ssl: true
  monitor_port: 1993
  virtual_ip: 9.11.126.99/27
```

---

## 9. Ingress Passthrough + RGW HTTPS

**Traffic Flow:** `Client --HTTPS--> Ingress (passthrough) --HTTPS--> RGW (HTTPS)`

**End-to-End Encryption**

Cephadm can now deploy Ingress over Ceph Object Gateway with the ingress service. HAProxy is configured in **TCP mode** for passthrough. Setting up "haproxy" in TCP mode allows encrypted messages to be passed through "haproxy" directly to Ceph Object Gateway **without** "haproxy" needing to understand the message contents. This allows end-to-end SSL for Ingress and Ceph Object Gateway setups.

With this configuration, users can now specify a certificate for the `rgw` service and set `use_tcp_mode_over_rgw` as `true` in the ingress specification to deploy it for TCP mode. Haproxy deploys/deployed for Ingress in TCP mode, rather than in HTTP mode.

- **Pros:** End to end encryption, HA, load balancing
- **Cons:** No SSL offloading, higher RGW CPU usage

> SSL: TRUE - RGW: SSL: FALSE INGRESS
>
> More Info: https://access.redhat.com/solutions/7655961
> https://bugzilla.redhat.com/show_bug.cgi?id=2008002

### RGW Spec

```yaml
service_type: rgw
service_id: rgw_only_pt
service_name: rgw.ssl_rgw_only_pt
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  certificate_source: cephadm-signed
  generate_cert: true
  rgw_exit_timeout_secs: 120
  rgw_frontend_port: 8043
  ssl: true
```

### Ingress Spec

```yaml
service_type: ingress
service_id: ingress.passthrough
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  backend_service: rgw.ssl_rgw_only_pt
  certificate_source: cephadm-signed
  virtual_ip: 9.11.126.77/23
  frontend_port: 448
  monitor_port: 1976
  use_tcp_mode_over_rgw: true
  ssl: false  # Passthrough mode - no SSL at ingress
```

---

## 10. Ingress Passthrough + RGW HTTPS + QAT at RGW

**Traffic Flow:** `Client --HTTPS--> Ingress (passthrough) --HTTPS--> RGW (SSL + QAT)`

End-to-end encryption with QAT acceleration at the backend RGW.

### RGW Spec

```yaml
service_type: rgw
service_id: rgw_only_pt_qat
service_name: rgw.ssl_rgw_only_pt_qat
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  qat:
    compression: hw
  certificate_source: cephadm-signed
  generate_cert: true
  rgw_exit_timeout_secs: 120
  rgw_frontend_port: 8044
  ssl: true
```

### Ingress Spec

```yaml
service_type: ingress
service_id: ingress.passthrough.two
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  backend_service: rgw.ssl_rgw_only_pt_qat
  frontend_port: 449
  virtual_ip: 9.11.126.79/23
  monitor_port: 1977
  use_tcp_mode_over_rgw: true
  ssl: false  # Passthrough mode - no SSL at ingress
```

---

## 11. Ingress Re-encrypt + RGW HTTPS

**Traffic Flow:** `Client --HTTPS--> Ingress (decrypt + re-encrypt) --HTTPS--> RGW (port 443)`

The ingress terminates the client SSL connection and establishes a **new** SSL connection to the backend RGW. This provides full encryption on both segments while allowing HAProxy to inspect and manipulate traffic.

**Important:** Include `generate_cert: true` and `ssl: true` in **both** RGW and Ingress specs, and `verify_backend_ssl_cert: true` in the ingress spec.

> More Info: https://bugzilla.redhat.com/show_bug.cgi?id=2556896

### RGW Spec

```yaml
service_type: rgw
service_id: ssl_rgw_reencrypt
service_name: rgw.ssl_rgw_reencrypt
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  certificate_source: cephadm-signed
  generate_cert: true
  rgw_exit_timeout_secs: 120
  rgw_frontend_port: 8033
  ssl: true
```

### Ingress Spec

```yaml
service_type: ingress
service_id: ssl_ingress_reencrypt
service_name: ingress.ssl_ingress_reencrypt
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  backend_service: rgw.ssl_rgw_reencrypt
  first_virtual_router_id: 56
  frontend_port: 450
  generate_cert: true
  monitor_port: 1988
  ssl: true
  verify_backend_ssl_cert: true
  virtual_ip: 9.11.126.83/23
```

---

## 12. Ingress Re-encrypt + QAT at Ingress (NOT SUPPORTED)

**Traffic Flow:** `Client --HTTPS--> Ingress (decrypt/re-encrypt + QAT) --HTTPS--> RGW`

> **NOT SUPPORTED NOW** - Due to HAProxy Image Build Issue

This scenario would use QAT at the ingress level for SSL acceleration during the re-encrypt process.

### RGW Spec

```yaml
service_type: rgw
service_id: ssl_rgw_reencrypt_one
service_name: rgw.ssl_rgw_reencrypt_one
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  certificate_source: cephadm-signed
  generate_cert: true
  rgw_exit_timeout_secs: 120
  rgw_frontend_port: 8034
  ssl: true
```

### Ingress Spec

```yaml
service_type: ingress
service_id: rgw.ssl_rgw_reencrypt_one
service_name: ingress.qat_dim_auto_vip_single
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  backend_service: rgw.ssl_rgw_reencrypt_one
  first_virtual_router_id: 57
  frontend_port: 447
  haproxy_qat_support: true
  ssl: true
  verify_backend_ssl_cert: true
  monitor_port: 1973
  virtual_ip:
    - 9.11.126.65/23
    - 9.11.106.5/23
```

---

## 13. Ingress Re-encrypt + QAT at RGW

**Traffic Flow:** `Client --HTTPS--> Ingress (decrypt/re-encrypt) --HTTPS--> RGW (SSL + QAT)`

Supported since QAT is not supported at Ingress. **Only QAT with RGW is the only valid case for Re-encrypt with QAT.** QAT operates at the RGW level for compression, and no QAT at Ingress.

- **Pros:** Double encryption, backend QAT, HA
- **Cons:** Complex, high resource usage at ingress

### RGW Spec

```yaml
service_type: rgw
service_id: ssl_rgw_reencrypt_two
service_name: rgw.ssl_rgw_reencrypt_two
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  qat:
    compression: hw
  certificate_source: cephadm-signed
  generate_cert: true
  rgw_exit_timeout_secs: 120
  rgw_frontend_port: 8035
  ssl: true
```

### Ingress Spec

```yaml
service_type: ingress
service_id: rgw.ssl_rgw_reencrypt_two
service_name: ingress.qat_dim_auto_vip_two
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  backend_service: rgw.ssl_rgw_reencrypt_two
  first_virtual_router_id: 58
  frontend_port: 451
  ssl: true
  generate_cert: true
  verify_backend_ssl_cert: true
  monitor_port: 1975
  virtual_ip:
    - 9.11.188.44/23
    - 9.11.106.6/23
```

---

## 14. Ingress Re-encrypt + QAT at RGW + QAT at Ingress (NOT SUPPORTED)

**Traffic Flow:** `Client --HTTPS--> Ingress (decrypt/re-encrypt + QAT) --HTTPS--> RGW (SSL + QAT)`

> **NOT SUPPORTED NOW** - Due to HAProxy Image Build Issue

This is the most complex scenario with QAT at both ingress and RGW. It would provide maximum security and maximum QAT acceleration.

- **Pros (if supported):** Maximum security, maximum QAT usage, HA
- **Cons:** Very complex, requires multiple QAT devices, high resource usage

### RGW Spec

```yaml
service_type: rgw
service_id: ssl_rgw_reencrypt_three
service_name: rgw.ssl_rgw_reencrypt_three
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  qat:
    compression: hw
  certificate_source: cephadm-signed
  generate_cert: true
  rgw_exit_timeout_secs: 120
  rgw_frontend_port: 8036
  ssl: true
```

### Ingress Spec

```yaml
service_type: ingress
service_id: rgw.ssl_rgw_reencrypt_three
service_name: ingress.qat_dim_auto_vip_three
placement:
  hosts:
    - ceph14
    - ceph15
    - ceph16
spec:
  backend_service: rgw.ssl_rgw_reencrypt_three
  first_virtual_router_id: 53
  frontend_port: 452
  ssl: true
  verify_backend_ssl_cert: true
  generate_cert: true
  haproxy_qat_support: true
  monitor_port: 1974
  virtual_ips_list:
    - 9.11.126.44/23
    - 9.11.106.7/23
```

---

## Quick Reference Summary

| # | Scenario | Encryption | HA | QAT | Status |
|---|----------|------------|----|-----|--------|
| 1 | RGW HTTP Only | None | No | No | Supported |
| 2 | RGW HTTP + QAT Compression | None | No | Compression | Supported |
| 3 | RGW HTTPS Only | End (RGW) | No | No | Supported |
| 4 | RGW HTTPS + QAT | End (RGW) | No | Compression | Supported |
| 5 | Ingress Terminate + RGW HTTP | Front (Ingress) | Yes | No | Supported |
| 6 | Ingress Terminate + QAT at Ingress | Front (Ingress) | Yes | SSL at Ingress | **Not Supported** |
| 7 | Ingress Terminate + QAT at RGW | Front (Ingress) | Yes | Compression at RGW | Supported |
| 8 | Ingress Terminate + QAT at Both | Front (Ingress) | Yes | Both | **Not Supported** |
| 9 | Ingress Passthrough + RGW HTTPS | End-to-End | Yes | No | Supported |
| 10 | Ingress Passthrough + RGW HTTPS + QAT | End-to-End | Yes | Compression at RGW | Supported |
| 11 | Ingress Re-encrypt + RGW HTTPS | Double (re-encrypt) | Yes | No | Supported |
| 12 | Ingress Re-encrypt + QAT at Ingress | Double (re-encrypt) | Yes | SSL at Ingress | **Not Supported** |
| 13 | Ingress Re-encrypt + QAT at RGW | Double (re-encrypt) | Yes | Compression at RGW | Supported |
| 14 | Ingress Re-encrypt + QAT at Both | Double (re-encrypt) | Yes | Both | **Not Supported** |

> **Note:** All scenarios marked "Not Supported" are blocked due to HAProxy image build issues with QAT support. Once HAProxy is compiled with QAT support, these scenarios may become available.

---

## Key Concepts

### SSL Termination Modes

- **Terminate:** HAProxy decrypts client SSL and forwards plain HTTP to backend RGW. Backend traffic is unencrypted.
- **Passthrough:** HAProxy operates in TCP mode, forwarding encrypted traffic directly to RGW without decryption. Provides true end-to-end encryption.
- **Re-encrypt:** HAProxy decrypts client SSL, then establishes a new SSL connection to backend RGW. Both segments are encrypted independently.

### QAT Usage

- **QAT at RGW:** Used for hardware-accelerated compression (`compression: hw`). Works in all supported scenarios.
- **QAT at Ingress:** Would be used for SSL offloading at HAProxy. Currently not supported due to HAProxy image build issues.

### Common Spec Parameters

| Parameter | Description |
|-----------|-------------|
| `ssl: true/false` | Enable/disable SSL on the service |
| `generate_cert: true` | Auto-generate cephadm-signed certificates |
| `certificate_source: cephadm-signed` | Use cephadm certificate authority |
| `use_tcp_mode_over_rgw: true` | Configure HAProxy in TCP passthrough mode |
| `verify_backend_ssl_cert: true` | Verify backend SSL certificate (for re-encrypt) |
| `qat: compression: hw` | Enable QAT hardware-accelerated compression at RGW |
| `qat: compression: sw` | Enable QAT software-based compression at RGW |
| `haproxy_qat_support: true` | Enable QAT support in HAProxy (not yet supported) |
| `virtual_ip` | Virtual IP for HAProxy HA/load balancing (single IP, e.g., `9.11.126.96/13`) |
| `virtual_ips_list` (list) | List of virtual IPs for multi-network HA/load balancing (e.g., `- 9.11.188.44/23`, `- 9.11.106.5/23`) |
| `first_virtual_router_id` | VRRP router ID for keepalived HA |
