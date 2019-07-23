Istio lets you connect, secure, control, and observe services. At a high level, Istio helps reduce the complexity of these deployments, and eases the strain on your development teams. It is a completely open source service mesh that layers transparently onto existing distributed applications. It is also a platform, including APIs that let it integrate into any logging platform, or telemetry or policy system. Istio’s diverse feature set lets you successfully, and efficiently, run a distributed microservice architecture, and provides a uniform way to secure, connect, and monitor microservices. In context of Vuclip istio allows us to reduce the code and environment configurations while keeping the similar or more feature sets at our disposal. 
Since istio is designed to bridge the gap for both development teams and SRE, it is essential to see and visualize that in practice. Istio will affect us in our ability to connect , secure(HTTPs TLS, mtls [Phase-2]), control(external communication , rbac) and observe(monitoring with prometheus and tracing). 

Since ISTIO is very new to us we are selectivity not investing much on Mutual TLS(mtls). `One feature i would like to stress is the istio's ability towards observations.`

# Tech stack 
For some people it makes a lot of sense to know the relevance thus here you go
* Springboot
* GKE(*G*oogle *K*ubernetes *E*ngine)
* GCP (*G*oogle *C*loud *P*latform)
* Something browed from Netflix
  * Hystrix
  * Ribbon 
* Consul for service discovery
* Prometheus + NewRelic for monitoring
* Storage: supported DBs like CloudSql, Cache like Redis and Hazelcast, MQ like RabbitMQ and Cloud Pub/Sub

