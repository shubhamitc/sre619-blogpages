# Overview 
In almost every blog when people talk about deploying something on K8S they use node mostly because setting up http server is not that easy. I wish that it could be this easy for our case, (which it wasn't). Manly because the way we wanted the application to work, plans for *SEO* and multiple data pipelines for business including Amplitude for client and Pubsub for application and business metrics.

Making all these calls from nodejs was easy for developers just 1 more promise. Everything went smoothly as long as functional testing was required. We were ready to launch and somebody from SRE/Devops team asked: *"Have you done load testing?"*. Everything was superb up to this point where load testing results are required to make service live. 

# Load Testing
We started to put load to our services using apache benchmark. Very soon we realised our application is not scaling as much as we expected. To our surprise application was able to handle only `3 requests/second`. 

With drastic results and tight timelines, it soon became extremely difficult at various technical and emotional levels. Teams started to blame each other, stating either infra issues or code issues. It was SRE teams' first encounter with NodeJs single server side pages which made them uncomfortable. SRE team lead and principal developer started to spend a lot of time to figure out the bottleneck. 

Initial thoughts were around `event loop` time and CPU bound operations. Code was making some call to message strings which showed up on cpu head graph. We fixed that in a hope that everything will be alright, well it didn't. We have to go back and look the problem at different level. 

# Code Proxy
Developer wrote the code keeping 2 folders client and server. Client code was supposed to build and ship to client while server code was suppose to run on Kubernetes and facilitate SEO. In order to achieve that developers took library/vendor kind of approach where common code was kept. Downside of this was API call code was also common except for the fact that server to server call destination is different from client i.e browser to server call. To solve this developers asked SRE team to provide 1 DNS for both. However if you are in GCP(GKE service) and calling another GKE service URL with public IP , RTT is as high as 150ms. If you care to access the same over private DNS RTT is 3ms. 


With this in mind developers made changes to API module and made sure that it understands which context it is running at. With above change application performance increased from *RPS(Request per second) 3 to 7*. 

# Making calls is expensive
Well, compare to 3 - 7 was good, however not what we wanted. This new system was replacement of old monolith which was service Global traffic of 12k r/s. Considering that number of instances required to handle the traffic was beyond comprehension, for at least that time. We instrumented our code with NewRelic to debug deeper. Very soon we saw that DNS resolution is taking as high as 90ms for our containers. 

## Solution
Since nodejs delegates tasks like network calls, dns resolution to LIVUB, consider increase `LIBUV_THREADPOOL_SIZ` to maximum i.e. 128. 
On top of that if possible reduce calls to other services. Possibly use facade pattern to another java like service which can provide you all the data at single call. 
Because of LIBUV application performance is inversely propositional to api calls you make
Scale
Since NodeJs app was not scaling well with CPU usage because low cpu usage and limit of libuv thread pool size. Kubernetes was not able to scale our application that time, since it was not supporting scale based on custom metrics. We have to upgrade our Kubernetes to version 1.9 and above to achieve that. Since our application was already desined to export prometheus metrics, we utilised that to export `queries_per_second`[Queries per second] metric and made below changes in deployment configuration, 

# Deployed custom controller on GKE
First thing first, we need to upgrade to Kubernetes version 1.9 to support metrics based autosacling and code change to add 1 custom metric `queries_per_second`. With below in mind we experimented with [GKE Autoscaler document](https://cloud.google.com/kubernetes-engine/docs/tutorials/custom-metrics-autoscaling) and found to provide the results that we want on staging environment. In a nutshell autoscaler requires 3 things:
* Deploy custom metric stackdriver adaptor
* Export metric to stackdriver
* Deploy HPA settings to scale deployment based on cutom metrics exported by step-2 

### Deploy adaptor and rolebindings
GKE requires to use clusterrolebindings to allow adaptor to access cluster resources. In order to do that one need to run below command with clusteradmin privilege. 

```
kubectl create clusterrolebinding cluster-admin-binding \
 --clusterrole cluster-admin --user "$(gcloud config get-value account)"
```
Deploy the adaptor in cluster 
```
kubectl create -f https://raw.githubusercontent.com/GoogleCloudPlatform/k8s-stackdriver/master/custom-metrics-stackdriver-adapter/deploy/production/adapter.yaml
```
### Export app metrics to stackdriver
Final content of deployment file was modified to include side car to push data to stackdriver. We have followed the recommendation to setup this however now Kubernetes also allows us to scaled baesd on third party metrics. Side car is running with Image[image: gcr.io/google-containers/prometheus-to-sd:v0.2.3]. 

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "7"
  creationTimestamp: 2018-05-22T11:47:43Z
  generation: 164
  labels:
    app: browser2.0
    env: production
    harness-app: browser2.0
    harness-env: production
    harness-revision: "201"
    harness-service: browser2
    revision: "49"
    run: viu-browser2
    service: browser2
  name: browser2.0.browser2.201
  namespace: viu-browser2
spec:
  replicas: 2
  selector:
    matchLabels:
      harness-app: browser2.0
      harness-env: production
      harness-revision: "201"
      harness-service: browser2
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: browser2.0
        env: production
        harness-app: browser2.0
        harness-env: production
        harness-revision: "201"
        harness-service: browser2
        revision: "49"
        run: viu-browser2
        service: browser2
    spec:
      containers:
      - env:
        - name: GLUSTER_ENABLED
          valueFrom:
            configMapKeyRef:
              key: gluster_enabled
              name: viu-browser2-browser-vuclip-config
        - name: PUB_SUB_ENABLED
          valueFrom:
            configMapKeyRef:
              key: pub_sub_enabled
              name: viu-browser2-browser-vuclip-config
        - name: NR_ENABLED
          valueFrom:
            configMapKeyRef:
              key: nr_enabled
              name: viu-browser2-browser-vuclip-config
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: spec.nodeName
        - name: NODE_IP
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.hostIP
        - name: POD_IP
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.podIP
        - name: HOST_URL
          value: https://browser.vuclip.com
        - name: NEW_RELIC_LICENSE_KEY
          value: <Nr license key>
        - name: NEW_RELIC_APP_NAME
          value: browser2
        - name: NEW_RELIC_ENABLED
          value: "true"
        - name: SERVICE_NAME
          value: viu-browser-1
        - name: SERVICE_8000_CHECK_HTTP
          value: /metrics
        - name: SERVICE_8000_CHECK_INTERVAL
          value: 15s
        - name: SERVICE_8080_CHECK_TIMEOUT
          value: 1s
        - name: NODE_OPTS
          value: --max-old-space-size=800 
        - name: NODE_ENV
          value: production
        - name: SERVICE_TAGS
          value: env=prod,kubernetes,stack=viu-browser,type=app,service=browser2,job=browser2
        image: us.gcr.io/ppp-prod/viu-browser2:v435
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /health
            port: 8000
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 3
          successThreshold: 1
          timeoutSeconds: 1
        name: viu-browser2
        ports:
        - containerPort: 8000
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /health
            port: 8000
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 3
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          limits:
            cpu: "1"
            memory: 1717986918400m
          requests:
            cpu: 800m
            memory: 1717986918400m
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /opt/general-shared-dir
          name: general-shared-dir
        - mountPath: /mnt/secrets-config
          name: secret-configs
      - command:
        - /monitor
        - --source=:http://localhost:8000/metrics
        - --stackdriver-prefix=custom.googleapis.com
        - --pod-id=$(POD_ID)
        - --namespace-id=$(POD_NAMESPACE)
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        - name: POD_ID
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.uid
        image: gcr.io/google-containers/prometheus-to-sd:v0.2.3
        imagePullPolicy: IfNotPresent
        name: prometheus-to-sd
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      volumes:
      - emptyDir: {}
        name: general-shared-dir
      - name: secret-configs
        secret:
          defaultMode: 420
          items:
          - key: keyfile.p12
            mode: 511
          secretName: browser2
```
### HPA setting
Below is custom metric with 2 components, 
* Based on CPU
* Based on custom metric `queries_per_second`
* Setup theshhold for scaling where CPU is setup for 30% average usage among pods, custom metric is setup to support 9 request/second among pods of deployment **browser2.0.browser2.201**

Highlighting the block for same:
```
- type: Pods
    pods:
      metricName: queries_per_second
      targetAverageValue: 9
```
**Complete HPA file**
```
apiVersion: "autoscaling/v2beta1"
kind: "HorizontalPodAutoscaler"
metadata:
  finalizers: []
  name: "browser2.0.browser2.production-40"
  namespace: "viu-browser2"
  ownerReferences: []
spec:
  maxReplicas: 60
  minReplicas: 40
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: "Deployment"
    name: "browser2.0.browser2.201"
  metrics:
  - type: "Resource"
    resource:
      name: "cpu"
      targetAverageUtilization: 30
  - type: Pods
    pods:
      metricName: queries_per_second
      targetAverageValue: 9
```
Below


# Akamai ...
With above configrations we have a stable application now. We have applied a few caching in code in order to avoid network calls. Even with all of this our application response time was a little off, so we made sure that content is served with local caching where akamai helps us to do that. We have setup a *GCS bucket* to store js, css and images of every build and we push that to GCS. Akamai is configured to serve static JS from this bucket. 

## TODO 
* Make sure application can serve Dynamic content via akamai. 








