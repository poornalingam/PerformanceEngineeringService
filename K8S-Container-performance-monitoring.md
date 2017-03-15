# Kubernetes cluster and Docker container performance analysis

## Kubernetes cluster monitoring

In a Kubernetes cluster, application performance can be examined at many different levels: containers, pods, services, and whole clusters. It gives users deep insights into how their applications are performing and where possible application bottlenecks may be found. In comes Heapster, a project meant to provide a base monitoring platform on Kubernetes.

### Heapster
Heapster is a cluster-wide aggregator of monitoring and event data. It currently supports Kubernetes natively and works on all Kubernetes setups. Heapster runs as a pod in the cluster, similar to how any Kubernetes application would run. The Heapster pod discovers all nodes in the cluster and queries usage information from the nodes’ Kubelets, the on-machine Kubernetes agent. The Kubelet itself fetches the data from cAdvisor. Heapster groups the information by pod along with the relevant labels. This data is then pushed to a configurable backend for storage and visualization. Currently supported backends include InfluxDB (with Grafana for visualization), The overall architecture of the service can be seen below:

![K8S monitoring archtecture](/Images/K8S-monitoring-heapster.png)

#### cAdvisor
cAdvisor is an open source container resource usage and performance analysis agent. It is purpose built for containers and supports Docker containers natively. In Kubernetes, cadvisor is integrated into the Kubelet binary. cAdvisor auto-discovers all containers in the machine and collects CPU, memory, filesystem, and network usage statistics. It is a running daemon that collects, aggregates, processes, and exports information about running containers. Specifically, for each container it keeps resource isolation parameters, historical resource usage, histograms of complete historical resource usage and network statistics. This data is exported by container and machine-wide.

[Here](https://github.com/kubernetes/heapster/blob/master/docs/storage-schema.md) is list of metrics collected by cAdvisor. These metrics are stored in InfluxDB. All these metrics available through REST APIs.

#### Kubelet
The Kubelet acts as a bridge between the Kubernetes master and the nodes. It manages the pods and containers running on a machine. Kubelet translates each pod into its constituent containers and fetches individual container usage statistics from cAdvisor. It then exposes the aggregated pod resource usage statistics via a REST API.

#### Prometheus
Prometheus is an open source telemetric observability to collect VM and container metrics. Its being explored to use in GCP environment. It has enhanced capability for custom alerting.

Will discuss about it capabilities later. TBD.

#### Telegraph
Telegraph is an open source telemetric observability to collect VM and container metrics. its being explored to use in OpenStack environment.

Will discuss about it capabilities later. TBD.

## Docker container monitoring
Resource level monitoring can be collected using heapster, Telegraph and Prometheus. Still in search of good Kernel level metrics collection tool for performance observability. It need to collect data from the Linux kernel control groups (cgroup) and from the namespace of the container and expose them through a REST API. Need to assess the IOSP and overload before choosing the agent.

Combination of Vector and PCP collector agent are one of the possible options for Kernel level analysis. Yet to be analyzed in detail.

## Container Performance analysis
Playbooks TBD
Analyze the performance in host linux OS namespace and cgroup level. Will have playbook for this topic.

[Back to top] (#kubernetes-cluster-monitoring)
