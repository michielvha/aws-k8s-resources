# Kyverno setup

I had to use ``kubectl apply -f template.yaml --server-side`` to get the Kyverno app installed.

Something about the policy and clusterPolicy crd was causing the following error without server-side apply:

```log
Error from server (Invalid): error when creating ".\\template.yaml": CustomResourceDefinition.apiextensions.k8s.io "policies.kyverno.io" is invalid: metadata.annotations: Too long: may not be more than 262144 bytes
```

### Helm Chart

manually adding the kyverno helm chart to the repo and pulling it down.

```bash
helm repo add kyverno https://kyverno.github.io/kyverno && \
helm pull kyverno/kyverno --version 3.3.1 --untar --untardir /tmp
```

## Troubleshooting


When the cert for the helm repo is not recognized by the cluster, you can add the CA to argocd after extracting it with openssl.

````bash
  openssl s_client -showcerts \
    -connect kyverno.github.io:443 \
    -servername kyverno.github.io \
    -verify_return_error </dev/null | tee /tmp/kyverno.certlog
    
# Still inside the repo‑server pod …
# 1) Grab just the self‑signed root into a file
openssl s_client -connect kyverno.github.io:443 \
  -servername kyverno.github.io -showcerts </dev/null \
| awk '/-----BEGIN CERTIFICATE-----/{n++}
       n==2{print}
       /-----END CERTIFICATE-----/{if(n==2)exit}' \
> /tmp/fortinet-root.pem

# Optional: check what we captured
openssl x509 -in /tmp/fortinet-root.pem -noout -subject -issuer -fingerprint

````

add custom CA to the cluster

ofcourse this will be added to the git repo, below is just to see what we should do.

```bash
# assuming /tmp/fortinet-root.pem has been copied here
kubectl -n argocd create configmap argocd-tls-certs-cm \
  --from-file=kyverno.github.io=fortinet-root.pem \
  --dry-run=client -o yaml | kubectl apply -f -
```

outcome:

````yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-tls-certs-cm
  namespace: argocd
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: argocd-tls-certs-cm
    app.kubernetes.io/part-of: argocd
    app.kubernetes.io/instance: argocd
data:
  kyverno.github.io: |
    -----BEGIN CERTIFICATE-----
    MIID5jCCAs6gAwIBAgIIahK5ESQ1PKYwDQYJKoZIhvcNAQELBQAwgakxCzAJBgNV
    BAYTAlVTMRMwEQYDVQQIDApDYWxpZm9ybmlhMRIwEAYDVQQHDAlTdW5ueXZhbGUx
    ETAPBgNVBAoMCEZvcnRpbmV0MR4wHAYDVQQLDBVDZXJ0aWZpY2F0ZSBBdXRob3Jp
    dHkxGTAXBgNVBAMMEEZHVk1FTFRNMjQwMDQyOTIxIzAhBgkqhkiG9w0BCQEWFHN1
    cHBvcnRAZm9ydGluZXQuY29tMB4XDTI0MDQxOTIxMDMzN1oXDTM0MDQyMDIxMDMz
    N1owgakxCzAJBgNVBAYTAlVTMRMwEQYDVQQIDApDYWxpZm9ybmlhMRIwEAYDVQQH
    DAlTdW5ueXZhbGUxETAPBgNVBAoMCEZvcnRpbmV0MR4wHAYDVQQLDBVDZXJ0aWZp
    Y2F0ZSBBdXRob3JpdHkxGTAXBgNVBAMMEEZHVk1FTFRNMjQwMDQyOTIxIzAhBgkq
    hkiG9w0BCQEWFHN1cHBvcnRAZm9ydGluZXQuY29tMIIBIjANBgkqhkiG9w0BAQEF
    AAOCAQ8AMIIBCgKCAQEArMATWZd2NRj3kRjHXRcNItmPPixI4whBNecCZe64RVcv
    QHukswRfOLm9Cup31EVzwOs5DlziWeEWdze+ZFmdhtGdTsV7RvNqBNha4I1etxTE
    MJQDoMf3sAVdzOkbF72+uWUhrC9HnqsYPbxstn2qDUVpAhsbQEMlVrxFJmmnFVrr
    kUTHugQvJ8sp9CkKOy5EYbmKDauZa+kIIrugyXXN0VOMkrbfRpBt1DSXsFJMwvc3
    9+bo5p5GZaG1s3YdN6y23eXFo5lRtaKAbqElzc5VlbXHFvMRu3GIDUmgHct/p7Jq
    kbzEv0ajLEVaj3sUBb98FdyFV+OiA7r0PDUm2fo2PwIDAQABoxAwDjAMBgNVHRME
    BTADAQH/MA0GCSqGSIb3DQEBCwUAA4IBAQCLznyaqdmBQ1HxEDFhcAzYCqjs8yJz
    uWOdPGu1i/7OI9mror4VGVRSgNCkQEqLbAsbMzV1ind0FAvW9BkoweMTbhL5vWe/
    QZEXQX6L6ovMr4+rnSMXX7Z6LJ8Lcs/4fPXUNrmmh7iHhR5l0JVCX2VztR4GUcvM
    U2imqJ2+yVD4ayqtXFx/6eKNwAyXaN1Z2sR5AwauAaUw5DNCPBfdw9mmOc9cQGLB
    YxN6XPi1eHAA0FRBHeHx7IXCMoDaqCE3mrqPbhc5u8fAx/97h4cr/uLOG652jkr6
    0/O4yRLD8dR5OxEQyKFK3qk3tKqqFiygRsun9knCpTY/qsYLSEewWc7Z
    -----END CERTIFICATE-----
