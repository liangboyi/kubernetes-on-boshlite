# Kubernetes on boshlite
## Prepare Environment
VirtualBox  https://www.virtualbox.org/wiki/Downloads
BOSH CLI  https://bosh.io/docs/cli-v2.html#install
CredHub CLI https://github.com/cloudfoundry-incubator/credhub-cli#installing-the-cli
Kubernetes CLI https://kubernetes.io/docs/tasks/tools/install-kubectl/


## Update bosh-lite (use credhub and uaa)
use credhub (-o bosh-deployment/credhub.yml)
use uaa (-o bosh-deployment/uaa.yml)
```
bosh create-env bosh-deployment/bosh.yml \
  -o bosh-deployment/virtualbox/cpi.yml \
  -o bosh-deployment/virtualbox/outbound-network.yml  \
  -o bosh-deployment/bosh-lite.yml \
  -o bosh-deployment/bosh-lite-runc.yml \
  -o bosh-deployment/uaa.yml \
  -o bosh-deployment/credhub.yml \
  -o bosh-deployment/jumpbox-user.yml \
  --vars-store vbox/bosh-lite-creds.yml \
  -v director_name=bosh-lite \
  -v internal_ip=192.168.50.6 \
  -v internal_gw=192.168.50.1 \
  -v internal_cidr=192.168.50.0/24 \
  -v outbound_network_name=NatNetwork \
  --state vbox/bosh-lite-state.json
```
## update cloud-config
```
bosh update-cloud-config <(curl -L https://github.com/cloudfoundry/cf-deployment/raw/master/iaas-support/bosh-lite/cloud-config.yml)
```

## Deploy kubo
### add kubo-deployment
```
mkdir kubo
cd kubo
git clone -b v0.12.0 https://github.com/cloudfoundry-incubator/kubo-deployment
git checkout -b v0.12.0
```
### Upload Stemcell
```
bosh -e vbox upload-stemcell ~/Downloads/bosh-stemcell-3468.20-warden-boshlite-ubuntu-trusty-go_agent.tgz
```
### replace stemcell
```
cd kubo-deployment
cat <<EOF > manifests/ops-files/use-specific-stemcell.yml
- type: replace
  path: /stemcells/0/version
  value: ((stemcell_version))
EOF
```

### Upload relation releases
```
wget --content-disposition https://github.com/pivotal-cf-experimental/kubo-etcd/releases/download/v7/kubo-etcd.7.tgz
bosh -e vbox upload-release kubo-etcd.7.tgz

wget --content-disposition https://bosh.io/d/github.com/cloudfoundry-incubator/kubo-release?v=0.12.0
bosh -e vbox upload-release kubo-release-0.12.0.tgz

wget --content-disposition https://github.com/cloudfoundry-community/docker-boshrelease/releases/download/v30.1.4/docker-30.1.4.tgz
bosh -e vbox upload-release docker-30.1.4.tgz

wget --content-disposition https://bosh.io/d/github.com/cloudfoundry/bosh-dns-release?v=0.0.11
bosh -e vbox upload-release bosh-dns-release-0.0.11.tgz
```
### replace kubo-release
```
cd kubo-deployment
cat <<EOF > manifests/ops-files/kubernetes-kubo-0.12.0.yml
- type: replace
  path: /releases/name=kubo
  value:
    name: kubo
    url: https://bosh.io/d/github.com/cloudfoundry-incubator/kubo-release?v=0.12.0
    version: 0.12.0
EOF
```

### deploy
```
bosh -e vbox deploy -d cfcr cfcr.yml \
            -o ops-files/kubernetes-kubo-0.12.0.yml \
            -o ops-files/use-specific-stemcell.yml \
            -v stemcell_version="'3468.20'" \
            --no-redact
```

### credhub
```
credhub login \
        -s 192.168.50.6:8844 \
        --client-name=credhub-admin \
        --client-secret=`bosh int vbox/bosh-lite-creds.yml --path /credhub_admin_client_secret` \
        --ca-cert <(bosh int vbox/bosh-lite-creds.yml --path /uaa_ssl/ca) \
        --ca-cert <(bosh int vbox/bosh-lite-creds.yml --path /credhub_ca/ca)
        
```

### credhub find
```
➜ credhub find
credentials:
- name: /dns_healthcheck_client_tls
  version_created_at: 2018-01-30T16:10:11Z
- name: /dns_healthcheck_server_tls
  version_created_at: 2018-01-30T16:10:11Z
- name: /dns_healthcheck_tls_ca
  version_created_at: 2018-01-30T16:10:11Z
- name: /bosh-lite/cfcr/tls-kubernetes-dashboard
  version_created_at: 2018-01-30T16:10:11Z
- name: /bosh-lite/cfcr/kubernetes-dashboard-ca
  version_created_at: 2018-01-30T16:10:11Z
- name: /bosh-lite/cfcr/tls-etcd-peer
  version_created_at: 2018-01-30T16:10:10Z
- name: /bosh-lite/cfcr/tls-etcd-client
  version_created_at: 2018-01-30T16:10:10Z
- name: /bosh-lite/cfcr/tls-etcd-server
  version_created_at: 2018-01-30T16:10:10Z
- name: /bosh-lite/cfcr/tls-docker
  version_created_at: 2018-01-30T16:10:10Z
- name: /bosh-lite/cfcr/tls-kubernetes
  version_created_at: 2018-01-30T16:10:10Z
- name: /bosh-lite/cfcr/tls-kubelet
  version_created_at: 2018-01-30T16:10:09Z
- name: /bosh-lite/cfcr/kubo_ca
  version_created_at: 2018-01-30T16:10:09Z
- name: /bosh-lite/cfcr/route-sync-password
  version_created_at: 2018-01-30T16:10:09Z
- name: /bosh-lite/cfcr/kube-scheduler-password
  version_created_at: 2018-01-30T16:10:09Z
- name: /bosh-lite/cfcr/kube-controller-manager-password
  version_created_at: 2018-01-30T16:10:09Z
- name: /bosh-lite/cfcr/kube-proxy-password
  version_created_at: 2018-01-30T16:10:09Z
- name: /bosh-lite/cfcr/kubelet-password
  version_created_at: 2018-01-30T16:10:09Z
- name: /bosh-lite/cfcr/kubo-admin-password
  version_created_at: 2018-01-30T16:10:08Z
```

### init kubo env
```
export admin_password=$(credhub get -n /bosh-lite/cfcr/kubo-admin-password | bosh int --path /value -)
export master_host=10.244.0.128
export cluster_name=cfcr
export user_name=cfcr-admin
export context_name=cfcr

kubectl config set-cluster "${cluster_name}" --server="https://${master_host}:8443"   --insecure-skip-tls-verify=true
kubectl config set-credentials "${user_name}" --token="${admin_password}"
kubectl config set-context "${context_name}" --cluster="${cluster_name}" --user="${user_name}"
kubectl config use-context "${context_name}"
```
### nodes
```
kubectl get node -o wide
```

### start proxy
localhost:8001/ui
```
kubectl proxy
```

## demo
```
cat <<EOF | kubectl apply -f - 
apiVersion: v1
kind: Service
metadata:
  name: hello-service
  namespace: default
spec:
  replicas: 2
  type: Nodeport
  ports:
    - name: http
      protocol: TCP
      port: 8080
  selector:
    app: abc
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: abc
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: abc
  template:
    metadata:
      labels:
        app: abc
    spec:
      containers:
      - name: demo
        image: cloudfoundry/lattice-app
        ports:
        - containerPort: 8080
EOF
```






