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

## Choosing a TLD

The `TLD` variable in `.default.env` (overridable via `.env`) controls the top-level domain used for cluster services. The default is `.internal`. When choosing a TLD, keep the following in mind:

* **Avoid `.local`** -- it is reserved for mDNS (RFC 6762) and receives special treatment on macOS. More importantly, corporate networks often push DNS search domains (e.g. `corp.local`) to connected devices. When these search domains share the same TLD as the one configured in dnsmasq on the host, in-cluster DNS queries can resolve against the wrong DNS server, breaking east-west traffic.
* **Use an IANA-reserved TLD** that will never be delegated in the global DNS root:
  * `.internal` (RFC 8375) -- reserved for private-use applications. Good default.
  * `.test` (RFC 6761) -- reserved for testing. Also a safe choice.
* **Check your host's search domains** (`scutil --dns` on macOS) and ensure the chosen TLD does not appear in any search domain suffix. If it does, the Kubernetes default `ndots:5` setting will cause pod DNS queries to try appending those search domains before resolving the bare name, potentially matching an unintended DNS server.
