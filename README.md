kube-router
==========
Kube-router is a distributed load balancer, firewall and router for Kubernetes. Kube-router can be configured to provide on each cluster node:

- an ingress firewall for the pods running on the node as per the defined Kubernetes network policies
- a service proxy on each node for *ClusterIP* and *NodePort* service types, providing service discovery and load balancing
- a router to advertise the routes to the pod IP's on the node to the peer nodes in the cluster and automatically install learned routes

## Why Kube-router
We have Kube-proxy which provides service proxy and load balancer. We have several addons or solutions like Flannel, Calico, Weave etc to provide cross node pod-to-pod networking. Simillarly there are solutions like Calico that enforce network policies. Then why do we need Kube-router for a similar job? Here is the motivation:

- It is challenging to deploy, monitor and troubleshoot multiple solutions at runtime. These independent solution need to work well together. Kube-router aims to provide operational simplicity by combining all the networking functionality that can be provided at the node in to one cohesive solution. Run Kube-router as daemonset, by just running one command ``kubectl --namespace=kube-system create -f  kube-router-daemonset.yaml`` you have solution for pod-to-pod networking, service proxy and firewall on each node.

- Kube-router is motivated to provide optimized solution for performance. Kube-router uses IPVS for service proxy as compared to iptables by Kube-proxy. Kube-router uses solutions like ipsets to optimize iptables rules matching while providing ingress firewall for the pods. For inter-node pod-to-pod communication, routing rules added by kube-router ensures data path is efficient (one hop for pod-to-pod connectivity) with out overhead of overlays.

- Kube-router builds on standard Linux technologies, so you can verify the configuration and troubleshoot with Linux networking tools (ipvsadm, ip route, iptables, ipset etc).

## Theory of Operation

Kube-router can be run as a agent or a pod (through daemonset) on each node and leverages standard Linux technologies **iptables, ipvs/lvs, ipset, iproute2** 

### service proxy and load balancing 

Kube-router uses IPVS/LVS technology built in Linux to provide L4 load balancing. Each of the kubernetes service of **ClusterIP** and **NodePort** type is configured as IPVS virtual service. Each service endpoint is configured as real server to the virtual service.
Standard **ipvsadm** tool can be used to verify the configuration and monitor the active connections. 

Below is example set of services on kubernetes

![Kube services](./Documentation/img/svc.jpg)

and the endpoints for the services

![Kube services](./Documentation/img/ep.jpg)

and how they got mapped to the ipvs by kube-router

![IPVS configuration](./Documentation/img/ipvs1.jpg)

Kube-router watches kubernetes API server to get updates on the services, endpoints and automatically syncs the ipvs
configuration to reflect desired state of services. Kube-router uses IPVS masquerading mode and uses round robin scheduling
currently. Source pod IP is preserved so that appropriate network policies can be applied.

### pod ingress firewall 

Kube-router provides implementation of network policies semantics through the use of iptables, ipset and conntrack.
All the pods in a namespace with 'DefaultDeny' ingress isolation policy has ingress blocked. Only traffic that matches
whitelist rules specified in the network policies are permitted to reach pod. Following set of iptables rules and 
chains in the 'filter' table are used to achieve the network policies semantics.

Each pod running on the node, which needs ingress blocked by default is matched in FORWARD and OUTPUT chains of fliter table 
and send to pod specific firewall chain. Below rules are added to match various cases

- traffic getting switched between the pods on the same node through bridge
- traffic getting routed between the pods on different nodes
- traffic originating from a pod and going through the service proxy and getting routed to pod on same node

![FORWARD/OUTPUT chain](./Documentation/img/forward.png)

Each pod specific firewall chain has default rule to block the traffic. Rules are added to jump traffic to the network policy 
specific policy chains. Rules cover only policies that apply to the destination pod ip. A rule is added to accept the
the established traffic to permit the return traffic.

![Pod firewall chain](./Documentation/img/podfw.png)

Each policy chain has rules expressed through source and destination ipsets. Set of pods matching ingress rule in network policy spec
forms a source pod ip ipset. set of pods matching pod selector (for destination pods) in the network policy forms
destination pod ip ipset.

![Policy chain](./Documentation/img/policyfw.png)

Finally ipsets are created that are used in forming the rules in the network policy specific chain

![ipset](./Documentation/img/ipset.jpg)

Kube-router at runtime watches Kubernetes API server for changes in the namespace, network policy and pods and
dynamically updates iptables and ipset configuration to reflect desired state of ingress firewall for the the pods.

### Pod networking

Kube-router is expected to run on each node. Subnet of the node is learnt by kube-router from the CNI configuration file on the node or through the node.PodCidr. Each kube-router
instance on the node acts a BGP router and advertise the pod CIDR assigned to the node. Each node peers with rest of the 
nodes in the cluster forming full mesh. Learned routes about the pod CIDR from the other nodes (BGP peers) are injected into
local node routing table. On the data path, inter node pod-to-pod communication is done by routing stack on the node.

