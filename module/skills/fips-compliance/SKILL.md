---
name: fips-compliance
description: >
  Enforce FIPS 140-2/140-3 compliance for RHEL, OpenShift, and Go workloads.
  Use when building, configuring, reviewing, or auditing systems that require
  FIPS-validated cryptographic modules, RHEL crypto-policy enforcement,
  FIPS-ready Go binaries, or FIPS-mode kernel and cluster configuration.
category: "secure_development"
subcategory: "crypto"
---

# FIPS Compliance

Guidance for enforcing Federal Information Processing Standards (FIPS) 140-2 and 140-3 across RHEL/RHCOS nodes, OpenShift clusters, Go applications, and container workloads. For general algorithm guidance and post-quantum readiness, see [`algorithm-selection`](../algorithm-selection/SKILL.md). For TLS version and cipher suite enforcement on Kubernetes, see [`tls-compliance`](../tls-compliance/SKILL.md).

## FIPS Mode Enforcement

| Layer | Mechanism | Validation Command |
|-------|-----------|-------------------|
| RHCOS/RHEL kernel | `fips=1` boot parameter | `cat /proc/sys/crypto/fips_enabled` returns `1` |
| RHEL crypto policy | `update-crypto-policies --set FIPS` | `update-crypto-policies --show` returns `FIPS` |
| OpenShift cluster | `fips: true` in `install-config.yaml` | `oc get cm -n openshift-config -o jsonpath='{.items[*].data.install-config}'` |
| etcd encryption | `aescbc` or `aesgcm` encryption type | See [etcd Encryption at Rest](#etcd-encryption-at-rest) verification commands |

> **Critical:** OpenShift FIPS mode must be enabled at install time — it cannot be enabled post-deployment. The installer must run from a RHEL host already in FIPS mode, using the `openshift-install-fips` binary extracted from the release image.

> **SSH keys:** Do not use ed25519 keys when installing a FIPS-enabled cluster. ed25519 is not FIPS-approved — use RSA instead.

## FIPS-Approved and Prohibited Algorithms

| Category | Approved | Prohibited in FIPS Mode |
|----------|----------|------------------------|
| Symmetric encryption | AES-128, AES-192, AES-256 (GCM, CBC, CCM) | ChaCha20-Poly1305, RC4, Blowfish, DES, 3DES |
| Hashing | SHA-256, SHA-384, SHA-512, SHA-3 | MD5, SHA-1 (for signatures/integrity) |
| Asymmetric | RSA >= 2048, ECDSA (P-256, P-384, P-521) | RSA < 2048, non-NIST curves |
| Key exchange | ECDHE (P-256, P-384), DH >= 2048 | X25519 (not FIPS-validated), static RSA |
| MAC | HMAC-SHA-256, HMAC-SHA-384 | Poly1305 |
| Post-quantum | None currently FIPS-validated | ML-KEM, ML-DSA, X25519MLKEM768 (awaiting NIST validation) |

> **FIPS and PQC are mutually exclusive today.** Post-quantum algorithms like X25519MLKEM768 cannot be used on FIPS-enabled clusters because they have not been submitted to NIST for FIPS validation. Plan for this to change as NIST completes PQC module validation.

## RHEL Crypto Policies

RHEL system-wide crypto policies enforce FIPS constraints across all crypto libraries (OpenSSL, GnuTLS, NSS, libgcrypt, Kernel Crypto API).

```bash
# Set FIPS policy
update-crypto-policies --set FIPS

# Verify active policy
update-crypto-policies --show
```

### Subpolicies

Apply subpolicies on top of FIPS for additional restrictions or compatibility workarounds:

| Subpolicy | Command | Effect |
|-----------|---------|--------|
| `NO-SHA1` | `--set FIPS:NO-SHA1` | Disables SHA-1 in all signatures and certificates |
| `NO-ENFORCE-EMS` | `--set FIPS:NO-ENFORCE-EMS` | Relaxes mandatory TLS Extended Master Secret (see [TLS in FIPS Mode](#tls-in-fips-mode)) |
| `OSPP` | `--set FIPS:OSPP` | Common Criteria restrictions — RSA/DH >= 3072 bits |
| `NO-CAMELLIA` | `--set FIPS:NO-CAMELLIA` | Removes Camellia ciphers |

### Custom Subpolicies

Create `.pmod` files in `/etc/crypto-policies/policies/modules/` for site-specific restrictions:

```ini
# /etc/crypto-policies/policies/modules/SITE.pmod
cipher@SSH = -CHACHA20-POLY1305
ssh_etm = 0
```

```bash
update-crypto-policies --set FIPS:SITE
```

## Go FIPS Builds

Two distinct paths exist for building FIPS-compliant Go binaries. Choose based on your environment.

### Red Hat Go Toolset (RHEL)

Red Hat's Go toolset routes crypto calls to the FIPS-validated OpenSSL module via CGO — it does **not** use Google's BoringCrypto.

| Constraint | Requirement |
|-----------|-------------|
| `CGO_ENABLED` | Must be `1` (default) — setting to `0` disables OpenSSL linkage |
| Static linking | **Prohibited** — `-extldflags "-static"` breaks `dlopen` for OpenSSL |
| `-tags no_openssl` | **Prohibited** — disables the OpenSSL integration |
| Base image | Must include OpenSSL (UBI images include it) |
| FIPS detection | Automatic via `/proc/sys/crypto/fips_enabled` at runtime |

```go
import "github.com/golang-fips/openssl/v2"

func init() {
    openssl.Init("")
}

// Check FIPS status at runtime
if openssl.FIPS() {
    // Using FIPS-validated OpenSSL path
}
```

```bash
# Verify binary links to OpenSSL
go tool nm ./myapp | grep -i dlopen_openssl
```

### Upstream Go 1.24+ (Native FIPS 140-3)

Go 1.24 introduced native FIPS 140-3 support with CMVP Certificate #5247, independent of Red Hat's OpenSSL approach.

| Setting | Options |
|---------|---------|
| `GOFIPS140` (build-time) | `off` (default), `latest`, `v1.0.0`, `certified`, `inprocess` |
| `GODEBUG=fips140` (runtime) | `off` (default), `on`, `only` (non-production — panics on non-FIPS calls) |
| API | `crypto/fips140.Enabled()`, `crypto/fips140.Version()` |

When FIPS mode is active, Go performs integrity self-checks at init, runs known-answer self-tests per FIPS 140-3 Implementation Guidance, and restricts `crypto/tls` to approved cipher suites and key exchange mechanisms.

### Common Violations

| Violation | Why It Breaks FIPS | Fix |
|-----------|-------------------|-----|
| `CGO_ENABLED=0` | No OpenSSL linkage (Red Hat path) | Set `CGO_ENABLED=1` |
| `-extldflags "-static"` | Prevents `dlopen` of OpenSSL | Use dynamic linking |
| `-tags no_openssl` | Disables OpenSSL integration | Remove the tag |
| Importing `golang.org/x/crypto` | Not FIPS-validated — may bypass validated modules | Use standard `crypto/*` packages |
| Using `crypto/md5` for integrity | MD5 is not FIPS-approved for security purposes | Use `crypto/sha256` |
| ChaCha20-Poly1305 cipher suites | Not FIPS-approved | Use AES-GCM cipher suites |

### Historical: CVE-2023-3089

OpenShift 4.10–4.12 Go components were not configured to use FIPS-validated crypto modules, defaulting to Go's standard crypto library instead of OpenSSL. Approximately 50% of certificates on affected clusters were produced by non-compliant modules. Remediation required updating to patched releases and rotating certificates (or full reinstall for the most regulated environments). Red Hat has since added build-time enforcement policies that terminate builds using non-compliant crypto.

## TLS in FIPS Mode

### FIPS-Approved Cipher Suites

FIPS mode filters out non-approved ciphers. On OpenShift, the Ingress Operator automatically detects FIPS mode and removes non-compliant ciphers.

**TLS 1.2 (FIPS-approved):**

| Cipher Suite | Key Exchange | Encryption |
|-------------|--------------|------------|
| `TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256` | ECDHE | AES-128-GCM |
| `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256` | ECDHE | AES-128-GCM |
| `TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384` | ECDHE | AES-256-GCM |
| `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384` | ECDHE | AES-256-GCM |

**TLS 1.3 (FIPS-approved):**

| Cipher Suite | FIPS Status |
|-------------|-------------|
| `TLS_AES_128_GCM_SHA256` | Approved |
| `TLS_AES_256_GCM_SHA384` | Approved |
| `TLS_CHACHA20_POLY1305_SHA256` | **Filtered out** on FIPS clusters |

### Extended Master Secret (EMS) Enforcement

Starting with RHEL 9.2, TLS Extended Master Secret ([RFC 7627](https://datatracker.ietf.org/doc/html/rfc7627)) is **mandatory** for all TLS 1.2 connections in FIPS mode. This is a FIPS 140-3 requirement.

**Breaking change:** Legacy clients that do not support EMS or TLS 1.3 (including RHEL 6/7 and Go <= 1.18) **cannot connect** to FIPS-enabled RHEL 9.2+ systems over TLS 1.2.

Workaround (non-compliant — use only during migration):

```bash
update-crypto-policies --set FIPS:NO-ENFORCE-EMS
```

## etcd Encryption at Rest

| Encryption Type | Algorithm | Status |
|----------------|-----------|--------|
| `aescbc` | AES-CBC with PKCS#7 padding, 32-byte key | Supported |
| `aesgcm` | AES-GCM | Supported |
| `identity` | No encryption | **Not compliant** — secrets stored in plaintext |

### Verification

```bash
# OpenShift API server
oc get openshiftapiserver -o=jsonpath='{range .items[0].status.conditions[?(@.type=="Encrypted")]}{.reason}{"\n"}{.message}{"\n"}'

# Kubernetes API server
oc get kubeapiserver -o=jsonpath='{range .items[0].status.conditions[?(@.type=="Encrypted")]}{.reason}{"\n"}{.message}{"\n"}'

# OAuth API server
oc get authentication.operator.openshift.io -o=jsonpath='{range .items[0].status.conditions[?(@.type=="Encrypted")]}{.reason}{"\n"}{.message}{"\n"}'
```

Expected output: `EncryptionCompleted`. Keys are rotated automatically on a weekly basis.

> **Limitation:** Encryption keys are stored in plaintext on the nodes hosting the API server, co-located with etcd. This provides protection for etcd backups and offline disk access, but limited benefit against node-level compromise.

## Operator and Workload Considerations

- **cert-manager:** Certificate resources must specify FIPS-compliant key algorithms — RSA >= 2048 or ECDSA P-256/P-384 via `spec.privateKey.algorithm` and `spec.privateKey.size`. Do not use ed25519.
- **Service mesh:** mTLS cipher suites must use FIPS-approved algorithms only. Post-quantum key exchange (X25519MLKEM768) is **not available** on FIPS-enabled clusters. Ambient mode (ztunnel) supports FIPS 140-2 with TLS 1.2.
- **Container images:** Use FIPS-enabled UBI base images (`registry.access.redhat.com/ubi9/ubi-minimal`). The Red Hat Go toolchain must be used for the build stage. OpenSSL must be present in the runtime image.
- **Containers inherit host FIPS:** Containers detect FIPS mode via `/proc/sys/crypto/fips_enabled` from the host kernel — no per-container FIPS configuration is needed.
- **FIPS validation terminology:** Use "FIPS validated" (applies to cryptographic modules). "FIPS compliant" and "FIPS certified" are incorrect terms per NIST.

## FIPS Validation Status

Red Hat submits five cryptographic modules per RHEL release for NIST CMVP validation:

| Module | Purpose |
|--------|---------|
| OpenSSL | Primary crypto library (TLS, certificates, Go crypto) |
| Kernel Crypto API | Kernel-level encryption (dm-crypt, IPsec) |
| GnuTLS | Alternative TLS library |
| NSS | Network Security Services (Firefox, certificate tools) |
| libgcrypt | GnuPG and other applications |

| RHEL Version | FIPS Standard | Validation Deadline |
|-------------|---------------|-------------------|
| RHEL 8.x | FIPS 140-2 | Active through September 21, 2026 |
| RHEL 9.0+ | FIPS 140-3 | Current standard |
| RHEL 10 | FIPS 140-3 only | Reuses RHEL 9 OpenSSL module |

## Auditing FIPS Compliance

```bash
# Verify all nodes have FIPS kernel enabled
oc debug node/<node> -- chroot /host cat /proc/sys/crypto/fips_enabled

# Check crypto policy on nodes
oc debug node/<node> -- chroot /host update-crypto-policies --show

# Verify FIPS boot parameter
oc debug node/<node> -- chroot /host grep -o 'fips=[01]' /proc/cmdline

# Check IngressController TLS profile (should not include ChaCha20 on FIPS clusters)
oc get ingresscontroller default -n openshift-ingress-operator -o jsonpath='{.spec.tlsSecurityProfile}'

# Verify etcd encryption status
oc get kubeapiserver -o=jsonpath='{range .items[0].status.conditions[?(@.type=="Encrypted")]}{.reason}{"\n"}'

# Verify Go binary uses OpenSSL (Red Hat Go toolset)
go tool nm <binary> | grep -i dlopen_openssl

# Check kernel FIPS self-test results
oc debug node/<node> -- chroot /host journalctl -b | grep -i fips
```

## Implementation Checklist

- [ ] Cluster installed with `fips: true` in install-config from a FIPS-enabled RHEL host
- [ ] All nodes report `fips_enabled = 1` via `/proc/sys/crypto/fips_enabled`
- [ ] Crypto policy set to `FIPS` (verify with `update-crypto-policies --show`)
- [ ] Go binaries built with `CGO_ENABLED=1` and dynamic linking (Red Hat path) or `GOFIPS140=latest` (upstream Go 1.24+)
- [ ] No static linking, `no_openssl` tags, or `golang.org/x/crypto` usage in FIPS-critical paths
- [ ] etcd encryption at rest enabled (`aescbc` or `aesgcm`) — not `identity`
- [ ] TLS cipher suites restricted to FIPS-approved algorithms (no ChaCha20-Poly1305)
- [ ] EMS enforcement verified for TLS 1.2 connections (RHEL 9.2+) or `NO-ENFORCE-EMS` applied with documented justification
- [ ] cert-manager Certificates use RSA >= 2048 or ECDSA P-256/P-384 — no ed25519
- [ ] Container base images are FIPS-enabled UBI with OpenSSL present
- [ ] SSH keys use RSA — no ed25519
- [ ] OCP 4.10–4.12 clusters have applied CVE-2023-3089 remediation and rotated certificates

## References

- [OpenShift FIPS Support Documentation](https://docs.openshift.com/container-platform/latest/installing/overview/installing-fips.html)
- [NIST CMVP Validated Modules List](https://csrc.nist.gov/projects/cryptographic-module-validation-program/validated-modules)
- [RHEL 9 Security Hardening — Crypto Policies](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/security_hardening/using-the-system-wide-cryptographic-policies_security-hardening)
- [Go FIPS 140-3 Security Documentation](https://go.dev/doc/security/fips140)
- [Red Hat Go Toolset FIPS Mode](https://developers.redhat.com/articles/2025/01/23/fips-mode-red-hat-go-toolset)
- [Is Your Go Application FIPS Compliant?](https://developers.redhat.com/articles/2022/05/31/your-go-application-fips-compliant)
- [RHSB-2023-001: CVE-2023-3089 — OpenShift FIPS Misconfiguration](https://access.redhat.com/security/vulnerabilities/RHSB-2023-001)
- [TLS Extended Master Secret and FIPS in RHEL](https://www.redhat.com/en/blog/tls-extended-master-secret-and-fips-rhel)
- [RHEL Core Cryptographic Components](https://access.redhat.com/articles/3655361)
- [OpenShift FIPS Compliance FAQ](https://access.redhat.com/articles/openshift_fips_compliance_faq)
