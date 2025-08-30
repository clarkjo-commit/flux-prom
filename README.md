# Monitoring & Logging GitOps Pattern

This repository provides a GitOps pattern to install **Prometheus** and **Fluent Bit** on a Tanzu Kubernetes Grid Service (TKGS) cluster using Flux and Kustomize. It leverages Tanzu PackageInstalls from a Carvel/Tanzu package repository and supports per-environment customization via overlays.

---

## Table of Contents

* [Overview](#overview)
* [Prerequisites](#prerequisites)
* [Repository Structure](#repository-structure)
* [Flux Bootstrap](#flux-bootstrap)
* [Configure Your Environment](#configure-your-environment)
* [Deploying Monitoring & Logging](#deploying-monitoring--logging)
* [Customizing Settings](#customizing-settings)

  * [Adjust LoadBalancer IP](#adjust-loadbalancer-ip)
  * [Templatize Fluent Bit Output Host](#templatize-fluent-bit-output-host)
* [Troubleshooting](#troubleshooting)
* [Cleanup](#cleanup)

---

## Overview

This pattern uses Flux CD and Kustomize to declaratively manage:

* **Prometheus**: Installed via a Tanzu `PackageInstall` with a `LoadBalancer` service.
* **Fluent Bit**: Installed via a Tanzu `PackageInstall` with a templated `output.host` value.

Environments (`dev`, `staging`, `prod`) are defined as Kustomize overlays, allowing IP addresses and log destinations to differ per environment. Flux will sync the correct overlay based on the `${ENV}` variable.

---

## Prerequisites

1. **TKGS** cluster running and accessible.
2. **kubectl** configured for your cluster.
3. **flux CLI** installed locally.
4. GitHub (or Git hosting) access for your repo.
5. A Tanzu Package Repository (Carvel `imgpkgBundle`) URL.

---

## Repository Structure

```text
.
├── flux-system
│   ├── bootstrap
│   │   ├── gitrepository.yaml    # Flux Source manifest for this repo
│   │   └── kustomization.yaml    # Flux Kustomization to install controllers
│   ├── package-repo.yaml        # Tanzu/Carvel PackageRepository definition
│   └── kustomization.yaml       # Flux Kustomization to apply overlays/${ENV}
└── apps
    ├── base
    │   ├── prometheus
    │   │   ├── kustomization.yaml
    │   │   └── prometheus-packageinstall.yaml
    │   └── fluentbit
    │       ├── kustomization.yaml
    │       └── fluentbit-packageinstall.yaml
    └── overlays
        ├── dev
        │   ├── kustomization.yaml
        │   ├── prometheus-patch.yaml
        │   └── fluentbit-patch.yaml
        ├── staging
        │   ├── kustomization.yaml
        │   ├── prometheus-patch.yaml
        │   └── fluentbit-patch.yaml
        └── prod
            ├── kustomization.yaml
            ├── prometheus-patch.yaml
            └── fluentbit-patch.yaml
```

* **flux-system/bootstrap**: Contains manifests to bootstrap Flux from the repo itself.
* **Base**: Core definitions of the two PackageInstalls with defaults.
* **Overlays**: Environment-specific patches for LoadBalancer IP and Fluent Bit output host.
* **flux-system**: Flux’s own Kustomizations and the PackageRepository to install Tanzu packages.

---

## Flux Bootstrap

We include Flux bootstrap manifests under `flux-system/bootstrap` so you can install Flux directly from Git:

1. **Apply the bootstrap manifests**:

   ```bash
   kubectl apply -f flux-system/bootstrap/gitrepository.yaml
   kubectl apply -f flux-system/bootstrap/kustomization.yaml
   ```

2. **Or bootstrap via CLI** (GitHub example):

   ```bash
   flux bootstrap github \
     --owner=<owner> --repository=<repo> --branch=<branch> \
     --path=flux-system/bootstrap \
     --personal
   ```

This will install Flux controllers into `flux-system` and point them at this repository’s `flux-system/bootstrap` directory.

---

## Configure Your Environment

Export the environment variable corresponding to your target overlay. For example, to deploy to **dev**:

```bash
export ENV=dev
```

Ensure your GitRepository resource (in `bootstrap/gitrepository.yaml`) uses the correct URL, branch, and path.

---

## Deploying Monitoring & Logging

Once Flux is bootstrapped and watching the repo:

```bash
flux reconcile source git flux-system          # Refresh Git source
flux reconcile kustomization monitoring-logging # Apply the overlay
```

This will install or update:

* **Prometheus** in the `monitoring` namespace (LoadBalancer service).
* **Fluent Bit** in the `logging` namespace (sending logs to your environment-specific host).

---

## Customizing Settings

### Adjust LoadBalancer IP

Each overlay’s `prometheus-patch.yaml` sets the `loadBalancerIP`:

```yaml
spec:
  values:
    - inline:
        server:
          service:
            loadBalancerIP: 10.0.0.100
```

Edit this IP in `apps/overlays/<env>/prometheus-patch.yaml` to match your environment’s networking.

### Templatize Fluent Bit Output Host

In your overlay’s `fluentbit-patch.yaml`, set the log destination host:

```yaml
spec:
  values:
    - inline:
        output:
          host: logs-dev.example.local
```

Update the `host` per environment in:

```
apps/overlays/dev/fluentbit-patch.yaml
apps/overlays/staging/fluentbit-patch.yaml
apps/overlays/prod/fluentbit-patch.yaml
```

---

## Troubleshooting

* **Flux not syncing?** Check `flux logs source git flux-system` and `flux get kustomizations -A`.
* **Prometheus LoadBalancer pending?** Verify your TKGS network supports LoadBalancer services and that the IP is not in use.
* **Fluent Bit errors?** Inspect the `fluentbit` Pod logs and confirm the `output.host` DNS resolves and accepts TCP on port 24224.

---

## Cleanup

To remove all resources deployed by this pattern:

```bash
flux delete kustomization monitoring-logging --namespace flux-system
kubectl delete -f flux-system/package-repo.yaml
kubectl delete -f flux-system/bootstrap
```

Then clean up namespaces and PackageInstalls:

```bash
kubectl delete namespace monitoring logging
kubectl delete packageinstall prometheus -n monitoring
kubectl delete packageinstall fluentbit -n logging
```

---

For further customization (scraping additional targets, adjusting retention, TLS, etc.), modify the base `values.inline` sections in `apps/base/*` or add additional patches in your overlays.

Happy GitOps! 🚀
