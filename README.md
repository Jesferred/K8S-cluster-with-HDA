# K8S cluster with HDA. Heterogenous dynamic architecture.
## Підхід до автомасштабування кластерів Kubernetes на основі персональних обчислювальних ресурсів користувачів.

**Environment: Kubernetes, Helm, Ansible, Docker, Linux Ubuntu 22.04, Prometheus + Grafana, Github Actions**

**Tecnologies for Kubernetes: Cluster API, Metal3-io, Cluster Autoscaler.**
____

## How to set up it?
Linux with Ubuntu 22.04:
```bash
sudo visudo
```
Insert:
```bash
username  ALL=(ALL) NOPASSWD: ALL
```
*Optional (If you are running on a newly installed Linux):*
```bash
sudo apt install openssh-server -y
sudo apt install git -y
sudo apt install make -y
```
____
## Start
```bash
git clone https://github.com/Jesferred/K8S-cluster-with-HDA.git
cd K8S-cluster-with-HDA/
export IMAGE_OS=ubuntu
export EPHEMERAL_CLUSTER=minikube
make
```
## If the infra is up without errors, we move on. I would recommend checking the variables
```bash
env | grep IMAGE
```
## Run each script, then *watch kubectl get machines -n metal3*, so that they have time to be provisioned.
```bash
./tests/scripts/provision/cluster.sh
./tests/scripts/provision/controlplane.sh
./tests/scripts/provision/worker.sh
./tests/scripts/provision/pivot.sh
```
## If we want to test something on the master node.
```bash
ssh metal3@192.168.111.249
```
## Everything from ephermal cluster minikube moved to workload cluster, kubeconfig is here
```bash
export KUBECONFIG=/tmp/kubeconfig-test1.yaml
```
## Tests
```bash
kubectl get bmh -n metal3
kubectl get nodes
kubectl get machines -A
kubectl get metal3machines -A
kubectl get pods -A
kubectl create deployment nginx-deploy --image=nginx
kubectl expose deployment nginx-deploy --type=NodePort --port=80
kubectl get po
kubectl get svc
curl http://192.168.111.101:30244
```

## Cluster Autoscaler
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm repo update
helm install cluster-autoscaler autoscaler/cluster-autoscaler   --namespace kube-system   --set cloudProvider=clusterapi   --set autoDiscovery.clusterName=test1   --set extraArgs.scale-down-delay-after-add=10m   --set extraArgs.scale-down-unneeded-time=10m   --set clusterAPIWorkloadKubeconfigPath=/tmp/kubeconfig-test1.yaml --set extraArgs.scale-down-utilization-threshold=0.2 --set extraArgs.scan-interval=1m --set extraArgs.max-node-provision-time=30m
kubectl --namespace=kube-system get pods -l "app.kubernetes.io/name=clusterapi-cluster-autoscaler,app.kubernetes.io/instance=cluster-autoscaler"
export KUBE_EDITOR=nano
```
## To make autoscaling work you need to add two lines to annotations machinedeployment
```bash
kubectl edit machinedeployments.cluster.x-k8s.io -n metal3

cluster.x-k8s.io/cluster-api-autoscaler-node-group-min-size: "0"
cluster.x-k8s.io/cluster-api-autoscaler-node-group-max-size: "1"
capacity.cluster-autoscaler.kubernetes.io/cpu: "2"
capacity.cluster-autoscaler.kubernetes.io/ephemeral-disk: 5Gi
capacity.cluster-autoscaler.kubernetes.io/gpu-count: "0"
capacity.cluster-autoscaler.kubernetes.io/memory: 4096M
```
## **If using values min 0, the last three lines must be written.**
## You need to give permissions to the cluster autoscaler.
```bash
kubectl apply -f ca-metal3-rbac.yaml
```
## Untaint controlplane
```bash
kubectl taint node controlplane-role.kubernetes.io/control-plane:NoSchedule-
```
## If the important pods are left on the worker's node, we do drain
```bash
kubectl drain test1-worker-node --delete-emptydir-data --ignore-daemonsets
```
## Test pods for Autoscaling
```bash
kubectl apply -f test-autoscale.txt
```
## Logs
```bash
kubectl logs -n kube-system cluster-autoscaler-clusterapi-cluster-autoscaler-cdcf8fbb89496b --tail 200 -f
```
## Monitoring
```bash
kubectl create namespace monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl taint no test1-worker-node key=value:NoSchedule
helm install prom prometheus-community/prometheus   --namespace monitoring   --set server.persistentVolume.enabled=false
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install graf grafana/grafana   --namespace monitoring
kubectl get secret --namespace monitoring graf-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
kubectl get po -n monitoring
kubectl port-forward --address 0.0.0.0 -n monitoring svc/grafana 3000:3000
kubectl taint no test1-worker-node key=value:NoSchedule-
```