````

In an **AWS Controllers for Kubernetes (ACK)`Bucket` custom resource (CR)** you populate `spec.acl` with any of the **S3 “canned” ACL strings** that Amazon S3 itself accepts.
That list is identical to the one in the S3 API and SDK docs:

| ACL value                   | Typical use‑case (very short)                                                                      |
| --------------------------- | -------------------------------------------------------------------------------------------------- |
| `private`                   | Bucket/object owner only (default)                                                                 |
| `public-read`               | Anyone can **GET** objects (website/static assets)                                                 |
| `public-read-write`         | Anyone can **GET/PUT/DELETE** objects (rarely recommended)                                         |
| `authenticated-read`        | Any AWS‑authenticated principal can read                                                           |
| `log-delivery-write`        | Lets the *LogDelivery* service principal write server‑access logs to the bucket                    |
| `bucket-owner-read`         | Gives the bucket owner **READ** access to an object that another account uploads                   |
| `bucket-owner-full-control` | Gives the bucket owner **FULL\_CONTROL** over an uploaded object (common for cross‑account uploads) |
| `aws-exec-read`             | Grants AmazonEC2 permission to read an AMI manifest/object                                         |

Sources
• Field description for `spec.acl` in the ACK S3 Bucket CRD([GitHub][1])
• Complete canned ACL value list in the S3 API (`CannedAccessControlList`)([AWS Documentation][2])
• `log‑delivery‑write` and other bucket‑level details in the ACL overview([AWS Documentation][3])
• Core four values shown in the `PutBucketAcl` API reference([AWS Documentation][4])

> **Heads‑up (2025‑07‑21):**
> If your bucket has **Object Ownership set to *Bucket owner enforced***, ACLs are disabled and S3 will reject any `acl` setting (except `bucket-owner-full-control` on object puts). In that case, manage access with bucket policies instead of ACK `spec.acl`.

[1]: https://raw.githubusercontent.com/aws-controllers-k8s/s3-controller/main/config/crd/bases/s3.services.k8s.aws_buckets.yaml "raw.githubusercontent.com"
[2]: https://docs.aws.amazon.com/AmazonS3/latest/API/API_control_S3AccessControlPolicy.html "S3AccessControlPolicy - Amazon Simple Storage Service"
[3]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/acl-overview.html?utm_source=chatgpt.com "Access control list (ACL) overview - Amazon Simple Storage Service"
[4]: https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutBucketAcl.html "PutBucketAcl - Amazon Simple Storage Service"