## Getting Started

### building

**Go version 1.7 or above is required to build kube-router**

All the dependencies are vendored already, so just run  *make* or *go build -o kube-router kube-router.go* to build
   
Alternatively you can download the prebuilt binary from https://github.com/cloudnativelabs/kube-router/releases

### command line options

```
  --run-firewall                    If false, kube-router won't setup iptables to provide ingress firewall for pods. true by default. 
  --run-router                      If true each node advertise routes the rest of the nodes and learn the routes for the pods. false by default
  --run-service-proxy               If false, kube-router won't setup IPVS for services proxy. true by default.
  --cleanup-config                  If true cleanup iptables rules, ipvs, ipset configuration and exit.
  --masquerade-all                  SNAT all traffic to cluster IP/node port. False by default
  --config-sync-period duration     How often configuration from the apiserver is refreshed. Must be greater than 0. (default 1m0s)
  --iptables-sync-period duration   The maximum interval of how often iptables rules are refreshed (e.g. '5s', '1m'). Must be greater than 0. (default 1m0s)
  --ipvs-sync-period duration       The maximum interval of how often ipvs config is refreshed (e.g. '5s', '1m', '2h22m'). Must be greater than 0. (default 1m0s)
  --kubeconfig string               Path to kubeconfig file with authorization information (the master location is set by the master flag).
  --master string                   The address of the Kubernetes API server (overrides any value in kubeconfig)
  --routes-sync-period duration     The maximum interval of how often routes are advertised and learned (e.g. '5s', '1m', '2h22m'). Must be greater than 0. (default 1m0s)
```
### deployment

Depending on what functionality of kube-router you want to use multiple deployment options are possible.

- you can use kube-router as replacement of kube-proxy and run on all nodes in the cluster, providing service proxy, load balancing and ingress firewall for the pods
- you can run kube-router along with kube-proxy on all nodes in the cluster, to provide ingress firewall and pod networking. kube-proxy can continue to provide service proxy and load balancing

### running

Kube-router need to access kubernetes API server to get information on pods, services, endpoints, network policies etc.
The very minimum information it requires is the details on where to access the kubernetes API server. This information can
be passed as

```
kube-router --master=http://192.168.1.99:8080/
```
or 
``` 
kube-router --kubeconfig=<path to kubeconfig file>
```

In this minimal configuration mode, kube-router provides service proxy and ingress firewall on the node on which it is running. You
can use the flags *--run-firewall*, *--run-router*, *--run-service-proxy* to selectively run the required services.

For e.g if you just want kube-router to provide ingress firewall for the pods then you can start kube-router as 
```
kube-router --master=http://192.168.1.99:8080/ --run-service-proxy=false --run-router=false
```

You can clean up all the configurations done (to ipvs, iptables, ip routes) by kube-router on the node by running
```
 kube-router --cleanup-config
```

#### trying kube-router as alternative to kube-proxy

If you have a kube-proxy in use, and want to try kube-router just for service proxy you can do
```
kube-proxy --cleanup-iptables
```
followed by
```
kube-router --master=http://192.168.1.99:8080/ --run-firewall=false --run-router=false
```
and if you want to move back to kube-proxy then 
```
 kube-router --cleanup-config
```
and run kube-proxy with the configuration you have

##### pod networking
This section details how you can use kube-router to provide a soltion for cross-node pod networking. kube-router does
not (at this point) do subnet management for the node. It assumes the information is provided in the
CNI conf file. Kube-router does not implement a CNI plugin, rather it uses standard CNI *bridge* and *host-local* plugins.
CNI conf file must have CNI plugin type set to *bridge* and IPAM plugin type set to *host-local*. Subnet that will be
used as pod CIDR must be specified in the conf file like e.g. conf file shown below.

```
{
  "name": "mynet",
  "type": "bridge",
  "bridge": "kube-bridge",
  "isDefaultGateway": true,
  "ipam": {
    "type": "host-local",
    "subnet": "10.1.3.0/24"
  }
}
```
In this case kube-router will advertise the availability of subnet "10.1.3.0/24" through the node ip to the peers.

Assuming CNI conf file is located at */etc/cni/net.d/mynet.conf* kube-router can be started as

```
kube-router --master=http://192.168.1.99:8080/ --run-router=true --cni-conf-file=/etc/cni/net.d/mynet.conf
```

## See it in action

For a quick walkthrough of kube-router in action take a look at https://cloudnativelabs.github.io/blog/post/kube-router/

## TODO
- convert Kube-router to docker image and run it as daemonset
- heathcheck pods
- get pod CIDR from node.PodCidr when kube-controller-manager is run with `--allocate-node-cidrs=true` option

## Acknowledgement

Kube-router build upon following libraries:

- Iptables: https://github.com/coreos/go-iptables
- GoBGP: https://github.com/osrg/gobgp
- Netlink: https://github.com/vishvananda/netlink
- Ipset: https://github.com/janeczku/go-ipset
- IPVS: https://github.com/mqliang/libipvs
