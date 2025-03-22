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

Read more in the article [East, west, north, south: How to fix your local cluster routes]()
