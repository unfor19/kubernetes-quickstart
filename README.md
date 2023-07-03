# kubernetes-quickstart

## Topics

- Basic understanding of Kubernetes
- Creating a local development environment for hands-on experience with Kubernetes
- Deploying an application to Kubernetes for the first time ever
- Playing with Kubernetes Pod autoscaling

## Requirements

- [Docker Desktop](https://docs.docker.com/get-docker/)
- [minikube](https://minikube.sigs.k8s.io/docs/start/) - For local development with Kubernetes (cross-platform)
- [LENS](https://k8slens.dev/) - The Kubernetes IDE
- [Helm](https://helm.sh/docs/intro/install/) - Kubernetes packages (charts) manager
- [ab](https://httpd.apache.org/docs/2.4/programs/ab.html) - Apache HTTP server benchmarking tool
  - Windows - `choco install apache-httpd`
  - macOS - natively installed from BigSur 11+

## Getting Started

1. Clone this repo
1. Create a Kubernetes cluster locally
   ```bash
   minikube start --kubernetes-version=1.24.0
   ```
   **Note**: For beginners, it's best to start from version `1.24.0` as version [1.25.0](https://kubernetes.io/blog/2022/08/04/upcoming-changes-in-kubernetes-1-25/) includes breaking changes such as the removal of [PSP](https://kubernetes.io/blog/2022/08/04/upcoming-changes-in-kubernetes-1-25/#podsecuritypolicy-removal) which breaks a lot of Helm charts.
1. Deploy [nginx ingress controller](https://github.com/kubernetes/ingress-nginx/tree/main/charts/ingress-nginx) to enable traffic from "the internet" to the cluster

   ```bash
   helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx && \
   helm repo update && \
   helm upgrade --namespace kube-system --install nginx ingress-nginx/ingress-nginx
   ```

1. To access the NGINX Ingress Controller from the Host machine (macOS/Windows), we need to map its domain name to `127.0.0.1`, which will listen to ports 80 and 443.

   1. Edit the `hosts` file
      - **macOS**: Edit `/etc/hosts` with your favotire editor
      ```bash
      sudo vim /etc/hosts
      ```
      - **Windows**: Edit `C:\Windows\System32\drivers\etc\hosts` with Notepad or [Notepad++](https://notepad-plus-plus.org/downloads/v7.9.5/) as Administrator
   2. Add the following line

   ```bash
   127.0.0.1 quickstart.kubemaster.me
   ```

1. Verify with `LENS` that the nginx-ingress-controller has an External IP `127.0.0.1` - otherwise you won't be able to receive traffic from the host machine (your PC)
1. Deploy the "quickstart" application - [docker-cats](https://github.com/unfor19/docker-cats)
   ```bash
   kubectl apply -f app.yml
   ```
1. Start [minikube tunnel](https://minikube.sigs.k8s.io/docs/commands/tunnel/) to allow traffic from the host machine to the Kubernetes cluster ingresses
   ```bash
   # Execute in a separated terminal - keep it running
   sudo minikube tunnel
   ```
1. Test the deployment - Navigate to http://quickstart.kubemaster.me
1. (Optional) For further learning about HTTPS and authentication, see [kubernetes-localdev](https://github.com/unfor19/kubernetes-localdev)

### Recap

1. [nginx-ingress-controller](https://docs.nginx.com/nginx-ingress-controller/) - deployed with [Helm](https://helm.sh/)
2. [docker-cats](https://github.com/unfor19/docker-cats) - deployed with [kubectl](https://kubernetes.io/docs/reference/kubectl/)

   1. [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) - A Deployment provides declarative updates for [Pods](https://kubernetes.io/docs/concepts/workloads/pods/) and [ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/).
   2. [Service](https://kubernetes.io/docs/concepts/services-networking/service/) - network abstraction to route traffic to the application and the ability to access the application internally in the cluster with the [Service DNS record](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#a-aaaa-records) `quickstart.default.svc.local`
   3. [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) - An API object that manages external access to the services in a cluster, typically HTTP. Ingress may provide load balancing, SSL termination and name-based virtual hosting.

### Autoscaling

1. Deploy [Kubernetes Metrics Server](https://github.com/kubernetes-sigs/metrics-server) - need to collect metrics such as CPU and Memory of pods to autoscale

   ```bash
   helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/ && \
   helm upgrade --namespace kube-system --install metrics-server metrics-server/metrics-server --values metrics-server-values.yml
   ```

2. Apply a [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/) object
   ```bash
   kubectl apply -f hpa.yml
   ```
3. Test autoscaling with `ab`

   ```bash
   ab -k -v1 -n 100000 -c 100 -s 60 http://quickstart.kubemaster.me/
   ```

   - `-k` - KeepAlive feature, performs multiple requests within one HTTP session. Default is no KeepAlive. If you skip this flag, you'll get `Total of 16314 requests completed` on macOS, see this [Stackoverflow answer](https://stackoverflow.com/a/30357879/5285732)
   - `-v1` - Verbosity level
   - `-n` - Number of requests
   - `-c` - Number of parallel requests
   - `-s` - Timeout in seconds, the default is 30 seconds

4. View scaling up and down in LENS
