# Perspectives for a geo-redundant Kubernetes cluster
This repo's purpose is to serve as a documentation of my research for possible implementations for a geo-redundant Kubernetes cluster. The research is part of a practical semester for university (dual study program) and will eventually serve as basis for a Bachelor thesis.

> Kubernetes clusters are like Pringles - you can't just have one! [[source]](https://linkerd.io/2020/02/17/architecting-for-multicluster-kubernetes/#)

Kubernetes is adopted all over the globe in production setups. While many companies run multiple clusters in order to separate workloads or simply
because multiple teams want to develop and test independently, having multiple clusters in order to reach HA and stability is also valid reason.
Ideas around multi-cluster Kubernetes deployments are just evolving, as this semi-official document shows. Also, current development efforts around
service meshes and kubefed (see below) show that this topic is anything but an old hand in the Kubernetes ecosystem.

## Organizational
### Strategy and expected outcome
At first, very generic solutions for solving geo-redundancy problems of K8s clusters are researched. The identified research focus is based on the company project this research is conducted at.
In the end, based on the research outcomes, an educated guess for the best fitting implementation strategy is attempted.

### Important note on sources and copyright
To see the sources of pictures not originating from my drawing skills, simply click on the picture, this will redirect to the respective website.
Sources are provided via links inside the text.

### Research topics
#### General Topics
When looking into solutions for replicating a Kubernetes cluster, four layers/ general topics can be identified:
1. Infrastructure: the general replication approach for the Kubernetes cluster (e.g. active-active or active-standby and implications, cluster
federation, service-mesh)
2. Middleware: How to replicate event-driven communication over a message bus (e.g. Kafka)? MoM multicluster deployment
3. Databases: cross datacenter deployment of databases, data consistency issues
4. Application: special requirements needed by the application itself

#### Cluster Components
The following basic components and functionality of the cluster at work have been identified:
* Storages (= databases): MongoDB (noSQL) and MariaDB (SQL)
* Services/ Ingress Endpoints: APIs (stateless, so not complicated to replicate geo-redundantly)
* Ingress: external calls to APIs (REST) over Kong API gateway
* Internal, event driven communication between microservices over Kafka message bus
* Egress: actor calls to external systems (e.g POD API)

### KPIs for measuring the solutions' suitability
When looking into different alternatives, several indices shall play a key role
* Latency: Kafka and Zookeeper are very latency sensitive
* Consistency: strong, eventual
* Durability/ data loss
* Networking overhead/ complexity

## 1. Infrastructure
This section describes how geo-redundant K8s clusters can be deployed from an infrastructure perspective.

Important aspects to consider:
* Cross-cluster communication (if any) and network topology: Kubernetes has inbuilt service discovery for Services in the cluster network.
Clients talk to a virtual IP and get routed to the selected pod. If two clusters were in the same flat network (e.g. via VPN) and would provide
non-conflicting CIDRs for their services, cross cluster communication would be easy because of direct routing of requests. On the other hand
in case of separate networks (as in separated datacenters), a solution for dynamic cross-cluster service discovery and communication is
needed. One could expose every single pod via a LoadBalancer or ExternalName Service, but this is not how simple scaling works.
* External systems to cluster communication, load balancing
* Deployment (symmetric or asymmetric, number of clusters, automation, how to deploy)
* Operation (active-active, active-standby, cluster failover, traffic mirroring?)

### Decisions and more desicions
First of all, one thing has to be considered: in case of a partial cluster failure, do we abort requests completely and redirect them to another cluster, or
do we try to forward the half processed request to another cluster's Services for final processing?

| Abort partially failed request | Forward for final processing |
| :----------------------------- | :--------------------------- |
| no inter-cluster communication needed | request and state (!) must be communicated to other cluster before further processing |
| "something" must abort the request and send 500-error (total failure will produce timeout) | what about latencies? |
| issuer of request must retry until working EMS is found | is it really needed? introduces a lot of complexity and additional tooling! |
| health checks for single Deplyoments/ Services must expand to the whole cluster, so that it will not be contacted until all services are healthy again |  |

### Comparison of alternatives
This section first considers solutions for deploying and managing redundant Kubernetes clusters in the most elegant way.
The next subsection is dedicated to infrastructure setups and their implementation.
Last but not least, several ways for realizing cross-cluster communication are evaluated (service meshes, mainly).

