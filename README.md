# WAO-Scheduler plugin

---
Kubernetes is a portable, extensible, open source platform for facilitating declarative configuration management and automation, and managing containerized workloads and services. Kubernetes has a huge and fast-growing ecosystem with a wide range of services, support and tools available.
This Document shows the steps to build and deploy Workload Allocation Optimizer (WAO)-Scheduler with power minimizing policy.

---

## Architecture overview
WAO-scheduler gets the CPU usage of each node with Metrics-server, also gets the ambient temperature and CPU temperature with ipmi_exporter. Then WAO-Scheduler predicts power increases with Tensorflow-serving and score the nodes. Finally, WAO-Scheduler selects the node that is expected to the power increases is minimum.
![architecture](https://user-images.githubusercontent.com/2385205/113538962-8a155480-9617-11eb-9215-77ddffdaa5b7.png)

## Prerequisites

* [Go-lang v1.15.1](https://golang.org/)
* [Kubernetes v1.19.7](https://github.com/kubernetes/kubernetes/releases/tag/v1.19.7)
* [ipmi_exporter](https://github.com/soundcloud/ipmi_exporter)  
  Running on each node
* local docker repository  
* docker image of tensorflow serving containing power consumption model of each node.  

## Build WAO-Scheduler with power minimization policy

1. Download the Kubernetes source code


    *Since the directory structure is different in the latest version, use [Kubernetes v1.19.7](https://github.com/kubernetes/kubernetes/releases/tag/v1.19.7)
    ```
    curl -L -o kubernetes.tar.gz https://github.com/kubernetes/kubernetes/archive/v1.19.7.tar.gz
    tar xvzf kubernetes.tar.gz
    ```
2. Add WAO-Scheduler source code

```
$ git clone https://github.com/kaz260/WAO-Scheduler
$ cp plugins ~kubernetes/pkg/scheduler/framework/plugins
```
  *Overwrite the file with the same name

3. Add the requied packages
    Add the prom2json package
    ```
    go get github.com/prometheus/prom2json
    ```
4. Build a WAO-Schduler
    Build result is output to `kubernetes/cmd/proxy/`
    ```
    cd ~kubernetes/cmd/kube-scheduler/
    CG0_ENABLE=0 go build -mod=mod scheduler.go
    ```
## Deploy to Kubernetes

1. Create a Docker image for scheduler

    Create `Dockerfile` with the following contents.
    The oiginal image has been confirmed and may be up to date.
    ``` Dockerfile
    FROM busybox
    ADD ./kube-scheduler /usr/local/bin/kube-scheduler
    ```
    Copy the `proxy` built in the above steps to the same directory as the `Dockerfile`.
    Create an image and push it to your local repositoty.

    ``` 
    docker build -t [repository-address]/[image-name] .
    docker image push [repository-address]/[image-name]
    ```
    
2. Preparing to start WAO-Scheduler
    * Create configmap
    ``` 
    kubectl create -f wao_configmap.yaml
    ```
    * Launch metrics-server
    ``` 
    kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
    ```
    * launch tensorflow serving
    ``` 
    kubectl create -f tensorflow-server-dep.yaml
    ```
    * Labeling
    Give each node the following label;
    * ambient/max: Maximum ambient temperature in celsius.
    * ambient/min: Minimum ambient temperature in celsius.
    * cpu1/max: Maximum CPU1 temperature in celsius.
    * cpu1/min: Minimum CPU1 temperature in celsius.
    * cpu2/max: Maximum CPU2 temperature in celsius.
    * cpu2/min: Minimum CPU2 temperature in celsius.
    * tensorflow/host: IP address of tensorflow serving.

3. launch MinimizePower scheduler
    ```
    kubectl create -f wao-scheduler-deployment.yaml
    ```
    Check if minimizePower was started normally. Success if the pod status changes to "Running".
    ``` 
    kubectl get pod -n kube-system -o wide | grep wao-scheduler
    ```
    If you want to use wao-scheduler, please refer test.yaml.
    ```
    kubectl apply -f test.yaml
    ```
## License
Apache License 2.0, see LICENSE.
