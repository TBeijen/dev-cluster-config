# Dev Cluster Config

A [Taskfile](https://taskfile.dev/) setup for:

* macOS

Using:

* brew
* k3d
* openssl
* dnsmasq
* cert-manager
* trust-manager

That provides:

* Setup TLS certificate for local development
* DNS configuration from macOS host to cluster workloads (north-south)
* Similar DNS configuration between services within clusters (east-west)

Read more in the article [East, west, north, south: How to fix your local cluster routes](https://www.tibobeijen.nl/2025/03/24/east-west-north-south-fix-local-cluster-routes/)

## Setup

### Prepare

Look at `.default.env`, override anything you want to adjust to `.env`. 

Example:

```ini
K3D_KUBECONFIG_TARGET_DIR=~/workspaces/local/k3d/.kube/
ADDITIONAL_CA_BUNDLE_FILE=/Users/<me>/corporate-bundle-combined.pem
```

### Podman machine

Only needed once, after creating a new podman machine.
Only needed if using a corporate MitM proxy.

```sh
task podman-machine-prepare [machine-name-if-not-default]
```

### TLS and DNS

Typically once. Requires root.

```sh
task cert
task dnsmasq-brew
```

### Setup cluster

```sh
task k3d-cluster-setup-1
# Add example apps + netshoot (recommended)
task k3d-cluster-examples-1
```

## Changing TLD

If you need to adjust the TLD, for example because hitting corporate search domain issues (see below):

Cleanup resolver and dnsmasq config for the old TLD.

```sh
# Old TLD in these examples: local
ls -al /etc/resolver/
sudo rm /etc/resolver/local

ls -al /opt/homebrew/etc/dnsmasq.d/
rm /opt/homebrew/etc/dnsmasq.d/local.conf
```

Then re-run the TLS and DNS steps as described above, with the updated `.env`.

## Choosing a TLD

The `TLD` variable in `.default.env` (overridable via `.env`) controls the top-level domain used for cluster services. The default is `.internal`. When choosing a TLD, keep the following in mind:

* **Avoid `.local`** -- it is reserved for mDNS (RFC 6762) and receives special treatment on macOS. More importantly, corporate networks often push DNS search domains (e.g. `corp.local`) to connected devices. When these search domains share the same TLD as the one configured in dnsmasq on the host, in-cluster DNS queries can resolve against the wrong DNS server, breaking east-west traffic.
* **Use an IANA-reserved TLD** that will never be delegated in the global DNS root:
  * `.internal` (RFC 8375) -- reserved for private-use applications. Good default.
  * `.test` (RFC 6761) -- reserved for testing. Also a safe choice.
* **Check your host's search domains** (`scutil --dns` on macOS) and ensure the chosen TLD does not appear in any search domain suffix. If it does, the Kubernetes default `ndots:5` setting will cause pod DNS queries to try appending those search domains before resolving the bare name, potentially matching an unintended DNS server.

## Corporate CA Bundle

If your network uses a TLS-intercepting proxy (e.g. Zscaler), the cluster will be affected at two levels:

1. **Node level** -- containerd cannot pull images from registries (e.g. Docker Hub) because the proxy's CA is not trusted by the k3d node OS.
2. **Pod level** -- application pods that make outbound HTTPS requests will fail certificate verification.

Set `ADDITIONAL_CA_BUNDLE_FILE` in your `.env` to the path of a PEM file containing the additional CA certificate(s):

```ini
ADDITIONAL_CA_BUNDLE_FILE=~/corporate-ca-bundle.pem
```

When set:

* `k3d-cluster-setup-*` mounts the CA bundle into the k3d node at `/etc/ssl/certs/additional-ca.crt`, so containerd trusts it when pulling images.
* `k3d-cluster-configure-*` creates a ConfigMap `additional-ca-bundle` in the `cert-manager` namespace and includes it as an additional source in the trust-manager `Bundle`. Pods that mount the `default-ca-bundle` ConfigMap (like the example `curl` pod) will then trust the corporate CA alongside the system CAs and the local dev root CA.

When left empty (the default), no volume mount or additional ConfigMap is created.

### Podman machine preparation

If Podman is used as the container runtime (the macOS default), the Podman VM itself also needs to trust the corporate CA. Without this, image pulls from within the VM may fail with TLS errors before k3d even starts.

This is a one-time setup step, repeated only when the CA bundle changes or the Podman machine is recreated:

```sh
task podman-machine-prepare
# Or, for a non-default machine name:
task podman-machine-prepare -- my-machine-name
```

This copies the `ADDITIONAL_CA_BUNDLE_FILE` into the VM at `/etc/pki/ca-trust/source/anchors/additional-ca.pem` and runs `update-ca-trust`. A bundle PEM file containing multiple certificates is fine -- `update-ca-trust` parses each PEM block independently.
