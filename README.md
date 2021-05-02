Lo Yuen Shan(SID:20433485)

# COMP4651 Assignment-6 (6 points)

Deadline: 11:59 pm, May 2 (Sunday)

---

## DockerCoins

- DockerCoins is made of 5 services
  - `rng` = web service generating random bytes
  - `hasher` = web service computing hash of POSTed data
  - `worker` = background process calling `rng` and `hasher`
  - `webui` = web interface to watch progress
  - `redis` = data store (holds a counter updated by `worker`)

![](https://raw.githubusercontent.com/hkust-comp4651-21S/assets/main/assignment6/dockercoins.png)

In this assignment, you will deploy an application called `DockerCoins` which generates a few random bytes, hashes these bytes, increments a counter (to keep track of speed) and repeats forever! You will try to find its bottlenecks when scaling and use HPA (Horizontal Pod Autoscaler) to scale the application. Please follow these instructions carefully!

## Environment Setup

The assignment needs to be setup in `AWS`. You should choose `Ubuntu Server 18.04 LTS (HVM), SSD Volume Type` as AMI (Amazon Machine Image) and `m4.xlarge` as the instance type. And you need to configure security group as followed, to make sure that you can access the service of minikube on the web browser later.

| Type        | Protocol | Port Range | Source    | Description |
| ----------- | -------- | ---------- | --------- | ----------- |
| All traffic | All      | All        | 0.0.0.0/0 |             |

Run the following commands to satisfy the requirements.

```bash
# Install Minikube
$ curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
$ sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Install kubectl
$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
$ sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install Docker
$ sudo apt-get update && sudo apt-get install docker.io -y

# Install conntrack
$ sudo apt-get install -y conntrack

# Install httping
$ sudo apt-get install httping

# Install Helm
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

## Running Minikube on EC2 Ubuntu

Become a root user.

```bash
$ sudo -i
```

> If you are not comfortable running commands as root, you must always add `sudo` before the commands `minikube` and `kubectl`.

### Start Minikube

```bash
$ minikube start --kubernetes-version=v1.18.0 HTTP_PROXY=https://minikube.sigs.k8s.io/docs/reference/networking/proxy/ --extra-config=apiserver.service-node-port-range=6000-32767 disk=20000MB --vm=true --driver=none
```

## Start the application

You can `git clone` the assignment repo and `cd` into this directory.

* Start the application from the `dockercoins` yaml

```bash
$ kubectl apply -f dockercoins.yaml
```

* Wait until all the components are running

```bash
$ kubectl get po
# NAME                     READY   STATUS    RESTARTS   AGE
# hasher-89f59d444-r7v7j   1/1     Running   0          27s
# redis-85d47694f4-m8vnt   1/1     Running   0          27s
# rng-778c878fb8-ch7c2     1/1     Running   0          27s
# webui-7dfbbf5985-24z79   1/1     Running   0          27s
# worker-949969887-rkn48   1/1     Running   0          27s
```

* Check the results from UI

```bash
$ minikube service webui
# |-----------|-------|-------------|----------------------------|
# | NAMESPACE | NAME  | TARGET PORT |            URL             |
# |-----------|-------|-------------|----------------------------|
# | default   | webui |          80 | http://172.31.67.128:30163 |
# |-----------|-------|-------------|----------------------------|
# 🎉  Opening service default/webui in default browser...
# 👉  http://172.31.67.128:30163
```

You can access the DockerCoin Miner WebUI on a web browser. The address is &lt;Public IPv4 address&gt;:&lt;port&gt;. (e.g. `3.238.254.199`:`30163`, where `3.238.254.199` is Public IPv4 address of the instance)

![](https://raw.githubusercontent.com/hkust-comp4651-21S/assets/main/assignment6/webui.png)

## Bottleneck detection

### Workers

Scale the # of workers from 2 to 10 (change the number of `replicas`).

```bash
$ kubectl scale deployment worker --replicas=3
```

| # of workers  | 1    | 2    | 3    | 4    | 5    | 10   |
| ------------- | ---- | ---- | ---- | ---- | ---- | ---- |
| hashes/second |  4   |   8  | 8-12 | 9-12 |  10  |  10  |


> Remarks: when the # of workers = 3,4, the # hashes/second is flutuated a lot within a range. So a range is provided above when workers = 3,4

> Question:  What is the speedup bump when you have 10x workers?
>
> Answer: &lt;You answer goes here&gt;.
> The max speed was 13.9, speedup bump is 10hashes/s , which is 2.5 times compared to when # of worker = 1. However, the max number of hashes/s happen when worker = 3,4. Increasing # of workers does not necessary increase the # of hases/s after reaching the peak performance.

### Rng / Hasher

Keep the number of workers as `10`. Note that, `rng` or `hasher` service is a bottleneck.

To identify which one is the bottleneck, you can use `httping` command for latency detection.

```bash
# Expose the service, since we are detecting the latency outside the k8s cluster
$ kubectl expose service rng --type=NodePort --target-port=80 --name=rng-np
$ kubectl expose service hasher --type=NodePort --target-port=80 --name=hasher-np

# Get the url of the service
$ kubectl get svc rng-np hasher-np

# Find the minikube address
$ kubectl cluster-info
# Kubernetes control plane is running at https://172.31.67.128:8443
# KubeDNS is running at https://172.31.67.128:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
# Here, minikube address is 172.31.67.128

# Detect the latency of hasher
$ httping <minikube-address>:<hasher-np-port>

# Detect the latency of rng
$ httping <minikube-address>:<rng-np-port>
```

| Service      | Hasher | Rng  |
| ------------ | ------ | ---- |
| Latency (ms) |    1ms |725.8ms|

> Question: Which service is the bottleneck? 
> 
> Answer: &lt;You answer goes here&gt;.
> Rng is the bottleneck as the latency is signlifcantly bigger than haser.

## HPA

To solve the bottleneck of the application, you can specify a horizontal pod autoscaler which can scale the number of Pods based on some metrics.

```bash
# Install prometheus
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ helm install prometheus prometheus-community/prometheus

# Expose service port
$ kubectl expose service prometheus-server --type=NodePort --target-port=9090 --name=prometheus-server-np
$ minikube service prometheus-server-np
```

Set a HPA service and export the latency for further hpa measurement. (Please replace `<FILL IN>` in the `httplat` yaml as which you need).

```bash
$ kubectl apply -f httplat.yaml
$ kubectl expose deployment httplat --port=9080

# Check if the deployment is ready
$ kubectl get deploy httplat
# NAME      READY   UP-TO-DATE   AVAILABLE   AGE
# httplat   1/1     1            1           43s
```

Configure Prometheus to gather the detected latency.

```bash
$ kubectl annotate service httplat \
          prometheus.io/scrape=true \
          prometheus.io/port=9080 \
          prometheus.io/path=/metrics
```

Connect to Prometheus

```bash
$ kubectl get svc prometheus-server-np
# NAME                   TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# prometheus-server-np   NodePort   10.96.122.34   <none>        80:24057/TCP   87s
```

Check if `httplab` metrics are available. You can try to graph the following PromQL expression, by accessing the address &lt;Public IPv4 address&gt;:&lt;port&gt;. (e.g. `24057` is the port of prometheus server)

```sql
rate(httplat_latency_seconds_sum[2m])/rate(httplat_latency_seconds_count[2m])
```

![](https://raw.githubusercontent.com/hkust-comp4651-21S/assets/main/assignment6/prometheus.png)

Create the autoscaling policy. (Please replace `<FILL IN>` in the `hpa-httplat` yaml as which you need)

```bash
$ kubectl apply -f hpa-httplat.yaml
```

After a little while we should see messages like this:  '`<unknown>/100m`', because the Kubernetes API server doesn't implement `custom.metrics.k8s.io`. We need to start an API service implementing this API group and register it with our API server.

The [Prometheus adapter](https://github.com/DirectXMan12/k8s-prometheus-adapter) is an open source project. It's a Kubernetes API service implementing API group `custom.metrics.k8s.io `. It maps the requests it receives to Prometheus metrics.

```bash
# Install the Prometheus adapter:
$ helm upgrade prometheus-adapter prometheus-community/prometheus-adapter \
       --install \
       --set prometheus.url=http://prometheus-server.default.svc \
       --set prometheus.port=80

# Check if the Prometheus adapter is ready:
$ kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
```

Configure the Prometheus adapter by editing the adapter's configmap.

```bash
$ kubectl edit configmap prometheus-adapter
```

Add the following new rule in the rules section of `prometheus-adapter`. Save and quit.

```yaml
- seriesQuery: |
        httplat_latency_seconds_sum{kubernetes_namespace!="",kubernetes_name!=""}
      resources:
        overrides:
          kubernetes_namespace:
            resource: namespace
          kubernetes_name:
            resource: service
      name:
        matches: "httplat_latency_seconds_sum"
        as: "httplat_latency_seconds"
      metricsQuery: |
        rate(httplat_latency_seconds_sum{<<.LabelMatchers>>}[5m])
        /rate(httplat_latency_seconds_count{<<.LabelMatchers>>}[5m])
```

Restart the `prometheus-adapter`.

```bash
$ kubectl rollout restart deployment prometheus-adapter
```

Now you can see the hpa running correctly.

```bash
$ kubectl get hpa
```

> Open question: Apart from the autoscaling policy we provided, think about how to reduce the fluctuation of the HPA (e.g. using two HPAs simultaneously to scale the pods)? Explain your method in details and show how you setup your own HPA.
>
> Answer: &lt;Your answer goes here&gt;.

## Grading

In this assignment, you are required to finish the following things.

- Report the performance of DockerCoins with different number of workers.
- Detect latency and find the bottleneck component (rng or hasher).
- Upload the screenshot which shows the performance graph after setting up the hpa.
- Open question: how to reduce the fluctuation of the HPA?
