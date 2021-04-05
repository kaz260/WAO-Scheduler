# MinimizePower plugin
Kubernetes is a portable, extensible, open source platform for facilitating declarative configuration management and automation, and managing containerized workloads and services. Kubernetes has a huge and fast-growing ecosystem with a wide range of services, support and tools available.
MinimizePower plugin is one of Kubernetes' scheduler plugin implementations that minimizes power growth when placing workloads on nodes.
Architecture overview

## Install
### Building from source
To build MinimizePower plugin from source code, first ensure that have a working Go environment with version 1.15.1 or greater installed. You also  need Kubernetes  with verson 1.19.7.
In addition, it is necessary to prepare a power consumption model for each node and make it available for reference by Tensorflow Serving and ipmi_exporter should be installed in all nodes.
```
$ curl -L -o kubernetes.tar.gz https://github.com/kubernetes/kubernetes/archive/v1.19.7.tar.gz
$ tar xvf kubernetes.tar.gz
$ curl -L -o https://github.com/kaz260/WAO-Scheduler
$ cp plugins ~kubernetes/pkg/scheduler/framework/plugins
$ go get github.com/prometheus/prom2json
$ cd ~kubernetes/cmd/kube-scheduler/
$ CG0_ENABLE=0 go build -mod=mod scheduler.go
```
## Deploy to Kubernetes
1. build Docker image of scheduler using Dockerfile like this:
```ROM busybox
ADD ./kube-scheduler /usr/local/bin/kube-scheduler
```
2. now build Docker image of scheduler using Dockerfile like this:
```
$ docker build t [your-local-repositry]/[image-name] .
$ docker image push [your-local-registry]/[image-name]
```
3. launch metrics-server.
```
$ kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
4. create ConfigMap
```
$ kubectl create -f oc_configmap.yaml
```
5. launch tensorflow serving
```
$ kubectl create -f tensorflow-server-dep.yaml
```
6. give each node the following label;
- ambient/max: Maximum ambient temperature in celsius.
- ambient/min: Minimum ambient temperature in celsius.
- cpu1/max: Maximum CPU1 temperature in celsius.
- cpu1/min: Minimum CPU1 temperature in celsius.
- cpu2/max: Maximum CPU2 temperature in celsius.
- cpu2/min: Minimum CPU2 temperature in celsius.
- tensorflow/host: IP address of tensorflow serving.
7. launch MinimizePower scheduler.
```
$ kubectl create -f oc-scheduler-deployment.yaml
```
8. Check if minimizePower was started normally. Success if the pod status changes to "Running".
```
$ kubectl get pod -n kube-system -o wide | grep oc-scheduler
```
9. If you want to use oc-scheduler, please refer test.yaml.
```
$ kubectl apply -f test.yaml
```
## License
Apache License 2.0, see LICENSE.