# Credits
- [Envoy XDS APIs](https://blog.envoyproxy.io/the-universal-data-plane-api-d15cec7a) 
- [Control plane APIs]( https://www.envoyproxy.io/docs/envoy/latest/api-v2/api)

# Conceptual architecture
It may be difficult to conceptualise Istio in general considering it takes care of a few tasks at a time, like loadbalancing, service discovery, fault injetions, etc... It is best to look it as a netowrk devise like switch in context of OpenFlow protocol. 
[![Openflow Control planes](https://github.com/shubhamitc/sre619-blogpages/blob/master/istio-doc/openflow-context.png?raw=true&size=100x20)](https://sre619.blogspot.com). Each switch device in openflow requires next forward path, which can either be pushed by Openflow controller or switch can pull this information from controller. In Istio each envoy works like a switch and Pilot provides information for next hop using adopters. One can see similar setup with Istio planes: 
[![Istio Planes](https://github.com/shubhamitc/sre619-blogpages/blob/master/istio-doc/istio-plane.png?raw=true&size=100x20)](https://sre619.blogspot.com).

# Istio planes explained
## Mixer
Mixer is a platform-independent component. Mixer enforces access control and usage policies across the service mesh, and collects telemetry data from the Envoy proxy and other services. The proxy extracts request level attributes, and sends them to Mixer for evaluation. You can find more information on this attribute extraction and policy evaluation in our Mixer Configuration documentation.
Mixer includes a flexible plugin model. This model enables Istio to interface with a variety of host environments and infrastructure backends. Thus, Istio abstracts the Envoy proxy and Istio-managed services from these details.

## Pilot
Pilot provides service discovery for the Envoy sidecars, traffic management capabilities for intelligent routing (e.g., A/B tests, canary deployments, etc.), and resiliency (timeouts, retries, circuit breakers, etc.).
Pilot converts high level routing rules that control traffic behavior into Envoy-specific configurations, and propagates them to the sidecars at runtime. Pilot abstracts platform-specific service discovery mechanisms and synthesizes them into a standard format that any sidecar conforming with the Envoy data plane APIs can consume. This loose coupling allows Istio to run on multiple environments such as Kubernetes, Consul, or Nomad, while maintaining the same operator interface for traffic management.

## Citadel
Citadel provides strong service-to-service and end-user authentication with built-in identity and credential management. You can use Citadel to upgrade unencrypted traffic in the service mesh. Using Citadel, operators can enforce policies based on service identity rather than on network controls. Starting from release 0.5, you can use Istio’s authorization feature to control who can access your services.
## Galley
Galley validates user authored Istio API configuration on behalf of the other Istio control plane components. Over time, Galley will take over responsibility as the top-level configuration ingestion, processing and distribution component of Istio. It will be responsible for insulating the rest of the Istio components from the details of obtaining user configuration from the underlying platform (e.g. Kubernetes).

## Data Plane brief component description

### Envoy 
Istio uses an extended version of the Envoy proxy. Envoy is a high-performance proxy developed in C++ to mediate all inbound and outbound traffic for all services in the service mesh. Istio leverages Envoy’s many built-in features, for example:
- Dynamic service discovery
- Load balancing
- TLS termination
- HTTP/2 and gRPC proxies
- Circuit breakers
- Health checks
- Staged rollouts with %-based traffic split
- Fault injection
- Rich metrics

# How we roll... 
Before i describe them in detail i would like to state a few factors to provide some context:
* In order to support easy governance each team gets their own project in GCP
* Most of the access to the project is maintained by the same team. However SRE team will setup another project to configure storage services like RabbitMQ, Redis, Elasticsearch, etc... 
* All of them will be connected using Shared VPC from host project
* This setup allows us to transition deployments to Development teams and 

# Challenges
1. Language and framework
2. Serviec discovery 
3. Circuit breaker
4. Mapping services across clusters 
5. Regional services and their handling (AKA deployment considerations)

## 1. Language and framework
In order to have granular governanace we devided clusters to different project, this scenario cuts both ways, at one hand it solves some of governanace on other hand it creates a lot of problems for cluster creating and provisioning. Just for fun imagine these clusters running across regions... 
Out first attempt was to setup application with Netflix stack which was awesome for a while. We were running springboot 1.xx and created our spring template so all the applications have Hystrix, consul and ribbon for service discovery and prometheus for monitoring. Good right? just one teeny tiny problem, It started to take weeks in order to get the last mile certified for production while incorporating all these changes and getting them verified by teams. To add to this somebody came up and said, "you know what Spring released versoin 2.0 !". Now somebody has to upgrade all the services, bring them to 2.0 standards and create template for everything all over again. FYI default prometheus metrics are chagned in Spring 2.0, so good luck updating all the grafana dashboards.

## 2. Service Discovery
As i described before we are using Consul for service discovery, while we migration from DC to Cloud we wanted to have the same code base. Consul was a good fit for us (Wan discovery and near field detection) initially as we had plans to run both DC and Cloud which got changed in course of 1 year and 6 months. We decided to move everything to cloud.

First challenge was super micro services:
Consul based Service discovery was fine but we wanted our microservices to become more micros, by that I mean that these libraries should be taken out of codebase. We selected Istio based on our GCP roadmap and very soon reliased that Consul based service discovery will not work for us as Istio requires `Host headers` to identify a service.

Second challenge Istio with different clusters:
While we were running Consul on VMs, we were able to find services across Kubernetes clusters. Once we moved to Istio we were forced to used native service discovery. Now please be aware that Kubernetes native discovery that is service type ClusterIP is only routable within the cluster. So if you want to access services across you need to expose your services probably using LoadBalancers. This exercise was adding 40MS delay to our calls since they have to go to Public Ips now. 
To add to this Kubernetes with Istio now supports Single control plane architecture, which is control plane may reside at one place while dataplane running on different cluster will talk to same control plane. Such setting allows you to have single discovery of services. 
[![Istio Planes](https://istio.io/docs/concepts/multicluster-deployments/multicluster-with-vpn.svg?raw=true&size=100x20)](https://istio.io/docs/concepts/multicluster-deployments/).

Final architecture with Shared VPC and multiple projects will look something like:
[![Istio Planes](https://github.com/shubhamitc/sre619-blogpages/blob/master/istio-doc/network-model2.png?raw=true&size=100x20)](https://sre619.blogspot.com).
Where Flow showing Nginx is our current flow while Envoy show new traffic flow using Istio. 

## 3. Circuit breaker 
There is some difference between the way circuit breaker is implimented in both Istio and Hystrix. While Hystrix is completly instance specific and languge native , Istio is an infrastructure piece thus focus is moved towards governance as well. To explain this, Istio moves functionality for circuit breaking to server side while keeping the functionality of timeout to client side. This allows producer to control how many concurrent request it can handle and when to open circuit. 
* How to open circuit for same host from 2 consumers having different SLAs 
* How to provide context back to caller in terms for circuit break. Since Istio is not inside code, context can not be passed to application
* It is difficult to rewrite the logic of fallback, compared to Hystrix 

While there are 2 possible solutions to first problem, others are difficult to solve:
1. Have alias service name for producer
2. Pass context aware headers and create different virtual services based on the header match with different timeouts

## 4. Mapping services across clusters 
Once you setup Istio with above logic you need to follow below which is suggested by Istio docs:
`Kubernetes resolves DNS on a cluster basis. Because the DNS resolution is tied to the cluster, you must define the service object in every cluster where a client runs, regardless of the location of the service’s endpoints. To ensure this is the case, duplicate the service object to every cluster using kubectl. Duplication ensures Kubernetes can resolve the service name in any cluster. Since the service objects are defined in a namespace, you must define the namespace if it doesn’t exist, and include it in the service definitions in all clusters.`
So one needs to make sure that service definations and namespaces are replicated.

## 5. Regional services and their handling (AKA deployment considerations)
Istio document suggests below:
`The previous procedures provide a simple and step-by-step guide to deploy a multi cluster environment. A production environment might require additional steps or more complex deployment options. The procedures gather the endpoint IPs of the Istio services and use them to invoke Helm. This process creates Istio services on the remote clusters. As part of creating those services and endpoints in the remote cluster, Kubernetes adds DNS entries to the kube-dns configuration object.
This allows the kube-dns configuration object in the remote clusters to resolve the Istio service names for all Envoy sidecars in those remote clusters. Since Kubernetes pods don’t have stable IPs, restart of any Istio service pod in the control plane cluster causes its endpoint to change. Therefore, any connection made from remote clusters to that endpoint are broken. This behavior is documented in Istio issue #4822
To either avoid or resolve this scenario several options are available. This section provides a high level overview of these options:
`
1. Update the DNS entries - You are suppose to re-run remote deploy. Which is not fisible in my case
2. Use a load balancer service type - Internal loadbalancers are only available in same region. To explain this you can not deploy the control plane to work for all the regions. Public LBs are too great a risk for security. From Google docs: `Internal TCP/UDP Load Balancing makes your cluster's services accessible to applications outside of your cluster that use the same VPC network and are located in the same GCP region. For example, suppose you have a cluster in the us-west1 region and you need to make one of its services accessible to Compute Engine VM instances running in that region on the same VPC network.`
3. Expose the Istio services via a gateway - Same issue as Public LB
### Possible Solutions
1. You need to run 1 control plane per region to handle security
2. You need to do step-1 because NEG (Network Endpoint Groups) and Ingress controllers should have regional mapping in Backend service to balance traffic across regions with near instance ability (Regional failover)

