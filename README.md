# ansible-role-k8s-master
Ansible role for Kubernetes master nodes

- [ansible-role-k8s-master](#ansible-role-k8s-master)
- [Overview](#overview)
- [Configuration](#configuration)
  * [Versions](#versions)
  * [PKI](#pki)
  * [Kubernetes](#kubernetes)
- [Installation](#installation)

# Overview

This is an ansible role for configuring a master node within the Kubernetes container orchestration platform. It can be used on it's own, but it is recommended that this role is used in conjunction with the associated [worker node role](https://github.com/adammillerio/ansible-role-k8s-worker). This role supports being ran against multiple master nodes in a "HA" configuration, however, there must be an odd number of master nodes in order for the `etcd` state store to retain quorum.

This playbook performs the following steps:

* Creates the kubernetes configuration directories (dirs.yml)
* Installs the CloudFlare PKI and TLS toolkit (cfssl.yml)
* Generates the required Public Key Infrastructure (pki.yml)
    * Certificate Authority (ca.pem, ca-key.pem)
    * Administrator user certificate (admin.pem, admin-key.pem)
    * API server certificate (kubernetes.pem, kubernetes-key.pem)
    * Client certificate for each master node ($HOSTNAME.pem $HOSTNAME-key.pem)
    * Kubernetes proxy certificate (kube-proxy.pem, kube-proxy-key.pem)
* Generates the encryption key to be used for encrypting data at rest (encryption.yml)
* Installs and configures the Docker Container Runtime Interface (docker.yml)
* Installs and configures the Kubernetes base daemons (k8s-base.yml)
    * Container Networking Interface (CNI) plugins
    * Kubernetes client (kubectl)
    * Kubernetes agent (kubelet)
    * Kubernetes proxy (kube-proxy)
* (Optionally) Installs and configures the [consul](https://www.consul.io/) distributed service discovery daemon (consul.yml)
* Installs and configures the [etcd](https://github.com/coreos/etcd) distributed key-value store (etcd.yml)
* Installs and configures the Kubernetes control plane daemons (k8s-control-plane.yml)
    * Kubernetes API Server (kube-apiserver)
    * Kubernetes Controller Manager (kube-controller-manager)
    * Kubernetes Scheduler (kube-scheduler)
* Deploys the [flannel](https://github.com/coreos/flannel) Container Networking Interface as a Kubernetes DaemonSet (flannel.yml)
* Deploys the Kubernetes DNS cluster addon (kube-dns.yml)
* (Optionally) Installs and configures the [Helm](https://www.helm.sh/) Tiller packaging tool

There are a few major configuration choices of note in this cluster configuration:
* Targets Debian 9.0 "stretch"
* Uses systemd units for the `kubelet` and `docker` runtimes
* Uses Kubernetes [Static Pods](https://kubernetes.io/docs/tasks/administer-cluster/static-pod/) for all other configured daemons that are managed via the `kubelet`
* Uses the [flannel](https://github.com/coreos/flannel) Container Networking Interface to implement the [Kubernetes Networking Model](https://kubernetes.io/docs/concepts/cluster-administration/networking/) which allows for bare-metal deployments

# Configuration
The following variables are utilized within the role. Sensible defaults have been provided for each:

## Versions

| Name  | Default  | Description  |
|---|---|---|
| cfssl_version | 1.2 | Version of the CloudFlare PKI and TLS toolkit to install |
| consul_version | 1.2.3 | Version of the Consul service discovery daemon to install |
| etcd_version | 3.3.9 | Version of the etcd distributed KV store to install |
| k8s_version | 1.11.3 | Version of the Kubernetes components to install |
| flannel_version | 0.10.0 | Version of the flannel CNI to install |
| kubedns_version | 1.14.13 | Version of the Kubernetes DNS addon to install |
| docker_version | 18.06.1 | Version of the Docker CRI to install |
| docker_edition | ce | Edition of the Docker CRI to install |
| cni_plugins_version | 0.7.2 | Version of the CNI plugins to install |
| helm_tiller_version | 2.11.0 | Version of the Helm Tiller to install |

## System
| Name  | Default  | Description  |
|---|---|---|
| system_interface | default | Interface for services to bind to, "default" will use the first interface Ansible finds |

## PKI
| Name  | Default  | Description  |
|---|---|---|
| pki_expiry | 8760h | Time period before the generated Certificate Authority certificate expires |
| pki_algorithm | rsa | Algorithm used to generate the certificates |
| pki_key_size | 2048 | Size of the generated certificates in bits |
| pki_country | US | Country of the generated certificates |
| pki_locality | Annandale | Locality of the generated certificates |
| pki_state | VA | State of the generated certificates |

# Consul
| Name  | Default  | Description  |
|---|---|---|
| consul_enable | true | Whether or not to install Consul for service discovery |

# Helm
| Name  | Default  | Description  |
|---|---|---|
| helm_enable | true | Whether or not to install Helm Tiller for packaging |
| helm_tiller_replicas | 1 | How many replicas of the Helm Tiller to deploy |

## Kubernetes
| Name  | Default  | Description  |
|---|---|---|
| k8s_apiserver_public_host | api.k8s.local | Public hostname of the K8S API Server, used in certificate generation |
| k8s_apiserver_internal_host | kubernetes.service.consul | Internal hostname of the K8S API Server, used in certificate generation, this should always use consul DNS if consul is enabled |
| k8s_apiserver_cni_host | 100.64.0.1 | IP address of the in-cluster Service IP of the K8S API Server, used in certificate generation |
| k8s_apiserver_count | # of hosts | Number of master nodes, be sure to override this if not targetting all masters in a play |
| k8s_cni_cluster_cidr | 100.64.0.0/10 | CIDR range to be used within the cluster |
| k8s_cni_node_cidr | 100.96.0.0/11 | CIDR range to be assigned to nodes within the cluster |
| k8s_cni_service_cidr | 100.64.0.0/13 | CIDR range to use for services within the cluster |
| k8s_cni_service_ports | 30000-32767 | Port range to use for NodePort services within the cluster |
| k8s_cni_cluster_dns_host | 100.64.0.10 | In-cluster IP address to be used by the Kubernetes DNS addon |
| k8s_cni_cluster_dns_domain | cluster.local | Domain to be used by the Kubernetes DNS addon |

Other variables are defined in `defaults/main.yml` but should only be overriden in special circumstances.

# Installation
Before using this role, ensure the following are available/configured on the host:
* ansible (Tested on 2.6.4)
* CloudFlare SSL toolkit (Tested on 1.2)
* User running the playbook has access to the `files` directory, as keys will be generated on the host and placed there

Then, simply add the `k8s-master` role to a playbook