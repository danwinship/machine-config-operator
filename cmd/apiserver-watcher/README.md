# apiserver-watcher / openshift-${platform}-routes

## Intro

On cloud platforms, the installer creates internal and external cloud
load balancers for the apiserver. On some platforms, the internal load
balancer needs some special handling in order to work correctly.

## Background

### DNAT vs Direct Server Return

On most platforms, load balancers DNAT incoming traffic to the
destination node. However, on some platforms, they instead just
forward it to the destination with the packets still having the IP of
the load balancer as their destination IP. This allows "Direct Server
Return", where the reply packet goes directly back to the client
rather than needing to pass through the load balancer again, but it
requires that the node take some action to know that it should be
accepting packets addressed to the load balancer IP.

Currently GCP is the only OpenShift platform whose internal apiserver
load balancer is configured this way. The [`openshift-gcp-routes`]
service handles this by creating iptables rules to redirect inbound
load-balancer IP packets to the node IP (the same way that kube-proxy
does for load balancer IPs used by Kubernetes Services).

[`openshift-gcp-routes`]: ../../templates/master/00-master/gcp/files/opt-libexec-openshift-gcp-routes-sh.yaml

### Hairpin

Because default OpenShift installations are "self-driving", i.e. the
control plane is hosted as part of the cluster, we rely extensively on
"hairpin" apiserver connections (where a connection is addressed to
the apiserver load balancer, but ends up redirected back to the node
that it came from):

```
 +---------------+
 |               |          +-----------------+
 |  +---------+  |          |                 |
 |  | kubelet +------------->  layer-3        |
 |  +---------+  |          |  load balancer  |
 |               |     +----+                 |
 |  +---------+  |     |    +-----------------+
 |  |apiserver+<-------+
 |  +---------+  |
 |               |
 +---------------+
```

On Azure and Alibaba Cloud, a hairpin connection like this will not
work, because the load balancer DNATs the packet but does not SNAT it.
As a result, the kubelet sends out a packet with source `${node_ip}`
and destination `${loadbalancer_ip}` but the apiserver receives a
packet with source `${node_ip}` and destination `${node_ip}`. When it
tries to send the reply back, the kernel believes it should be able to
immediately deliver the packet locally, but this fails because there
is no client with a matching 5-tuple. (This does not happen for
non-hairpin connections because in that case the reply will pass
through the cloud network again, and so the cloud can un-DNAT it.)

The [`openshift-azure-routes`] service (and
[`openshift-alibaba-routes`] which is basically identical) handles
this by intercepting outbound connections to the apiserver load
balancer IP, and always redirecting them to the local apiserver,
completely ignoring the cloud load balancer.

On GCP, hairpin connections _do_ work, but only because the load
balancer will SNAT them. Since we don't want that,
`openshift-gcp-routes` also intercepts and redirects apiserver
connections in the same way we do on Azure and Alibaba.

[`openshift-azure-routes`]: ../../templates/master/00-master/azure/files/opt-libexec-openshift-azure-routes-sh.yaml
[`openshift-alibaba-routes`]: ../../templates/master/00-master/alibaba/files/opt-libexec-openshift-alibaba-routes-sh.yaml

## apiserver-watch Functionality

In all of the cases above, we need to redirect local clients to the
local apiserver when it is running, but allow them to reach a remote
apiserver when there is no local one.

The apiserver-watcher runs as a static pod on all the masters, and
monitors the apiserver's `/readyz` endpoint. (See the [kube-apiserver
healthcheck] documentation for more information about that.)

When `/readyz` reports that the apiserver is ready, apiserver-watcher
will create a file `/run/cloud-routes/$VIP.up` for each load balancer
IP used by the internal apiserver load balancer. When the apiserver is
not ready, it will remove `$VIP.up` and create
`/run/cloud-routes/$VIP.down`.

The `openshift-gcp-routes`, `openshift-azure-routes`, and
`openshift-alibaba-routes` services use systemd path activation to run
any time a file in `/run/cloud-routes` changes, to update their
iptables rules as needed.

[kube-apiserver healthcheck]: https://github.com/openshift/installer/docs/dev/kube-apiserver-health-check.md

## Notes/History

RHCOS also contains a service called `gcp-routes`, which is an older
version of the `openshift-gcp-routes` code. This is needed on the
bootstrap node (to allow it to act as an endpoint of the apiserver
loadbalancer), but it is disabled on all "real" OpenShift nodes.

(Originally the version of the script in RHCOS was used in OpenShift
as well, before it was forked into MCO for maintainability reasons.)

The word "routes" in the name is an artifact of the original
implementation of the service, which used routes to handle inbound
loadbalancer connections rather than iptables rules. However, the
routing-based implementation could not properly handle graceful
termination, so it was rewritten.
