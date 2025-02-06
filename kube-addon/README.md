## Kubectl cmpletion
```bash
# install bash-completion
apt-get install bash-completion

# Enable kubectl autocompletion
echo 'source <(kubectl completion bash)' >>~/.bashrc
kubectl completion bash >/etc/bash_completion.d/kubect

# If you have an alias for kubectl, you can extend shell completion to work with that alias:
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -F __start_kubectl k' >>~/.bashrc
```

## Install helm version 3
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

## Install local path storage class
```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.30/deploy/local-path-storage.yaml

# check storageClass
kubectl get sc

# Mark a StorageClass as default
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# check storageClass
kubectl get sc

# change volume path
kubectl apply -f local-path-storage/local-path-config.yml

# Restart local-path-storage deployment
kubectl -n local-path-storage rollout restart deployment local-path-provisioner
```

## Create sample pod with pvc for CSI test
```bash
cat <<'EOF' | kubectl apply -f -
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
EOF
```

## Install wavescope
```bash
kubectl apply -f https://github.com/weaveworks/scope/releases/download/v1.13.2/k8s-scope.yaml

# check all resource on wave namespace
kubectl get all -n weave
```

## Install and config ingress nginx
```bash
# add helm repository and update
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo list && helm repo update

# deploy ingress nginx
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx -f ingress-nginx/helm.values.yaml --create-namespace

# check resources on ingress nginx namespace
kubectl get all -n ingress-nginx
```

## Install and config cert manager
```bash
# add helm repository and update
helm repo add jetstack https://charts.jetstack.io
helm repo list && helm repo update

# deploy cert manager
helm install cert-manager jetstack/cert-manager --namespace cert-manager -f cert-manager/helm.values.yaml --create-namespace

# check resources on cert-manager namespace
kubectl get all -n cert-manager

# deploy cluster issuer
kubectl apply -f cert-manager/clusterIssuer.yaml

# check clusterIssuer
kubectl get clusterIssuer
```

## Check certmanager service
```bash
## Check ingress service
kubectl apply -f weavescope/ingress.yml
# check all resource on wave namespace
kubectl get ingress -n weave
```
watch on brower `scope.kube.<DOMAIN>`

## Install and config kube prometheus stack
```bash
# add helm repository and update
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo list && helm repo update

# deploy kube prometheus stack
helm install prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring -f kube-prometheus-stack/helm.values.yaml --create-namespace

# deploy secret config
kubectl apply -f kube-prometheus-stack/manifest.yaml

# check resources on monitoring namespace
kubectl get all -n monitoring

# check secret
kubectl get secret -n monitoring

# check ingress
kubectl get ingress -n monitoring

# create pod with all certificates
kubectl apply -f kube-prometheus-stack/get-etcd-cert.yaml

# create pod name variable
podname=$(kubectl get pods -o=jsonpath='{.items[0].metadata.name}' -n default | grep busybox)

# create etcd secret for monitoring
kubectl create secret generic etcd-client-cert -n monitoring \
--from-literal=etcd-ca="$(kubectl exec $podname -n default -- cat /etc/kubernetes/pki/etcd/ca.crt)" \
--from-literal=etcd-client="$(kubectl exec $podname -n default -- cat /etc/kubernetes/pki/apiserver-etcd-client.crt)" \
--from-literal=etcd-client-key="$(kubectl exec $podname -n default -- cat /etc/kubernetes/pki/apiserver-etcd-client.key)"

# delete pod
kubectl delete -f kube-prometheus-stack/get-etcd-cert.yaml
```

**Change kube-proxy config map for expose metrics url**

**check these url:**
  - https://prometheus.kube.<DOMAIN>/targets
  - https://grafana.kube.<DOMAIN>

## Install and config loki logging
```bash
# add helm repository and update
helm repo add grafana https://grafana.github.io/helm-charts
helm repo list && helm repo update

# deploy loki logging
helm install loki grafana/loki-stack --namespace loki-stack -f loki-stack/helm.values.yaml --create-namespace

# check resources on loki-stack namespace
kubectl get all -n loki-stack
```

