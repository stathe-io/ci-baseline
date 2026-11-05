# ci-baseline

GitLab CI/CD components for secure and air-gapped environments.

Every pipeline is assembled from these components and none copies them. A
control improved here is improved in every pipeline on its next release.

| Component | What it does |
|---|---|
| [scan](templates/scan.yml) | Checks an image for vulnerabilities. Fails on a fixable one at the chosen severity |
| [sbom](templates/sbom.yml) | Lists everything inside an image as an SBOM |
| [sign](templates/sign.yml) | Signs an image so the cluster lets it run |
| [verify](templates/verify.yml) | Refuses to go on unless an image carries your signature |
| [secrets](templates/secrets.yml) | Stops a pipeline when a secret is in the Git history |

Each file explains its settings at the top.

## The shortest start

`secrets` needs no setup beyond step 1 below. In any project:

```yaml
stages: [test]

include:
  - component: $CI_SERVER_FQDN/platform/ci-baseline/secrets@1.0.0
```

For `verify`, add `cosign.pub` as the file variable `COSIGN_PUB` and put it in
front of your deploy:

```yaml
include:
  - component: $CI_SERVER_FQDN/platform/ci-baseline/verify@1.0.0
    inputs:
      image: $IMAGE

deploy:
  needs: [verify]
```

## Setting it up

Do these steps once per GitLab instance. Each step ends with a check.

**1. Put this repository in your GitLab.** Create a group `platform` and a
project `ci-baseline` in it, then push this repository there.

> Check: the project shows a `templates/` folder with five files.

**2. Release a version.** Create a tag, for example `1.0.0`. Pipelines refer
to that version, so a change here reaches a pipeline only when it moves to the
next one.

> Check: under Code > Tags you see `1.0.0`.

**3. Give the runner the vulnerability database.** The scan and sbom
components read it from `/trivy-cache` in the job container. On a runner with
the Docker executor, add this to the runner's `config.toml` under
`[runners.docker]`:

```toml
volumes = ["/srv/trivy-cache:/trivy-cache:ro"]
```

Put the Trivy database on the runner host in `/srv/trivy-cache/db/`, so that
`trivy.db` and `metadata.json` are there. Replace it with every transfer.

> Check: on the runner host, `cat /srv/trivy-cache/db/metadata.json` shows an
> `UpdatedAt` less than a week old.

**4. Make the tool images available.** The components use the Trivy, cosign and
gitleaks images named in their `trivy_image`, `cosign_image` and
`gitleaks_image` inputs. In an
air-gapped network they must be in your enclave registry, under the same
digest.

> Check: `docker pull` of each image works from the runner host.

**5. Create the signing key.** Run `cosign generate-key-pair` once, on a
machine you trust. In the project that will sign images, add two variables
under Settings > CI/CD > Variables:

| Key | Type | Value |
|---|---|---|
| `COSIGN_KEY` | File | The contents of `cosign.key` |
| `COSIGN_PASSWORD` | Variable, masked | The password you chose |

Give `cosign.pub` to whoever sets up admission in the cluster. Do not commit
`cosign.key` anywhere.

> Check: both variables are listed in the project settings.

## Your first pipeline

Copy [examples/promote/.gitlab-ci.yml](examples/promote/.gitlab-ci.yml) into
an application project. Replace `@main` with your tag, for example `@1.0.0`.
Run it from Build > Pipelines > Run pipeline, with the variable `IMAGE` set to
an image pinned by digest.

> Check: scan, sbom and sign pass. Then `cosign verify --key cosign.pub
> --insecure-ignore-tlog <image>` confirms the signature.

## When something fails

| What you see | What it means | What to do |
|---|---|---|
| `sed: /trivy-cache/db/metadata.json: No such file or directory` | The runner has no database mounted | Step 3 |
| `vulnerability database updated ..., maximum 168h` and the job fails | The database is too old | Bring in a new one, step 3 |
| CVE IDs and `N fixable CRITICAL findings` | The image has known problems with a fix | Use a newer image. This is the component doing its job |
| `http: server gave HTTP response to HTTPS client` | The registry has no TLS | Give it a certificate. For a test setup only, set `insecure_registry: true` on scan and sbom and `http_registry: true` on sign |
| `sign` shows as skipped | scan or sbom failed first | Fix that job. Signing only happens after both pass |
| `no signatures found` | Nobody signed this image | Send it through the pipeline that signs. Do not deploy |
| `no matching signatures` | It was signed with another key | Check `COSIGN_PUB`. If the key is right, do not deploy |
| `leaks found` in `secrets` | A secret is in the history | Replace the secret. Deleting the line leaves it in Git |

## Tested

In GitLab CE 19.4.5 with a Docker runner. A clean image passed and was signed.
An image with fixable critical findings failed the scan and was not signed.
The database came from a bundle, mounted read-only.

`verify` and `secrets`: not run in GitLab yet. Their scripts ran as written in
their pinned images. `verify` passed a signed image, and failed on the wrong key
and on an unsigned image, against a registry without TLS. `secrets` passed a
clean repository and failed on a token deleted in a later commit, with the value
redacted in the report.

## Licence

Apache 2.0. See [LICENSE](LICENSE).
