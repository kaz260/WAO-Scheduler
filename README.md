# MinimizePower プラグイン

---

本書は MinimizePower プラグインを含む kube-scheduler のビルドおよびデプロイ手順を示す

---

## 前提条件

* [Go言語 v1.15.1](https://golang.org/)
* [Kubernetes v1.19.7](https://github.com/kubernetes/kubernetes/releases/tag/v1.19.7)
* [ipmi_exporter](https://github.com/soundcloud/ipmi_exporter)  
  各ノードで動いていること
* スケジューラ―の push 先となる docker repository  
* tensorflow serving の docker image  
  TODO: image は提供対象なのか、各自用意なのかがわからない

## MinimizePower プラグインを含む kube-scheduler をビルド

1. Kubernetes のソースコードをダウンロード

    ※ 最新バージョンではディレクトリ構成が違うため、[Kubernetes v1.19.7](https://github.com/kubernetes/kubernetes/releases/tag/v1.19.7) を使用します
    ```
    curl -L -o kubernetes.tar.gz https://github.com/kubernetes/kubernetes/archive/v1.19.7.tar.gz
    tar xvzf kubernetes.tar.gz
    ```

2. MinimizePower プラグインのソースコードを追加

    このプロジェクトのソースコードをダウンロードする
    ```
    TODO: githubリポジトリ等が決まらないとダウンロード先は不明
    ```
    解凍した後 `kubernetes/pkg/scheduler/framework/plugins` にコピーする  
    同名のファイルは上書きする

3. 必要なパッケージを追加

    prom2json パッケージを追加する
    ```
    go get github.com/prometheus/prom2json
    ```
    tensorflow パッケージを追加する
    ```
    TODO: vendor ディレクトリにある tensorflow および tensorflow_serving が必要だが追加方法が不明
    ```

4. スケジューラをビルド

    ビルド結果は `kubernetes/cmd/kube-scheduler/` に出力される
    ```
    cd ./kubernetes/cmd/kube-scheduler/
    CGO_ENABLED=0 go build -mod=mod scheduler.go
    ```

## kubernetesへのデプロイ

デプロイに必要なファイルが複数あるため、適当なディレクトリを作成しその中で作業することを推奨します

1. スケジューラの Docker イメージを作成

    以下内容で `Dockerfile` を作成する
    ``` Dockerfile
    FROM busybox
    ADD ./kube-scheduler /usr/local/bin/kube-scheduler
    ```
    上記手順でビルドした `kube-scheduler` を `Dockerfile` と同じディレクトリにコピーする  
    イメージを作成し、リポジトリに push する
    ```
    docker build -t [repository-address]/[image-name] .
    docker image push [repository-address]/[image-name]
    ```

2. minimize-scheduler の起動準備

    初回起動では以下の準備を行う必要があります
    * metrics-server を起動する([Kubernetes Metrics Server](https://github.com/kubernetes-sigs/metrics-server))
        ```
        kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
        ```
    * Configmap を追加する

        以下内容で `oc_configmap.yaml` を作成する
        ``` yaml
        kind: ConfigMap
        apiVersion: v1
        metadata:
        name: oc-scheduler-config
        namespace: kube-system
        data:
        policy.cfg: |
            {
            "kind" : "Policy",
            "apiVersion" : "v1",
            "metadata" : {
                "name": "oc-scheduler-config",
                "namespace": "kube-system"
            },
            "predicates" : [
                {"name" : "PodFitsResources"},
                {"name" : "PodToleratesNodeTaints"},
                {"name" : "MatchInterPodAffinity"}
            ],
            "priorities":[
                {"name" : "NodePreferAvoidPodsPriority", "weight" : 10000}
            ]
            }
        plugin.yaml: |
            kind: KubeSchedulerConfiguration
            apiVersion: kubescheduler.config.k8s.io/v1beta1
            leaderElection:
            leaderElect: false
            profiles:
            - plugins:
                score:
                enabled:
                - name: oc-score-plugin
                    weight: 100
            schedulerName: oc-scheduler
            percentageOfNodesToScore: 100
        ```
        以下コマンドで ConfigMap を追加する
        ```
        kubectl create -f oc_configmap.yaml
        ```
    * tensorflow の起動

        以下内容で `tensorflow-server-dep.yaml` を作成する  
        tensorflow のイメージと使用するモデルを指定してください
        ``` yaml
        apiVersion: apps/v1
        kind: Deployment
        metadata:
        name: [name]
        spec:
        replicas: 12
        selector:
            matchLabels:
            app: [name]
        template:
            metadata:
            labels:
                app: [name]
            spec:
            containers:
            - command:
                - /usr/bin/tf_serving_entrypoint.sh
                - --model_config_file=[model]
                name: [name]
                image: [tensorflow_image]
                ports:
                - containerPort: 8500
        ---
        apiVersion: v1
        kind: Service
        metadata:
        name: tensorflow-service
        spec:
        ports:
        - port: 8500
            targetPort: 8500
        selector:
            app: [name]
        type: ClusterIP
        ```
        tensorflow を起動する
        ```
        kubectl create -f tensorflow-server-dep.yaml
        ```
    * ラベルの付与

        各ノードに以下のラベルを付与します
        * ambient/max : 周辺温度の最大値(℃)
        * ambient/min : 周辺温度の最小値(℃)
        * cpu1/max : CPU1温度の最大値(℃)
        * cpu1/min : CPU1温度の最小値(℃)
        * cpu2/max : CPU2温度の最大値(℃)
        * cpu2/min : CPU2温度の最小値(℃)
        * tensorflow/host: tensorflow serving の IP
        * tensorflow/port: tensorflow serving のポート
        * tensorflow/name: tensorflow serving のモデル名
        * tensorflow/version: tensorflow serving のモデルバージョン(任意)
        * tensorflow/signature: tensorflow serving の signature(任意)

3. minimize-scheduler の起動準備

    以下内容で yaml ファイル(oc-scheduler-deployment.yaml)を作成する  
    上記手順で push したスケジューラのイメージを指定してください
    ``` yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
    name: oc-scheduler
    namespace: kube-system
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
    name: oc-scheduler-as-kube-scheduler
    subjects:
    - kind: ServiceAccount
    name: oc-scheduler
    namespace: kube-system
    roleRef:
    kind: ClusterRole
    name: system:kube-scheduler
    apiGroup: rbac.authorization.k8s.io
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
    name: oc-scheduler-volume-role-binding
    roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: system:volume-scheduler
    subjects:
    - kind: ServiceAccount
    name: oc-scheduler
    namespace: kube-system
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
    name: oc-scheduler-config-role-binding
    namespace: kube-system
    roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: extension-apiserver-authentication-reader
    subjects:
    - kind: ServiceAccount
    name: oc-scheduler
    namespace: kube-system
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
    name: metrics-reader-role
    rules:
    - apiGroups: ["metrics.k8s.io"]
    resources: [pods, nodes]
    verbs: [get, list, watch]
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
    name: oc-scheduler-metrics-role-binding
    roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: metrics-reader-role
    subjects:
    - kind: ServiceAccount
    name: oc-scheduler
    namespace: kube-system
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
    name: csinode-role
    rules:
    - apiGroups: ["storage.k8s.io"]
    resources: ["csinodes"]
    verbs: ["watch", "list", "get"]
    - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["watch", "list", "get"]
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
    name: oc-scheduler-csinode-role-binding
    roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: csinode-role
    subjects:
    - kind: ServiceAccount
    name: oc-scheduler
    namespace: kube-system
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    labels:
        component: scheduler
        tier: control-plane
    name: oc-scheduler
    namespace: kube-system
    spec:
    selector:
    matchLabels:
        component: scheduler
        tier: control-plane
    replicas: 1
    template:
    metadata:
        labels:
        component: scheduler
        tier: control-plane
        version: second
    spec:
        serviceAccountName: oc-scheduler
        containers:
        - command:
        - /usr/local/bin/kube-scheduler
        - --address=0.0.0.0
        - --leader-elect=false
        - --scheduler-name=oc-scheduler
        - --config=/etc/config/plugin.yaml
        - -v=5
        image: [repository-address]/[image-name]
        livenessProbe:
            httpGet:
            path: /healthz
            port: 10251
            initialDelaySeconds: 15
        name: kube-second-scheduler
        readinessProbe:
            httpGet:
            path: /healthz
            port: 10251
        resources:
            requests:
            cpu: '2'
        securityContext:
            privileged: false
        volumeMounts:
        - name: oc-scheduler-config
            mountPath: /etc/config
        tolerations:
        - key: node-role.kubernetes.io/master
        effect: NoSchedule
        - key: app
        value: batch
        effect: NoSchedule
        nodeSelector:
        node-role.kubernetes.io/master: ""
        hostNetwork: false
        hostPID: false
        volumes:
        - name: oc-scheduler-config
        configMap:
            name: oc-scheduler-config
    ```
    以下コマンドでスケジューラを起動する
    ```
    kubectl create -f oc-scheduler-deployment.yaml
    ```
    以下コマンドで起動を確認できれば成功  
    (ポッドのステータスが [Running] になれば成功)
    ```
    kubectl get pod -n kube-system -o wide | grep oc-scheduler
    ```
    MinimizePower プラグインが動いているスケジューラでポッドを起動する例  
    以下内容で使用する yaml ファイル(sche-ob-file.yaml)を作成します
    ``` yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
        name: test-minimize-sch
    spec:
        replicas: 1
        selector:
            matchLabels:
                application: test-minimize-sch
        template:
            metadata:
                labels:
                    application: test-minimize-sch
            spec:
                containers:
                    - name: test-minimize-scheduler
                    image: httpd
                    resources:
                        requests:
                            cpu: "1"
                        limits:
                            cpu: "0.5"
                schedulerName: oc-scheduler
    ```
    以下コマンドでポッドを起動し、スケジューラのログから使用されていることを確認出来ます
    ```
    kubectl create -f sche-ob-file.yaml
    ```
