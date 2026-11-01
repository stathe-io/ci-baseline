# ci-baseline

GitLab CI/CD components for secure and air-gapped environments.

Every pipeline is assembled from these components and none copies them. A
control improved here is improved in every pipeline on its next release.

| Component | |
|---|---|
| `scan` | Trivy against a database the runner mounts. Fails on a fixable vulnerability at the chosen severity and on a database past its maximum age |
| `sbom` | A CycloneDX SBOM as artifact |
| `sign` | cosign by digest with the key in the file variable `COSIGN_KEY`, no transparency log |

```yaml
include:
  - component: $CI_SERVER_FQDN/platform/ci-baseline/scan@1.0.0
    inputs:
      image: registry.enclave.example/app@sha256:...
```

[examples/promote](examples/promote) scans, describes and signs an image.

Run in GitLab CE 19.4.5 with a Docker runner. A clean image passes and is
signed. An image with fixable critical findings fails the scan and is not
signed. The Trivy database came from a bundle, mounted read-only.
