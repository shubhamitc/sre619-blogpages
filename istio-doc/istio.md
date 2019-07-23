# Introduction
Istio lets you connect, secure, control, and observe services. At a high level, Istio helps reduce the complexity of these deployments, and eases the strain on your development teams. It is a completely open source service mesh that layers transparently onto existing distributed applications. It is also a platform, including APIs that let it integrate into any logging platform, or telemetry or policy system. Istio’s diverse feature set lets you successfully, and efficiently, run a distributed microservice architecture, and provides a uniform way to secure, connect, and monitor microservices. In context of Vuclip istio allows us to reduce the code and environment configurations while keeping the similar or more feature sets at our disposal. 
Since istio is designed to bridge the gap for both development teams and SRE, it is essential to see and visualize that in practice. Istio will affect us in our ability to connect , secure(HTTPs TLS, mtls [Phase-2]), control(external communication , rbac) and observe(monitoring with prometheus and tracing). 

Since ISTIO is very new to us we are selectivity not investing much on Mutual TLS(mtls).

# Audience
- Application and system architects
- Development teams 
- SRE team
- Smart alert and monitoring specialists 

# Some pre-reading reference and credits
- [Envoy XDS APIs](https://blog.envoyproxy.io/the-universal-data-plane-api-d15cec7a) 
- [Control plane APIs]( https://www.envoyproxy.io/docs/envoy/latest/api-v2/api)

# Conceptual architecture