#### 1.1 Deployment and Maintenance Strategies
This subsection is about different strategies to deploy more than one Kubernetes cluster in different datacenters and how to keep their configuration
and versioning under control.

##### By Hand
One could deploy a cluster via whichever tool finds one's liking, and simply repeat the process for every redundant cluster needed. While this is
certainly no big deal for deplyoment, keeping all the clusters' configuration and microservices consistent and rolling out updates by hand simply does
not scale cost-wise. There are some pretty popular tools out there that simplify and assist in the process.

##### GitOps and Infrastructure as Code
Using tools like Terraform can automate provisioning the platform needed for a Kubernetes cluster (Infrastructure as Code), and a GitOps approach
will keep separate clusters in a well defined and consistent deplyoment state. Even canary deployments to one cluster are possible.
Both approaches speed up the provisioning of different clusters, so that even hundreds of clusters and thousands of nodes can be sporned within
minutes. They still aren't linked network-wise or accessible in any loadbalancing way, but at least they are up and running.

##### Gravity [[website]](https://gravitational.com/gravity/docs/)
I came across the Gravity project while googling for Kubefed (see below) and at least wanted to lose a few words about it. The project developed by Gr
avitational offers to compress a whole Kubernetes cluster (configuration, storage and everything) into a .tar file, which can be copied and unpacked on
"any infrastructure. All public clouds, private clouds and bare metal servers are supported." Additionally, it helps with managing the created clusters
from a single point, the gravity API.
Again, the clusters are not linked together in any way.

##### Kubefed [[website]](https://github.com/kubernetes-sigs/kubefed/blob/master/docs/userguide.md)
Federation v2, or kubefed, is a Kubernetes SIG (Special Interest Group). It is capable of creating and managing various federated Kubernetes clusters, originating from a single host cluster with the federation control plane, as depicted below.

Either the whole cluster or only selected ressources can be federated. Kubefed also offers management of the ressources, like versioning, canary deployments and so on. 

Like all the tools presented, kubefed aswell is not capable of connecting the resulting clusters. However, the documentation shows how to integrate ExternalDNS for accessing the clusters from a single infrastructure point.

#### 1.2 Infrastructure Setups
The concepts presented here are by no means spcial for a Kubernetes setup. They have been applied for years in datacenter and database planning.
Mainly two setups are possible:
* active-standby or
* active-active,
whereby active clusters receive traffic and the standby ones act as failover or desaster recovery.

##### Option 1: Active-Standby and Failover
This option means having one active Kubernetes cluster receiving all the traffic, and several standby ones waiting for the active to fail. The mechanism
is called failover.
The standby clusters can either be running all the time, which guarantees immediate failover, or they can be turned on only if needed, which causes a
delay in failing over. Having standby clusters turned off certainly saves a lot of money, especially with "pay as you go" infrastructure providers. On the
other hand, being able to route all traffic to a second cluster within seconds can be required in certain high frequency businesses.
So, how do you implement a failover mechanism with Kubernetes? Well, it depends on how the outside world communicates with your clusters. All
you need is a mechanism to detect the health of a cluster, and the possibility to redirect all the traffic to a new IP address.

I have come up with four solutions for this
1. The application does all the job. You can hardcode IP addresses to connect to, and if the first one is unavailable, the application tries the second one and so on. I do not really propose this as a good solution, but it would be possible and has been implemented in one or the other student project *cough*.
    1. Detecting cluster health: an unhealthy cluster would simply return a 500-ish HTTP error, et voilà.
    2. Redirecting traffic: please retry one of the other IPs.
2. Use a Load Balancer (or similar). This is especially attractive if you have one in place already, e.g. a reverse proxy-like entry point where clients connect to and get forwarded to a cluster. The Loadbalancer can simply route all traffic to the standby cluster and check the dead active one periodically. Do not forget to deploy the LB in HA mode aswell, as to not create a different SPOF!
    1. Detecting cluster health: the LoadBalancer does periodic health checks.
    2. Redirecting traffic: LB only forwards traffic to reachable clusters.
