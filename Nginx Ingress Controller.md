## Nginx Ingress Controller and Nginx ingress on OpenShift

### Install MetalLB Operator

### Create a Metal LB instance

```yaml
apiVersion: metallb.io/v1beta1
kind: MetalLB
metadata:
  name: metallb
  namespace: metallb-system
spec: {}
```

### Create the IP Address Pool
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ip-addresspool-sample1
  namespace: metallb-system
spec:
  addresses:
  - 10.10.11.42-10.10.11.44
  - 10.10.11.144-10.10.11.144
  autoAssign: true
  avoidBuggyIPs: false
```

### Create the L2advertisement

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  creationTimestamp: "2025-11-20T06:35:20Z"
  generation: 1
  name: l2-adv-sample1
  namespace: metallb-system
spec:
  ipAddressPools:
  - ip-addresspool-sample1
```

## Install the Nginx Ingress Controller Operator.

The Nginx Ingress Controller has to coexist as an additional ingress controller on the Red Hat OpenShift Cluster.

### Get the supplemental group

```bash
oc describe project nginx-ingress
```

spec.pod.runAsUser will be set as the value of **1000800000**

### Wildcard certificate

spec.wildcardTLS.secret will be set to the value of **openshift-ingress/router-certs-oct-2025**

### Create an Nginx Controller.

```yaml
kind: NginxIngress
apiVersion: charts.nginx.org/v1alpha1
metadata:
  name: nginxingress-sample
  namespace: nginx-ingress
spec:
  controller:
    affinity: {}
    annotations: {}
    appprotect:
      enable: false
    appprotectdos:
      debug: false
      enable: false
      maxDaemons: 0
      maxWorkers: 0
      memory: 0
    autoscaling:
      annotations: {}
      behavior: {}
      enabled: false
      maxReplicas: 3
      minReplicas: 1
      targetCPUUtilizationPercentage: 50
      targetMemoryUtilizationPercentage: 50
    config:
      annotations: {}
      entries: {}
    containerPort:
      http: 80
      https: 443
    customConfigMap: ''
    customPorts: []
    defaultHTTPListenerPort: 80
    defaultHTTPSListenerPort: 443
    defaultTLS:
      cert: ''
      key: ''
      secret: ''
    disableIPV6: false
    dnsPolicy: ClusterFirst
    enableCertManager: false
    enableCustomResources: true
    enableExternalDNS: false
    enableLatencyMetrics: false
    enableOIDC: false
    enableSSLDynamicReload: true
    enableSnippets: false
    enableTLSPassthrough: false
    env: []
    extraContainers: []
    globalConfiguration:
      create: false
      spec: {}
    healthStatus: false
    healthStatusURI: /nginx-health
    hostNetwork: false
    hostPort:
      enable: false
      http: 80
      https: 443
    image:
      pullPolicy: IfNotPresent
      repository: nginx/nginx-ingress
      tag: 5.2.1-ubi
    ingressClass:
      create: true
      name: nginx
      setAsDefaultIngress: false
    initContainerResources:
      requests:
        cpu: 100m
        memory: 128Mi
    initContainers: []
    kind: deployment
    lifecycle: {}
    logFormat: glog
    logLevel: info
    mgmt:
      licenseTokenSecretName: license-token
    minReadySeconds: 0
    name: controller
    nginxDebug: false
    nginxReloadTimeout: 60000
    nginxStatus:
      allowCidrs: 127.0.0.1
      enable: true
      port: 8080
    nginxplus: false
    wildcardTLS:
      cert: ''
      key: ''
      secret: openshift-ingress/router-certs-oct-2025
    pod:
      runAsUser: **1000800000**
      annotations: {}
      extraLabels: {}
    podDisruptionBudget:
      annotations: {}
      enabled: false
    readOnlyRootFilesystem: false
    readyStatus:
      enable: true
      initialDelaySeconds: 0
      port: 8081
    replicaCount: 1
    reportIngressStatus:
      annotations: {}
      enable: true
      enableLeaderElection: true
      ingressLink: ''
      leaderElectionLockName: nginx-ingress-leader
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
    selectorLabels: {}
    service:
      annotations: {}
      clusterIP: ''
      create: true
      customPorts: []
      externalIPs: []
      externalTrafficPolicy: Local
      extraLabels: {}
      httpPort:
        enable: true
        port: 80
        targetPort: 80
      httpsPort:
        enable: true
        port: 443
        targetPort: 443
      loadBalancerIP: ''
      loadBalancerSourceRanges: []
      type: LoadBalancer
    serviceAccount:
      annotations: {}
      imagePullSecretName: ''
      imagePullSecretsNames: []
    shareProcessNamespace: false
    strategy: {}
    terminationGracePeriodSeconds: 30
    tlsPassthroughPort: 443
    tolerations: []
    volumeMounts: []
    volumes: []
    watchNamespace: ''
    watchNamespaceLabel: ''
    watchSecretNamespace: ''
  nginxServiceMesh:
    enable: false
    enableEgress: false
  prometheus:
    create: true
    port: 9113
    scheme: http
    secret: ''
    service:
      create: false
      labels:
        service: nginx-ingress-prometheus-service
    serviceMonitor:
      create: false
      endpoints:
        - port: prometheus
      labels: {}
      selectorMatchLabels:
        service: nginx-ingress-prometheus-service
  rbac:
    create: true
  serviceInsight:
    create: false
    port: 9114
    scheme: http
    secret: ''
```
