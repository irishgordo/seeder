# Tinkerbell tinkerbell-stack

This chart installs the full Tinkerbell tinkerbell-stack.

## TL;DR

```bash
helm dependency build tinkerbell-stack/
trusted_proxies=$(kubectl get nodes -o go-template-file=tinkerbell-stack/kubectl.go-template)
helm install tinkerbell-stack-release tinkerbell-stack/ --create-namespace --namespace tink-system --wait --set "smee.trustedProxies={${trusted_proxies}}" --set "hegel.trustedProxies={${trusted_proxies}}"
```

## Introduction

This chart smeetraps a full Tinkerbell tinkerbell-stack on a Kubernetes cluster using the Helm package manager. The Tinkerbell tinkerbell-stack consists of the following components:

- [Boots](https://github.com/tinkerbell/smee)
- [Hegel](https://github.com/tinkerbell/hegel)
- [Tink](https://github.com/tinkerbell/tink)
- [Rufio](https://github.com/tinkerbell/rufio)

This chart also installs a load balancer ([kube-vip](https://kube-vip.io/)) in order to be able to provide a service type loadBalancer IP for the Tinkerbell tinkerbell-stack services and an Nginx server for handling proxying to the Tinkerbell service and for serving the Hook artifacts.

## Design details

The tinkerbell-stack chart does not use an ingress object and controller. This is because most ingress controllers do not support UDP. Boots uses UDP for DHCP, TFTP, and Syslog services. The ingress controllers that do support UDP require a lot of extra configuration, custom resources, etc. The tinkerbell-stack chart deploys a very light weight Nginx deployment with a straightforward configuration that accommodates all the Tinkerbell tinkerbell-stack services and serving Hook artifacts.

## Prerequisites

- Kubernetes 1.23+
- Kubectl 1.23+
- Helm 3.9.4+

## Installing the Chart

Before installing the chart you'll want to customize the IP used for the load balancer (`tinkerbell-stack.loadBalancerIP`). This IP provides ingress for Hegel, Tink, and Boots (TFTP, HTTP, and SYSLOG endpoints as well as unicast DHCP requests).

Now, deploy the chart.

```bash
helm dependency build tinkerbell-stack/
trusted_proxies=$(kubectl get nodes -o go-template-file=tinkerbell-stack/kubectl.go-template)
helm install tinkerbell-stack-release tinkerbell-stack/ --create-namespace --namespace tink-system --wait --set "smee.trustedProxies={${trusted_proxies}}" --set "hegel.trustedProxies={${trusted_proxies}}"
```

These commands install the Tinkerbell tinkerbell-stack chart in the `tink-system` namespace with the release name of `tinkerbell-stack-release`.

## Uninstalling the Chart

To uninstall/delete the `tinkerbell-stack-release` deployment:

```bash
helm uninstall tinkerbell-stack-release --namespace tink-system
```

## Upgrading the Chart

To upgrade the `tinkerbell-stack-release` deployment:

```bash
helm upgrade tinkerbell-stack-release tinkerbell-stack/ --namespace tink-system --wait
```

## Parameters

### tinkerbell-stack Service Parameters

| Name | Description | Value |
| ---- | ----------- | ----- |
| `tinkerbell-stack.enabled` | Enable the deployment of the Tinkerbell tinkerbell-stack chart | `true` |
| `tinkerbell-stack.name` | Name for the tinkerbell-stack chart | `tink-tinkerbell-stack` |
| `tinkerbell-stack.service.type` | Type of service to use for the Tinkerbell tinkerbell-stack services. One of the [standard](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types) Kubernetes service types. | `LoadBalancer` |
| `tinkerbell-stack.selector` | Selector(s) to use for the mapping tinkerbell-stack deployment with the service | `app: tink-tinkerbell-stack` |
| `tinkerbell-stack.loadBalancerIP` | Load balancer IP address to use for the Tinkerbell tinkerbell-stack services | `192.168.2.111` |
| `tinkerbell-stack.lbClass` | loadBalancerClass to use for in tinkerbell-stack service | `kube-vip.io/kube-vip-class` |
| `tinkerbell-stack.image` | Image to use for the proxying to Tinkerbell services and serving artifacts | `nginx:1.23.1` |
| `tinkerbell-stack.hook.enabled` | Enable the deployment of the Hook artifacts | `true` |
| `tinkerbell-stack.hook.name` | Name for the Hook artifacts server | `hook-files` |
| `tinkerbell-stack.hook.port` | Port to use for the Hook artifacts server | `8080` |
| `tinkerbell-stack.hook.image` | Image to use for downloading the Hook artifacts | `alpine` |
| `tinkerbell-stack.hook.downloadsDest` | The directory on disk to where Hook artifacts will downloaded  | `/opt/hook` |
| `tinkerbell-stack.hook.downloadURL` | The base URL where all Hook tarballs and checksum.txt file exist for downloading | `https://github.com/tinkerbell/hook/releases/download/latest` |

### Load Balancer Parameters (kube-vip)

| Name | Description | Value |
| ---- | ----------- | ----- |
| `kubevip.enabled` | Enable the deployment of the kube-vip load balancer | `true` |
| `kubevip.name` | Name for the kube-vip load balancer service | `kube-vip` |
| `kubevip.image` | Image to use for the kube-vip load balancer | `ghcr.io/kube-vip/kube-vip:v0.5.0` |
| `kubevip.imagePullPolicy` | Image pull policy to use for kube-vip | `IfNotPresent` |
| `kubevip.roleName` | Role name to use for the kube-vip load service | `kube-vip-role` |
| `kubevip.roleBindingName` | Role binding name to use for the kube-vip load service | `kube-vip-rolebinding` |
| `kubevip.interface` | Interface to use for advertizing the load balancer IP. Leaving it unset to allow Kubevip to auto discover the interface to use. | `""` |

### DHCP Relay Parameters

| Name | Description | Value |
| ---- | ----------- | ----- |
| `tinkerbell-stack.relay.name` | Name for the relay service | `dhcp-relay` |
| `tinkerbell-stack.relay.enabled` | Enable the deployment of the DHCP relay service | `true` |
| `tinkerbell-stack.relay.image` | Image to use for the DHCP relay service | `ghcr.io/jacobweinstock/dhcrelay` |
| `tinkerbell-stack.relay.maxHopCount` | Maximum number of hops to allow for DHCP relay | `10` |
| `tinkerbell-stack.relay.sourceInterface` | Host/Node interface to use for listening for DHCP broadcast packets | `eno1` |

### Tinkerbell Services Parameters

All dependent services(Boots, Hegel, Rufio, Tink) can have their values overridden here. The following format is used to accomplish this.

```yaml
<service name>:
  <key to override>: <value>
  <array key to override>:
    - <key>: <value>
```

Example:

```yaml
hegel:
  image: quay.io/tinkerbell/hegel:latest
```

### Boots Parameters

| Name | Description | Value |
| ---- | ----------- | ----- |
| `smee.hostNetwork` | Whether to deploy Boots using `hostNetwork` on the pod spec. When `true` Boots will be able to receive DHCP broadcast messages. If `false`, Boots will be behind the load balancer VIP and will need to receive DHCP requests via unicast. | `true` |