3. Use Next-Gen DNS servers. As described here, enhanced domain name servers are capable of registering multiple IP addresses for a
domain name, and can resolve only to healthy endpoints.
    1. Detecting cluster health: the DNS server is capable of issuing health checks and is aware of ressource status.
    2. Redirecting traffic: only healthy cluster IP addresses are resolved.
4. Federated API Gateways. From an outside perspective, the idea of federated API gateways is similar to using a Load Balancer that shifts the
traffic away from an unhealthy cluster. Gloo is the name of an Envoy based API gateway, which provides roughly the same capabilities as
Kong. However, solo.io, the company behind Gloo Gateway, went a step further and introduced Gloo Federation in 2020 as part of
(unfortunately) only their exterprise version of Gloo. In this technical deep divethe functionality of API gateway federation is explained. The
Gloo Federation Controller can be installed into an existing cluster with Gloo API Gateway or to a new cluster independent of any Gloo
instances or running application services. A detailed documentation for realising gateway failover can be found here.
    1. Detecting cluster health: The Envoy Proxy on which Gloo is built can determine the state of the endpoints the gateway exposes.
    2. Redirecting traffic: The Gloo Federation deployment will redirect failing traffic according to the configured FailoverScheme CRDs.
    3. As Gloo is based on the Envoy Proxy, it would certainly by manageable to implement this behavior in a custom project.

Just a quick hint for future sections: in this scenario it is especially important to reach eventual consistency of any state in the clusters, so that in case
of a failover the standby cluster has the same information as the former active one.

##### Option 2: Active-Active and Load Distribution
An active-active setup requires some intermediate proxy distributing the traffic (evenly or after some scheme). In case of an outage of one of the
clusters, the traffic is distributed only to the other healthy clusters. The proxy performs periodic health checks on the clusters to avoid forwarding to
offline clusters.

###### Load Balancer
One possible solution is the deployment of an upstream Load Balancer instance that receives all traffic and distributes it across the active clusters. It
needs to perform periodic health checks on the clusters to be able to leave out unhealthy ones. The traffic can either be distributed round robin or after
some weighted scheme, e.g. cluster workload.

###### DNS Server
The second solution involves a next gen DNS server with multiple A-records for a domain name. Based on periodic health checks it would only resolve
IPs of available clusters, again round robin or wighted. In order for this to work, Kubernetes ressources (e.g. the API gateway) need to be discoveryble
via this next gen DNS server, similar to the inbuilt Kubernetes DNS. Tools like ExternalDNS provide this functionality.

###### Comparison
When comparing the alternatives, the first two both are definitely feasible and widely used. While the Load Balancer needs to be deployed somewhere
for all incoming traffic to access, ExternalDNS is a Deployment inside the cluster that relies on one of the many supported external DNS providers on
the market. Gloo Federation goes forward with a little different approach, by realising the failover and load balancing and providing additional
configuration and management capabilities.
In this scenario reaching consistency is much harder and important than in the active-standby one, as requests can be forwarded to any healthy
cluster. Concepts like strong consistency and session affinity will be important, as well as latency of cluster syncronizing activities.

#### 1.3 Network Topology
A consideration which is mostly relevant for an active-active setup is the network topology used.
There are three obvious possibilities:
1. fully-connected networks (flat network, VPN): every pod can communicate with every other, unaware of which cluster they are deployed in;
no visible NAT or gateway; no overlapping pod CIDRs
2. partially-connected networks: Ingress and Egress points are the only cross-communication endpoints; IP ranges may or may not overlap
3. disconnected networks: the clusters are not "aware" of each other


#### 1.4 Cross-Cluster Communication
##### via exposed Services
Kubernetes offers the possibility to expose Services directly to the outside world via Services of types LoadBalancer or ExternalName. Additionally,
Ingress Controllers or API Gateways can be used to make Services accessible in a scalable way. However, this approach would mean to code into
the application's microservices a failover mechanism for switching to the external Services. There are certainly application agnostic ways to do this,
see below.
It is possible to create a VPN between the different clusters (e.g. full mesh). The Services then could be accessed directly via the usual Kubernetes
failover process.
https://medium.com/@chamilad/load-balancing-and-reverse-proxying-for-kubernetes-services-f03dd0efe80

