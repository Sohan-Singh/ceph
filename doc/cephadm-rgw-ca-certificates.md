# Getting Cephadm CA PEM Keys for Ceph RGW

This guide explains how to extract and use cephadm-generated CA certificates for Ceph RGW (RADOS Gateway) when SSL/TLS is enabled.

## Table of Contents

- [Overview](#overview)
- [Understanding Cephadm Certificate Management](#understanding-cephadm-certificate-management)
- [Prerequisites](#prerequisites)
- [Extracting Certificates](#extracting-certificates)
- [Using Certificates with AWS CLI](#using-certificates-with-aws-cli)
- [Troubleshooting](#troubleshooting)
- [Security Considerations](#security-considerations)

## Overview

When Ceph RGW is deployed with SSL/TLS using cephadm, it uses certificates signed by a cephadm-managed Certificate Authority (CA). This guide shows how to:

1. Extract the cephadm root CA certificate
2. Extract RGW service certificates
3. Use these certificates with S3-compatible clients like AWS CLI

## Understanding Cephadm Certificate Management

### Certificate Storage

Cephadm stores certificates in the Ceph config-key store under the `mgr/cephadm/cert_store.*` prefix:

- **Root CA Certificate**: `mgr/cephadm/cert_store.cert.cephadm_root_ca_cert`
- **Root CA Key**: `mgr/cephadm/cert_store.key.cephadm_root_ca_key`
- **Service Certificates**: `mgr/cephadm/cert_store.cert.cephadm-signed_<service>_cert`
- **Service Keys**: `mgr/cephadm/cert_store.key.cephadm-signed_<service>_key`

### Storage Format

Certificates are stored in JSON format with metadata:

```json
{
  "cert": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----\n",
  "user_made": false,
  "editable": false
}
```

For per-host certificates (like RGW), the format is:

```json
{
  "hostname1": {
    "cert": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----\n",
    "user_made": false,
    "editable": false
  },
  "hostname2": { ... }
}
```

### Certificate Hierarchy

```
┌─────────────────────────────────────┐
│   Cephadm Root CA Certificate       │
│   (Self-signed, 10-year validity)   │
│   CN=cephadm-root-<fsid>            │
└──────────────┬──────────────────────┘
               │ Signs
               ├──────────────────────────────┐
               │                              │
               ▼                              ▼
┌──────────────────────────┐    ┌──────────────────────────┐
│  RGW Certificates        │    │  Other Service Certs     │
│  (3-year validity)       │    │  (Grafana, Prometheus,   │
│  Per-host certificates   │    │   Alertmanager, etc.)    │
└──────────────────────────┘    └──────────────────────────┘
```

## Prerequisites

- Access to a Ceph cluster with cephadm
- `jq` command-line JSON processor installed
- Appropriate permissions to run `ceph` commands

Install `jq` if not available:

```bash
# RHEL/CentOS/Rocky
sudo dnf install jq

# Ubuntu/Debian
sudo apt-get install jq
```

## Extracting Certificates

### Step 1: List Available Certificates

First, verify what certificates are available:

```bash
ceph config-key ls | grep -E "cert|cephadm"
```

### Step 2: Extract Root CA Certificate

The root CA certificate is the trust anchor for all cephadm-signed certificates:

```bash
# Extract root CA certificate
ceph config-key get mgr/cephadm/cert_store.cert.cephadm_root_ca_cert | jq -r '.cert' > cephadm_root_ca.pem

# Extract root CA private key (if needed)
ceph config-key get mgr/cephadm/cert_store.key.cephadm_root_ca_key | jq -r '.key' > cephadm_root_ca_key.pem
```

### Step 3: Extract RGW Certificates

RGW certificates are stored per-host. First, check the structure:

```bash
# View the certificate structure
ceph config-key get mgr/cephadm/cert_store.cert.cephadm-signed_rgw.<service-name>_cert | jq
```

Extract certificates for each host:

```bash
# Replace <service-name> with your RGW service name (e.g., ssl_rgw_qat_hw)
# Replace <hostname> with your actual hostnames (e.g., ceph14, ceph15, ceph16)

# Extract certificate for host1
ceph config-key get mgr/cephadm/cert_store.cert.cephadm-signed_rgw.<service-name>_cert | \
  jq -r '.<hostname1>.cert' > rgw_<hostname1>_cert.pem

# Extract private key for host1
ceph config-key get mgr/cephadm/cert_store.key.cephadm-signed_rgw.<service-name>_key | \
  jq -r '.<hostname1>.key' > rgw_<hostname1>_key.pem
```

**Example** for a service named `ssl_rgw_qat_hw` with hosts `ceph14`, `ceph15`, `ceph16`:

```bash
# ceph14
ceph config-key get mgr/cephadm/cert_store.cert.cephadm-signed_rgw.ssl_rgw_qat_hw_cert | \
  jq -r '.ceph14.cert' > rgw_ceph14_cert.pem
ceph config-key get mgr/cephadm/cert_store.key.cephadm-signed_rgw.ssl_rgw_qat_hw_key | \
  jq -r '.ceph14.key' > rgw_ceph14_key.pem

# ceph15
ceph config-key get mgr/cephadm/cert_store.cert.cephadm-signed_rgw.ssl_rgw_qat_hw_cert | \
  jq -r '.ceph15.cert' > rgw_ceph15_cert.pem
ceph config-key get mgr/cephadm/cert_store.key.cephadm-signed_rgw.ssl_rgw_qat_hw_key | \
  jq -r '.ceph15.key' > rgw_ceph15_key.pem

# ceph16
ceph config-key get mgr/cephadm/cert_store.cert.cephadm-signed_rgw.ssl_rgw_qat_hw_cert | \
  jq -r '.ceph16.cert' > rgw_ceph16_cert.pem
ceph config-key get mgr/cephadm/cert_store.key.cephadm-signed_rgw.ssl_rgw_qat_hw_key | \
  jq -r '.ceph16.key' > rgw_ceph16_key.pem
```

### Step 4: Verify Certificates

Verify that the RGW certificates are signed by the cephadm root CA:

```bash
# Verify certificate chain
openssl verify -CAfile cephadm_root_ca.pem rgw_<hostname>_cert.pem

# View certificate details
openssl x509 -in rgw_<hostname>_cert.pem -text -noout

# Check certificate expiration
openssl x509 -in rgw_<hostname>_cert.pem -noout -dates
```

## Using Certificates with AWS CLI

### Method 1: Using --ca-bundle (Recommended)

Use the `--ca-bundle` option to specify the CA certificate for each command:

```bash
# Configure AWS CLI profile
aws configure --profile ceph-rgw
# Enter your RGW access key, secret key, region, and output format

# List buckets
aws --profile ceph-rgw \
    --endpoint-url https://<rgw-host>:<port> \
    --ca-bundle cephadm_root_ca.pem \
    s3 ls

# Create bucket
aws --profile ceph-rgw \
    --endpoint-url https://<rgw-host>:<port> \
    --ca-bundle cephadm_root_ca.pem \
    s3 mb s3://my-bucket

# Upload file
aws --profile ceph-rgw \
    --endpoint-url https://<rgw-host>:<port> \
    --ca-bundle cephadm_root_ca.pem \
    s3 cp myfile.txt s3://my-bucket/

# List objects
aws --profile ceph-rgw \
    --endpoint-url https://<rgw-host>:<port> \
    --ca-bundle cephadm_root_ca.pem \
    s3 ls s3://my-bucket/

# Download file
aws --profile ceph-rgw \
    --endpoint-url https://<rgw-host>:<port> \
    --ca-bundle cephadm_root_ca.pem \
    s3 cp s3://my-bucket/myfile.txt ./
```

### Method 2: Add CA to System Trust Store

For permanent trust, add the CA certificate to your system's trust store:

**RHEL/CentOS/Rocky Linux:**

```bash
# Copy CA certificate
sudo cp cephadm_root_ca.pem /etc/pki/ca-trust/source/anchors/cephadm-root-ca.crt

# Update CA trust
sudo update-ca-trust

# Now use AWS CLI without --ca-bundle
aws --profile ceph-rgw --endpoint-url https://<rgw-host>:<port> s3 ls
```

**Ubuntu/Debian:**

```bash
# Copy CA certificate
sudo cp cephadm_root_ca.pem /usr/local/share/ca-certificates/cephadm-root-ca.crt

# Update CA trust
sudo update-ca-certificates

# Now use AWS CLI without --ca-bundle
aws --profile ceph-rgw --endpoint-url https://<rgw-host>:<port> s3 ls
```

### Method 3: Using Environment Variables

Set environment variables to avoid repeating options:

```bash
# Set AWS credentials and CA bundle
export AWS_ACCESS_KEY_ID=<your-access-key>
export AWS_SECRET_ACCESS_KEY=<your-secret-key>
export AWS_CA_BUNDLE=/path/to/cephadm_root_ca.pem

# Now use AWS CLI without credentials or --ca-bundle
aws --endpoint-url https://<rgw-host>:<port> s3 ls
```

### Creating RGW Users

If you need to create RGW users with access keys:

```bash
# Create user
radosgw-admin user create \
    --uid=myuser \
    --display-name="My User" \
    --access-key=MYACCESSKEY123 \
    --secret-key=MYSECRETKEY456

# Get user information
radosgw-admin user info --uid=myuser

# List all users
radosgw-admin user list
```

## Troubleshooting

### Certificate Verification Errors

**Error**: `SSL certificate problem: unable to get local issuer certificate`

**Solution**: Ensure you're using the correct CA certificate with `--ca-bundle` or add it to the system trust store.

### Certificate Not Found

**Error**: `Error ENOENT` when trying to get certificate

**Possible causes**:
1. Certificate key name is incorrect
2. Service hasn't been deployed yet
3. SSL is not enabled for the service

**Solution**: List all certificates to find the correct key:

```bash
ceph config-key ls | grep cert_store
```

### JSON Parsing Errors

**Error**: `parse error: Invalid numeric literal`

**Solution**: Ensure you're using `jq -r` (raw output) to extract the certificate:

```bash
# Correct
ceph config-key get <key> | jq -r '.cert'

# Incorrect (includes JSON quotes)
ceph config-key get <key> | jq '.cert'
```

### Per-Host Certificate Structure

If `jq -r '.cert'` returns `null`, the certificate might be stored per-host:

```bash
# Check structure
ceph config-key get <key> | jq

# Extract for specific host
ceph config-key get <key> | jq -r '.<hostname>.cert'
```

### Certificate Expiration

Check certificate expiration dates:

```bash
# Check root CA expiration (typically 10 years)
openssl x509 -in cephadm_root_ca.pem -noout -dates

# Check RGW certificate expiration (typically 3 years)
openssl x509 -in rgw_cert.pem -noout -dates
```

Cephadm automatically rotates certificates before expiration if `certificate_automated_rotation_enabled` is set to `true` (default).

## Security Considerations

### Best Practices

1. **Protect Private Keys**: Keep private keys secure and restrict access
   ```bash
   chmod 600 rgw_*_key.pem
   ```

2. **Use System Trust Store**: For production, add CA to system trust store instead of using `--ca-bundle`

3. **Regular Audits**: Periodically check certificate status
   ```bash
   ceph orch certmgr cert ls
   ```

4. **Monitor Expiration**: Set up monitoring for certificate expiration
   ```bash
   ceph health detail | grep CERT
   ```

5. **Backup Certificates**: Backup the root CA certificate and key securely

### Certificate Rotation

Cephadm automatically handles certificate rotation when enabled:

```bash
# Check if automatic rotation is enabled
ceph config get mgr mgr/cephadm/certificate_automated_rotation_enabled

# Enable automatic rotation (default)
ceph config set mgr mgr/cephadm/certificate_automated_rotation_enabled true

# Check certificate health
ceph orch certmgr cert ls --status expiring
```

### Disabling SSL Verification (NOT RECOMMENDED)

For testing only, you can disable SSL verification:

```bash
# AWS CLI
aws --no-verify-ssl --endpoint-url https://<rgw-host>:<port> s3 ls

# curl
curl -k https://<rgw-host>:<port>
```

**Warning**: Never use `--no-verify-ssl` or `-k` in production as it disables security checks.

## Additional Resources

- [Cephadm Documentation](https://docs.ceph.com/en/latest/cephadm/)
- [RGW Documentation](https://docs.ceph.com/en/latest/radosgw/)
- [AWS CLI S3 Commands](https://docs.aws.amazon.com/cli/latest/reference/s3/)
- [OpenSSL Certificate Verification](https://www.openssl.org/docs/man1.1.1/man1/verify.html)

## Summary

This guide covered:

- ✅ Understanding cephadm certificate storage and structure
- ✅ Extracting root CA and service certificates from config-key store
- ✅ Converting JSON-wrapped certificates to PEM format
- ✅ Using certificates with AWS CLI and other S3 clients
- ✅ Adding certificates to system trust store
- ✅ Troubleshooting common certificate issues
- ✅ Security best practices for certificate management

For questions or issues, consult the Ceph documentation or community forums.