## Install and config argo-cd
```bash
# add helm repository and update
helm repo add argo https://argoproj.github.io/argo-helm
helm repo list && helm repo update

# deploy argocd
helm install argo argo/argo-cd --namespace argocd -f argocd/helm.values.yaml --create-namespace

# check resources on argocd namespace
kubectl get all -n argocd

# change argocd ingress and check
kubectl apply -f argocd/ingress.yml
kubectl -n argocd get ingress

# get admin password for argocd ui
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## Install VPA on kube
```bash
# clone VPA repository
cd /tmp/ && git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler

# install with this scripts
./hack/vpa-up.sh

# check vpa pods
kubectl -n kube-system get pod | grep vpa
```

# Install permission manager
```bash
# Create the Namespace
kubectl create namespace permission-manager

# Create a secret and ingress
kubectl apply -f permission-manager/manifest.yml

# check ingress
kubectl -n permission-manager get ingress,secret

# Then apply
kubectl apply -f https://github.com/sighupio/permission-manager/releases/download/v1.8.0/crd.yml
kubectl apply -f https://github.com/sighupio/permission-manager/releases/download/v1.8.0/seed.yml
kubectl apply -f https://github.com/sighupio/permission-manager/releases/download/v1.8.0/deploy.yml
```

## install and config minio
```bash
# add helm repository and update
helm repo add minio https://charts.min.io/
helm repo list && helm repo update

# deploy minio
helm install minio minio/minio --namespace minio -f minio/helm.values.yaml --create-namespace

# check resources on minio namespace
kubectl get all -n minio

# check ingress and secret
kubectl get ingress,secret -n minio
```

## create bucket and auth for velero backup

```bash
# install mc on linux
curl https://dl.min.io/client/mc/release/linux-amd64/mc \
  --create-dirs \
  -o /usr/local/bin/mc
chmod +x /usr/local/bin/mc
mc --help

# create minio alias with this command
mc alias set mecan https://object.kube.<DOMAIN> dfseoijlwmemsdepoqwvm2 akPEKIvsdcsfek7BbHGd7V73XgbjgWSB5hvnKwxyZKJzQU

# create bucket with cli
mc mb mecan/velero-backup
mc ls mecan

# create user velero-backup with command
mc admin user add mecan \
    velero-backup  \
    vsdcsfek7BbHGd7V73XgbjgW

# check all user on minio
mc admin user ls mecan

# create velero-backup policy
mc admin policy create mecan \
   velero-backup minio/velero-backup-policy.json

# check all policy on minio
mc admin policy ls mecan

# attach policy velero to user velero
mc admin policy attach mecan velero-backup --user velero-backup

# check all policy on user velero-backup
mc admin policy entities mecan --user velero-backup
```

## install and config velero
```bash
# add helm repository and update
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts/
helm repo list && helm repo update

# deploy velero
helm install velero vmware-tanzu/velero --namespace velero -f velero/helm.values.yaml --create-namespace

# create BackupStorageLocation
kubectl apply -f velero/manifest.yaml

# check resources on velero namespace
kubectl get all -n velero
```

## velero backup test and check
```bash
# install velero client
wget https://github.com/vmware-tanzu/velero/releases/download/v1.15.0-rc.2/velero-v1.15.0-rc.2-linux-amd64.tar.gz
tar -xzvf velero-v1.15.0-rc.2-linux-amd64.tar.gz
mv velero-v1.15.0-rc.2-linux-amd64/velero /usr/local/bin/
velero --version

# create backup with velero commands
velero backup create ingress-backup --include-resources ingress

# check backup
velero backup logs ingress-backup
velero backup describe ingress-backup
```

## create etcd backup and move to minio
```bash

# create bucket with cli
mc mb mecan/etcd-backup
mc ls mecan

# create user velero-backup with command
mc admin user add mecan \
    etcd-backup  \
    vsdcsfweek7BbHGddsdew7V73XgbjgW

# check all user on minio
mc admin user ls mecan

# create velero-backup policy
mc admin policy create mecan \
   etcd-backup minio/etcd-backup-policy.json

# check all policy on minio
mc admin policy ls mecan

# attach policy velero to user velero
mc admin policy attach mecan etcd-backup --user etcd-backup

# check all policy on user velero-backup
mc admin policy entities mecan --user etcd-backup

# create etcd backup every 4th hour and move to minio
kubectl apply -f etcd-backup/manifest.yaml

# check cronjob
kubectl get cronjobs -n kube-system

# create job from cronjob
kubectl create job sample --from=cronjob/etcd-backup-cronjob
```