##### via Service Mesh
First, what exactly is a service mesh? It manages the network and communication between microservices, e.g. encryption and certificates, monitoring
and telemetry or traffic shaping. Service meshes also provide Ingress and Egress capabilities for communication with the outside world. The service
mesh infrastructure is devided into a data plane (mesh traffic) and a control plane (service mesh controllers). Also take a look here for further
information.

In multi-cluster deployments, a service mesh serves as a bridge for cross-cluster communication. The primary problem it solves is partial failure of
Services. Deploying a service mesh across clusters enables the local pods to access services from another cluster, achieving forwarding of requests
in case local services become unavailable.

The provided Ingress Controller can also be leveraged for DNS cluster failover, however this would not be its primary purpose [link].

The number of service mesh projects out there for sure exceeds the number of fingers on both my hands. While this subsection does not compare the
service meshes in general, the ones considered here support multi-cluster deplyoment and cross-cluster communication one way or the other. So their
support for geo-redundant k8s deployments will be evaluated.

I only picked two projects for a more detailed description: istio and Cilium, as they are exemplary for the two approaches on cross-cluster Service
discovery. Cross-cluster communication is something we don't really need in A4-EMS, imho, and thus I did not want to spend too much time on it.
Other projects and links I took a look at are
* Hashicorp Consul
    * https://www.consul.io/docs/k8s/installation/multi-cluster
    * https://medium.com/@rahulgoffy/a-practical-guide-to-aws-elastic-kubernetes-service-cross-cluster-service-discovery-using-consul-654535bc3f2d
* Kuma
    * https://kuma.io/blog/2020/multi-cluster-cloud/
    * https://thenewstack.io/kuma-a-new-cncf-project-enhances-the-control-plane-for-mixed-infrastructure/
* Linkerd
    * https://linkerd.io/2/features/multicluster/
    * https://linkerd.io/2020/02/25/multicluster-kubernetes-with-service-mirroring/
    * https://linkerd.io/2020/06/09/announcing-linkerd-2.8/
    * cool: no new CRDs are introduced, remote Services get DNS names that reflect the cluster they reside in.
* Submariner
    * https://submariner.io/
    * https://github.com/submariner-io/submariner
    * https://rancher.com/blog/2019/announcing-submariner-multi-cluster-kubernetes-networking/
    * Flat network pod-to-pod communication

###### Istio [website]
Istio is certainly one of the most popular and mature service meshes. The project was started by teams from Google and IBM, in partnership with the
Envoy team at Lyft, and is in active development. It offers extensive capabilities at the cost of complexity and a steep learning curve.
The functionylity is build on the Envoy Proxy, a sidecar injected into every pod in the cluster. The istio control plane, istiod, manages the service mesh,
while the Envoy Proxies communicate with each other over the data plane. Istio also provides Ingress and Egress capabilities.

Istio operates at the Service level, meaning it integrates with DNS servers provided inside Kubernetes to expose Services across clusters.
Istio has several deployment modes in a multicluster installation: single or multiple network, single or multiple control plane, single or multiple mesh.
So you can either use it with a single shared control plane, or with replicated control planes in each cluster (klick here).
Shared control plane: This mode realises cross-cluster control plane access, so that a single istio control plane manages multiple istio
meshes in different clusters. All Kubernetes control plane API servers must be routable to each other. The clusters can either be in the same
network, or the istio gateways must be accessible form every other cluster. The pod and Service CIDRs must not overlap. Then, the Istiod
discovery service (control plane) on the primary cluster is exposed to the remote clusters.
Replicated control planes: Istio gateways are used to communicate via mTLS across clusters. These must be accessible from every cluster.
A new Service DNS domain is introduced, \*.global, and Kubernetes’ DNS must be configured to stub a domain for .global. For every Service
that shall be accessible from other clusters, a ServiceEntry is added.
In a multicluster installation, istio is even capable of full cluster failover because of the provided Ingress capabilities, although this is not its primary
purpose.

To sum it up, istio is a very mature service mesh solution that integrates multicluster support in a seamless and stable way. There is considerable
overhead when the service mesh is not deployed anyways.

