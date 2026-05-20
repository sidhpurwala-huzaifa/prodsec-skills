---
name: tls-compliance
description: >
  Enforce TLS compliance for Kubernetes and OpenShift workloads. Use when
  configuring, reviewing, or auditing TLS settings on Ingress controllers,
  Routes, Services, or application endpoints for TLS version enforcement,
  certificate validation, and post-quantum readiness.
category: "secure_development"
subcategory: "kubernetes"
---

# TLS Compliance

Enforce strong TLS configuration across Kubernetes workloads to prevent protocol downgrade attacks, weak cipher exploitation, and certificate-related outages. For algorithm-level guidance (cipher selection, key sizes, post-quantum algorithms), see the [`algorithm-selection`](../algorithm-selection/SKILL.md) skill.

## TLS Version Enforcement

**Default to TLS 1.3 for all new services.** TLS 1.3 is the strongest available protocol — it mandates forward secrecy, uses only AEAD ciphers, eliminates legacy cipher negotiation, reduces handshake latency, and is inherently post-quantum ready (no static RSA key exchange).

For legacy services that cannot yet support TLS 1.3, require TLS 1.2 as the minimum. Disable TLS 1.0 and 1.1 — both are deprecated by [RFC 8996](https://datatracker.ietf.org/doc/html/rfc8996).

| Version | Status | Recommendation |
|---------|--------|----------------|
| TLS 1.3 | Current | **Default for new services** — PQC-ready, AEAD-only |
| TLS 1.2 | Acceptable | Minimum for legacy — AEAD cipher suites only |
| TLS 1.1 | Deprecated | Disable — RFC 8996 |
| TLS 1.0 | Deprecated | Disable — RFC 8996 |

### Kubernetes Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
  annotations:
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.3"
spec:
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls
```

### OpenShift Routes

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: app
spec:
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
  to:
    kind: Service
    name: app-svc
```

> **OpenShift Note:** Configure the IngressController's TLS security profile to enforce minimum versions cluster-wide:

```yaml
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: default
  namespace: openshift-ingress-operator
spec:
  tlsSecurityProfile:
    type: Modern
```

Available profiles: `Old` (TLS 1.0+), `Intermediate` (TLS 1.2+), `Modern` (TLS 1.3 only, recommended for new deployments), `Custom`.

## Centralized TLS Configuration (OpenShift)

OpenShift components must **not** hardcode TLS settings. Hardcoded TLS configuration creates a security vulnerability because it cannot be updated when the platform's security policy evolves — particularly for post-quantum cryptography readiness. This is a [release blocker for OCP 5.0](https://redhat.atlassian.net/browse/OCPSTRAT-2611).

Components must dynamically inherit TLS settings from one of three central configuration sources:

| Configuration Source | Use When |
|---------------------|----------|
| **API Server** (`apiserver.config.openshift.io`) | Default for most components |
| **Kubelet** (`kubeletconfig`) | Components running on nodes |
| **Ingress** (`ingresscontroller.operator.openshift.io`) | Components serving ingress traffic |

### Requirements for OpenShift Components

- Remove all locally hardcoded TLS protocol versions, cipher suites, and curves from code and deployment scripts
- Fetch and apply TLS policy from the appropriate central source at runtime
- Respect custom TLS profiles — customers create profiles beyond the built-in Old/Intermediate/Modern presets by starting with a base profile and disabling specific algorithms
- Do not rely on Go's default TLS settings — explicitly read and apply the configured profile
- This applies to all TLS **server** settings; client settings are not affected
- TLS configuration should apply to all CRs an operator manages, not just the operator itself

If a component has a legitimate reason to deviate from the cluster default (e.g., environment-specific constraints), it may implement its own knob with these conditions: it is clearly documented, and by default (or unset) it follows the cluster-wide configuration.

### Validation

Run [openshift/tls-scanner](https://github.com/openshift/tls-scanner) to confirm all service endpoints comply with the configured TLS policy after making changes.

## Certificate Validation

### Hostname Verification

TLS clients must verify that the server certificate's Subject Alternative Name (SAN) matches the requested hostname. Certificates with mismatched hostnames indicate misconfiguration or potential MITM.

### Certificate Chain

Servers must present the full certificate chain (leaf + intermediates). Missing intermediate certificates cause validation failures in clients that don't have them cached.

### Expiry Monitoring

Monitor certificate expiry and alert before expiration. A common threshold is 30 days for warning, 7 days for critical. Expired certificates cause hard outages with no graceful degradation.

### Self-Signed Certificates

Avoid self-signed certificates in production. For internal services, use a private CA (self-signed root issuing leaf certificates) rather than individual self-signed certificates — this enables chain validation and centralized trust management.

## Testing TLS Endpoints

Set up test endpoints to validate TLS configuration before deploying to production. This pattern uses nginx to create endpoints with specific TLS versions for verification.

### TLS 1.3 Test Endpoint

Create a self-signed certificate and an nginx ConfigMap that only accepts TLS 1.3:

```bash
openssl req -x509 -nodes -days 1 -newkey rsa:2048 \
  -keyout /tmp/tls.key -out /tmp/tls.crt \
  -subj "/CN=tls-test.tls-test-ns.svc"

kubectl create namespace tls-test-ns

kubectl create secret tls tls-test-cert \
  --cert=/tmp/tls.crt --key=/tmp/tls.key \
  -n tls-test-ns

kubectl create configmap nginx-tls13-only -n tls-test-ns --from-literal=default.conf='
server {
    listen 8443 ssl;
    ssl_certificate /etc/tls/tls.crt;
    ssl_certificate_key /etc/tls/tls.key;
    ssl_protocols TLSv1.3;
    location / { return 200 "tls13-only"; }
}'
```

Deploy the test workload:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tls13-only
  namespace: tls-test-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tls13-only
  template:
    metadata:
      labels:
        app: tls13-only
    spec:
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:1.27
          ports:
            - containerPort: 8443
          volumeMounts:
            - name: tls
              mountPath: /etc/tls
              readOnly: true
            - name: conf
              mountPath: /etc/nginx/conf.d
      volumes:
        - name: tls
          secret:
            secretName: tls-test-cert
        - name: conf
          configMap:
            name: nginx-tls13-only
---
apiVersion: v1
kind: Service
metadata:
  name: tls13-only-svc
  namespace: tls-test-ns
spec:
  selector:
    app: tls13-only
  ports:
    - port: 443
      targetPort: 8443
```

### Verify the Endpoint

From a pod in the cluster, confirm only TLS 1.3 is accepted:

```bash
# Should succeed (TLS 1.3)
openssl s_client -connect tls13-only-svc.tls-test-ns.svc:443 -tls1_3

# Should fail (TLS 1.2 rejected)
openssl s_client -connect tls13-only-svc.tls-test-ns.svc:443 -tls1_2
```

## Post-Quantum TLS Readiness

**The most impactful PQC preparation step is adopting TLS 1.3 now.** TLS 1.3 eliminates static RSA key exchange (the primary quantum-vulnerable pattern) and mandates forward secrecy, meaning even if a server's long-term key is broken by a future quantum computer, past session keys cannot be recovered.

"Harvest now, decrypt later" attacks make long-lived data vulnerable today — encrypted traffic captured now could be decrypted once quantum computers mature.

TLS 1.3 supports Hybrid ML-KEM key encapsulation when both endpoints have it enabled. Clusters using the Intermediate profile (TLS 1.2 minimum) do not prevent this — clients capable of TLS 1.3 will still negotiate it when the server offers it. This means PQC-resilient key exchange can work today without requiring the Modern (TLS 1.3 only) profile.

### Assessment Steps

1. **Adopt TLS 1.3 for all new services** — the single highest-impact action for PQC readiness
2. **Ensure centralized TLS configuration** — components that dynamically inherit TLS settings can adopt PQC ciphers platform-wide through a single configuration change
3. **Audit existing endpoints** — identify services still using TLS 1.2 with static RSA or non-AEAD ciphers
4. **Identify long-lived secrets** — data encrypted today that must remain confidential for 10+ years is at risk
5. **Monitor hybrid key exchange** — TLS 1.3 with X25519Kyber768 is available in some implementations; track adoption in your ingress controllers and load balancers
6. **Follow NIST PQC standards** — ML-KEM (Kyber), ML-DSA (Dilithium), and SLH-DSA (SPHINCS+) are finalized; watch for upstream OpenSSL/Go crypto adoption

For detailed algorithm selection and post-quantum algorithm guidance, see [`algorithm-selection`](../algorithm-selection/SKILL.md).

## Scanning Tools

- [openshift/tls-scanner](https://github.com/openshift/tls-scanner) — the official OpenShift TLS scanning tool for auditing endpoint TLS versions and cipher suites across the cluster; required for [OCPSTRAT-2611](https://redhat.atlassian.net/browse/OCPSTRAT-2611) compliance validation
- [tls-compliance-operator](https://github.com/sebrandon1/tls-compliance-operator) — community project providing continuous TLS compliance monitoring with post-quantum readiness visibility via a kubectl plugin

## Certsuite Test Mapping

| Guidance | Certsuite Test ID | Profiles |
|----------|-------------------|----------|
| TLS version enforcement | [`networking-tls-minimum-version`](https://github.com/redhat-best-practices-for-k8s/certsuite/pull/3456) | Telco/Far-Edge/Extended: mandatory, Non-Telco: optional |

## Implementation Checklist

- [ ] TLS 1.3 is the default for all new services
- [ ] Legacy services use TLS 1.2 as the minimum
- [ ] TLS 1.0 and 1.1 are disabled on all endpoints
- [ ] No hardcoded TLS settings — configuration inherited from API Server, Kubelet, or Ingress
- [ ] Custom TLS profiles are respected (not just Old/Intermediate/Modern)
- [ ] tls-scanner confirms endpoint compliance with configured TLS policy
- [ ] Server certificates have correct SANs matching all served hostnames
- [ ] Full certificate chain is served (leaf + intermediates)
- [ ] Certificate expiry is monitored with alerts at 30 and 7 days
- [ ] Self-signed certificates are not used in production (use private CA instead)
- [ ] TLS test endpoints validate configuration before production deployment
- [ ] Post-quantum readiness audit completed for long-lived data

## References

- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
- [NIST SP 800-52 Rev. 2 — TLS Implementation Guidelines](https://csrc.nist.gov/publications/detail/sp/800-52/rev-2/final)
- [RFC 8996 — Deprecating TLS 1.0 and TLS 1.1](https://datatracker.ietf.org/doc/html/rfc8996)
- [OpenShift TLS Security Profiles](https://docs.openshift.com/container-platform/latest/security/tls-security-profiles.html)
- [OCPSTRAT-2611 — Centralized TLS Configuration for OpenShift](https://redhat.atlassian.net/browse/OCPSTRAT-2611)
- [NIST Post-Quantum Cryptography Standards](https://csrc.nist.gov/projects/post-quantum-cryptography)