###### Cilium Cluster Mesh [website]
The open source project is maintained by Isovalent, a California based startup. The first commit on GitHub was 5 years ago, the project is in active
development.
Cilium does not call itself a service mesh (more a network plugin), but it offers the common networking, observability and security capabilities. What
makes it very special is that it relies on the eBPF linux kernel technology, making it a OSI layer 3/4 operating tool capable of dealing with layer 7
traffic. In Kubernetes terms, it operates on the CNI (container networking interface) level instead of the application level and is totally IP and port
agnostic. The microservices and even k8s DNS are not aware of the connected clusters. It can thus be configured application independently.
The primarily targeted failure scenario is partial failure of selective services.
Its use of kernel modules requires the Cilium agents to run in privileged containers.

Requirements (klick here and here for a much more detailed listing):
* Linux kernel 4.9.17
* etcd should be exposed via NodePort or LoadBalancer
* CAP_SYS_ADMIN container privileges for Cilium agents
* Unique IP addresses for all k8s worker nodes and IP interconnectivity (peering or VPN tunnel)
* Non-conflicting PodCIDR ranges in each cluster
* Network between clusters must allow inter-cluster communication (direct routing or tunneling mode, firewall rules listed in docu)

With Cilium Cluster Mesh, pods can route directly across clusters via tunneling or flat network without gateways or proxies. From the application's view
a multi-cluster environment appears as a single cluster. Communication between all pods (also cross-cluster) is TLS encrypted.

Several layers of capabilities are offered, which can be chosen selectively:

How it works:
After deploying the Cilium Cluster Mesh, the services to be shared with other clusters, called global services, get annotated accordingly.
Cilium agents run in every cluster and are completely independent. They connect read-only to the exposed etcds via a proxy (TLS protected), and
watch for relevant changes.
Pods can reach other pods cross-cluster via their IPs, which are resolved by standard k8s DNS servers like CoreDNS or kubeDNS. Possible modes
are tunneling (similar to VPN tunnel), direct-routing and hybrid. Upon a service discovery request, the local cluster resolves the ClusterIP of the local
pod. Cilium then will take care of loadbalancing the request to a registered global service endpoints in all clusters based on k8s health-checks. DNS
servers are not aware of the external services and only return internal ClusterIPs Fine-grained control mechanisms are available as additional
annotations, in order to tune locality preferences or other traffic shaping.

tl;dr
Cilium Cluster Mesh works on top of eBPF with minimum overhead and enables loadbalanced pod to pod cross-cluster communication in directly inter-
connected cluster networks.

Info taken from [ https://cilium.io/blog/2019/03/12/clustermesh/] and [https://docs.cilium.io/en/v1.8/].

TODO
https://itnext.io/kubernetes-multi-cluster-networking-cilium-cluster-mesh-bca0f5367d84

##### Service Mesh Hub [website]
With Service Mesh Hub, heterogenous service meshes deployed in separate Kubernetes clusters can be connected via a unified management-plane.
The resulting virtual mesh is accessible via an unified API. It supports the cause of cross-cluster communication by offering a global service registry
across networks/ clusters. Providing this, it connects arbitrary individual services across independent clusters, using mTLS.
As shown in the picture, Service Mesh Hub is best deployed in a separate cluster.
More info: https://www.solo.io/blog/multi-cluster-service-mesh-failover-and-fallback-routing/


## 2. Middleware
When it comes to looking at the middleware used in A4-EMS, the Apache Kafka used together with the Actor model comes to mind.
Kafka resilience and disaster recovery
https://ebaytech.berlin/resiliency-and-disaster-recovery-with-kafka-b683a88aea0?gi=f3d63c9172b4

## 3. Databases/ storage
The big question when replicating k8s clusters is: do we have any stateful microservices deployed? If yes, then we need to spend a considerable
amount of time on application data consistency across the various clusters. What we need to avoid is called 'split brain': different clusters having
different data or application states, so that requests will be answered differently.
In distributed systems, several layers of consistency are discussed by Tanenbaum and van Steen in a whole chapter (who wrote the holy bible of
distributed systems [free download]).
Shadowing traffic: https://blog.christianposta.com/microservices/advanced-traffic-shadowing-patterns-for-microservices-with-istio-service-mesh/
uses Envoy Proxy
traffic is mirrored asynchronously and out of band from the production traffic. Any responses are ignored.
Question: Can K8s PV/ PVCs mirror the data themselves? So not the db is responsible for syncing state, but k8s itself?

## 4. Application

