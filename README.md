# K3s HA kubernetes cluster

Installation instructions for a lightweight kubernetes cluster with high availability.  
Based on [Adrian Goins](https://www.youtube.com/channel/UCjjwExYSPRWwjj9WwydrVmA) instructions.  
Using:  

- K3s, Rancher
- Kube-VIP  
  As a controlplane LoadBalancer.
- MetalLb  
  For load balancing Kubernetes services.
- Traefik  
  Ingress controller.
- K3sup, Helm

---

## Table of Contents

- [K3s HA kubernetes cluster](#k3s-ha-kubernetes-cluster)
  - [Table of Contents](#table-of-contents)
  - [Prerequisites](#prerequisites)
    - [Cluster](#cluster)
    - [On your development machine](#on-your-development-machine)
    - [On each node in the cluster](#on-each-node-in-the-cluster)
  - [Setup K3s cluster and Kube-VIP (control plane load balancer)](#setup-k3s-cluster-and-kube-vip-control-plane-load-balancer)
  - [MetalLB - K3s Service LoadBalancer Installation](#metallb---k3s-service-loadbalancer-installation)
    - [Installation by manifest](#installation-by-manifest)
  - [Install Traefik via Helm chart](#install-traefik-via-helm-chart)
  - [Install Rancher](#install-rancher)
  - [Resources](#resources)
  
---

## Prerequisites

### Cluster

- An odd number of server nodes, in this example three server nodes.
- At least 1 worker / agent node, in this example four workers / agents.
- Enable public-key authentication for ssh on each node.
- Setup HostName for all nodes, for example
  
    ```sh
    # network 10.0.0.0/24
    # Server nodes
    10.0.0.10   k3s-srv01.local
    10.0.0.11   k3s-srv02.local
    10.0.0.12   k3s-srv03.local
    # Worker / Agents
    10.0.0.20   k3s-wrk01.local
    10.0.0.21   k3s-wrk02.local
    10.0.0.22   k3s-wrk03.local
    10.0.0.23   k3s-wrk04.local
    # kube-vip
    10.0.0.99   k3s-vip.local
    # reserve for MetalLb
    # 10.0.0.90-10.0.0.98
    10.0.0.90   k3s-traefik.local   # if you want to use Traefik-dashboard
    ```

### On your development machine

- Install k3sup (dev machine):  

  ```sh
  curl -sLS https://get.k3sup.dev | sh
  sudo install k3sup /usr/local/bin/
  k3sup --help
  ```  

- Install Helm  

  - From Apt (Debian/Ubuntu):  

    ```sh
    curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
    sudo apt-get install apt-transport-https --yes
    echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
    sudo apt-get update
    sudo apt-get install helm
    ```

  - Initialize a Helm Chart Repository, e.g. `stable`:

    ```sh
    helm repo add stable https://charts.helm.sh/stable
    ```

  - To fix warning `Kubernetes configuration file is group-readable. This is insecure. Location: /home/<user>/.kube/config`, run:
  
    ```sh
    chmod go-r ~/.kube/config
    ```

  > For other installation options, see [Helm install](https://helm.sh/docs/intro/install/).

### On each node in the cluster

- Disable sudo password on the nodes with  
  
  > Note  
  > At the time of writing, K3sup only works if no sudo password is set.  
  > However, with ssh and public-key authentication enabled, this shouldn't be much of a problem.  
  > You may enable password again, after k3s was installed.

  ```sh
  sudo visudo
  ```  

  At the bottom of the file, add the following line:  

  ```txt
  $USER ALL=(ALL) NOPASSWD: ALL
  ```  

  where $USER = username, e.g. `yourusername ALL=(ALL) NOPASSWD: ALL`

---

## Setup K3s cluster and Kube-VIP (control plane load balancer)

- Setup IP`s, Variables, ...  

  ```sh
  export K3S_VERSION=v1.20.4+k3s1 
  export K3S_NAME=k3s-demo 
  export IP_TLS_SAN=k3s-vip.local   #10.0.0.99
  export USERNAME=<yourusername>
  export SSHKEY=~/.ssh/id_rsa
  export SSHPORT=22
  ```

- Install via k3sup:  

  ```sh
  k3sup install \
    --host k3s-srv01.local \
    --user $USERNAME \
    --ssh-key $SSHKEY \
    --ssh-port $SSHPORT \
    --k3s-version=$K3S_VERSION \
    --local-path=config.$K3S_NAME.yaml \
    --context $K3S_NAME \
    --cluster \
    --tls-san $IP_TLS_SAN \
    --k3s-extra-args="--disable traefik --disable servicelb --node-taint  node-role.kubernetes.io/master=true:NoSchedule"
  ```  

- SSH into server, e.g. `ssh yourname@k3s-srv01.local -p $SSHPORT`  
- Switch user to `root` with `su root`  
- Get `kube-vip` manifest  
  
  ```sh
  curl -s https://kube-vip.io/manifests/rbac.yaml > /var/lib/rancher/k3s/server/manifests/kube-vip-rbac.yaml
  ```

- Check / edit `kube-vip-rbac.yaml`  

  ```yaml
    - apiGroups: ["coordination.k8s.io"]
      resources: ["leases"]
      verbs: ["list", "get", "watch", "update", "create"]
  ```  

- Create alias  
  
  ```sh
  alias kube-vip="docker run --network host --rm plndr/kube-vip:0.3.3"
  ```

  >**Note**  
  > We are using docker here, the documentation uses containerd.  However, the alias command in the documentation didn`t work.  
  > See also [kube-vip offical documentation](https://kube-vip.io/hybrid/daemonset/).

- Lookup interface name and export  
  
  ```sh
  ip a                      # --> e.g. ens18
  export INTERFACE=ens18
  export VIP=k3s-vip.local  # 10.0.0.99
  ```

- Generate manifest  

  ```sh
  kube-vip manifest daemonset \
      --arp \
      --interface $INTERFACE \
      --address $VIP \
      --controlplane \
      --leaderElection \
      --taint \
      --inCluster | tee /var/lib/rancher/k3s/server/manifests/kube-vip.yaml
  ```

- Check / edit `/var/lib/rancher/k3s/server/manifests/kube-vip.yaml`
  
  ```yaml
  tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
    operator: Exists
  ```

- ping vip `k3s-vip.local`
  
  ```sh
  ping k3s-vip.local

  PING k3s-vip.local (10.0.100.99) 56(84) bytes of data.
  64 bytes from k3s-vip.local (10.0.100.99): icmp_seq=1 ttl=64 time=0.024 ms
  64 bytes from k3s-vip.local (10.0.100.99): icmp_seq=2 ttl=64 time=0.030 ms
  64 bytes from k3s-vip.local (10.0.100.99): icmp_seq=3 ttl=64 time=0.059 ms
  ```

- Exit ssh, switch back to your development machine.
- Edit `$HOME/config.$K3S_NAME.yaml`, replace server-ip with vip-ip / vip-hostname:
  
  ```yaml
  apiVersion: v1
  clusters:
  - cluster:
      certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZ...
      server: https://k3s-vip.local:6443  #replace ip with vip-hostname / vip-ip
    name: k3s-demo
  #...
  ```  

- Add remaining server nodes  
  
  ```sh
  k3sup join \
    --host k3s-srv02.local \
    --user $USERNAME \
    --ssh-key $SSHKEY \
    --ssh-port $SSHPORT \
    --server-host=$IP_TLS_SAN \
    --k3s-version=$K3S_VERSION \
    --server \
    --k3s-extra-args="--disable traefik --disable servicelb --node-taint  node-role.kubernetes.io/master=true:NoSchedule"
  ```

  ```sh
  k3sup join \
    --host k3s-srv03.local \
    --user $USERNAME \
    --ssh-key $SSHKEY \
    --ssh-port $SSHPORT \
    --server-host=$IP_TLS_SAN \
    --k3s-version=$K3S_VERSION \
    --server \
    --k3s-extra-args="--disable traefik --disable servicelb --node-taint  node-role.kubernetes.io/master=true:NoSchedule"
  ```  

- Check cluster
  
  ```sh
  kubectl get nodes

  NAME        STATUS   ROLES                       AGE     VERSION
  k3s-srv01   Ready    control-plane,etcd,master   5m      v1.20.4+k3s1
  k3s-srv02   Ready    control-plane,etcd,master   30s     v1.20.4+k3s1
  k3s-srv03   Ready    control-plane,etcd,master   10s     v1.20.4+k3s1
  ```  

  ```sh
  kubectl get pods -n kube-system

  NAME                                      READY   STATUS    RESTARTS   AGE
  coredns-854c77959c-t7rqd                  1/1     Running   0          107m
  helm-install-traefik-7spjg                0/1     Pending   0          107m
  kube-vip-ds-jzhvf                         1/1     Running   0          56s
  kube-vip-ds-l45gf                         1/1     Running   0          28s
  kube-vip-ds-srrt5                         1/1     Running   0          18m
  local-path-provisioner-5ff76fc89d-x9ppj   1/1     Running   0          107m
  metrics-server-86cbb8457f-gvzfx           1/1     Running   0          107m
  ```  

- Add worker node(s)  

  ```sh
  k3sup join \
    --host=k3s-wrk01.local \
    --user $USERNAME \
    --ssh-key $SSHKEY \
    --ssh-port $SSHPORT \
    --server-host=$IP_TLS_SAN \
    --server-user $USERNAME \
    --k3s-version=$K3S_VERSION
  ```  

  ```sh
  k3sup join \
    --host=k3s-wrk02.local \
    --user $USERNAME \
    --ssh-key $SSHKEY \
    --ssh-port $SSHPORT \
    --server-host=$IP_TLS_SAN \
    --server-user $USERNAME \
    --k3s-version=$K3S_VERSION
  ```  

  ```sh
  k3sup join \
    --host=k3s-wrk03.local \
    --user $USERNAME \
    --ssh-key $SSHKEY \
    --ssh-port $SSHPORT \
    --server-host=$IP_TLS_SAN \
    --server-user $USERNAME \
    --k3s-version=$K3S_VERSION
  ```  

  ```sh
  k3sup join --host=k3s-wrk04.local \
    --user $USERNAME \
    --ssh-key $SSHKEY \
    --ssh-port $SSHPORT \
    --server-host=$IP_TLS_SAN \
    --server-user $USERNAME \
    --k3s-version=$K3S_VERSION
  ```  

---

## MetalLB - K3s Service LoadBalancer Installation

### Installation by manifest  

- Get default manifests and combine them in a single file `metallb.yaml`

  ```sh
  curl -s https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/namespace.yaml > /var/tmp/metallb.yaml; \
  echo '---' >> /var/tmp/metallb.yaml; \
  curl -s https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/metallb.yaml >> /var/tmp/metallb.yaml; \
  mv /var/tmp/metallb.yaml /var/lib/rancher/k3s/server/manifests
  ```

- Edit manifest `metallb.yaml`
  
  ```sh
  nano /var/lib/rancher/k3s/server/manifests/metallb.yaml
  ```

  ```yaml
    kubernetes.io/os: linux
  serviceAccountName: speaker
  terminationGracePeriodSeconds: 2
  tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
    operator: Exists                      #<------ add 
  ```

- Create secret

  ```sh
  kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
  ```

- create configmap `config.yaml` and assign a block of free addresses for MetalLb, here `10.0.0.90-10.0.0.98`

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    namespace: metallb-system
    name: config
  data:
    config: |
      address-pools:
      - name: default
        protocol: layer2
        addresses:
        - 10.0.0.90-10.0.0.98
  ```  

  and apply

  ```sh
  kubectl apply -f config.yaml
  ```

- Verify MetalLb is deployed

  ```sh
  kubectl get pods -n metallb-system -o wide
  NAME                          READY   STATUS    RESTARTS   AGE     IP            NODE        NOMINATED NODE   READINESS GATES
  controller-65db86ddc6-p8g74   1/1     Running   0          45s     10.42.6.10    k3s-wrk04   <none>           <none>
  speaker-fbr96                 1/1     Running   0          45s     10.0.0.12     k3s-srv03   <none>           <none>
  speaker-h5htt                 1/1     Running   0          45s     10.0.0.21     k3s-wrk02   <none>           <none>
  speaker-hqmpz                 1/1     Running   0          45s     10.0.0.23     k3s-wrk04   <none>           <none>
  speaker-ppbpn                 1/1     Running   0          45s     10.0.0.22     k3s-wrk03   <none>           <none>
  speaker-qmkjq                 1/1     Running   0          45s     10.0.0.11     k3s-srv02   <none>           <none>
  speaker-vdbqt                 1/1     Running   0          45s     10.0.0.10     k3s-srv01   <none>           <none>
  speaker-z2zbp                 1/1     Running   0          45s     10.0.0.20     k3s-wrk01   <none>           <none>
  ```

---

## Install Traefik via Helm chart

- Download example `values.yaml` file from GitHub

```sh
mkdir ~/traefik
wget https://raw.githubusercontent.com/traefik/traefik-helm-chart/master/traefik/values.yaml /traefik/values.yaml
```

- Edit `values.yaml` file  
  
  > **Note**  
  > We want to modify some values, to run traefik as DaemonSet instead of Deployment.  
  > Running as Daemonset, a instance of traefik will be spawn automatically on each worker node (failover).

  ```sh
  nano /traefik/values.yaml
  ```

  - change `kind: Deployment` to `kind: DaemonSet`
  
    ```yaml
    deployment:
    enabled: true
    # Can be either Deployment or DaemonSet
    #kind: Deployment
    kind: DaemonSet
    ```

  - Section `providers` set `publishedService enabled = true`  
  
    ```yaml
    providers:
    #. . .
      # IP used for Kubernetes Ingress endpoints
      publishedService:
        enabled: true
    ```

  - Check section `service` is `enabled: true` and `type: Loadbalancer`  

    ```yaml
    # Options for the main traefik service, where the entrypoints traffic comes
    # from.
    service:
      enabled: true
      type: LoadBalancer
    ```

  - Check section `rbac` is `enabled: true`  

    ```yaml
    # Whether Role Based Access Control objects like roles and rolebindings should be created
    rbac:
      enabled: true
    ```

  Save modification and exit editor.  

- Create namespace `traefik-system`  

  ```sh
  kubectl create namespace traefik-system
  ```

- Install traefik with helm

  ```sh
  helm repo add traefik https://helm.traefik.io/traefik
  helm repo update
  helm install traefik --values traefik/values.yaml --namespace traefik-system traefik/traefik
  ```

- Check installation succeeded
  
  ```sh
  kubectl get all -n traefik-system -o wide

  NAME                READY   STATUS    RESTARTS   AGE     IP           NODE        NOMINATED NODE   READINESS GATES
  pod/traefik-hqc8c   1/1     Running   0          30s     10.42.5.9    k3s-wrk03   <none>           <none>
  pod/traefik-m7kn7   1/1     Running   1          30s     10.42.3.17   k3s-wrk01   <none>           <none>
  pod/traefik-pltvd   1/1     Running   0          30s     10.42.6.11   k3s-wrk04   <none>           <none>
  pod/traefik-wfj5g   1/1     Running   0          30s     10.42.4.11   k3s-wrk02   <none>           <none>

  NAME              TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)                      AGE     SELECTOR
  service/traefik   LoadBalancer   10.43.82.54   10.0.0.90      80:31794/TCP,443:31030/TCP   30s     app.kubernetes.io/instance=traefik,app.kubernetes.io/name=traefik

  NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE     CONTAINERS   IMAGES          SELECTOR
  daemonset.apps/traefik   4         4         4       4            4           <none>          30s     traefik      traefik:2.4.7   app.kubernetes.io/instance=traefik,app.kubernetes.io/name=traefik
  ```

  > **Note**  
  > MetalLb will assign a external ip to the Traefik service, here `10.0.0.90` which is the first address of the block we reserved for MetalLb.

- Optional Install Traefik Dashboard
  
  - Create `traefik/dashboard.yaml` file and paste/edit
  
    ```yaml
    # dashboard.yaml
    apiVersion: traefik.containo.us/v1alpha1
    kind: IngressRoute
    metadata:
      name: dashboard
    spec:
      entryPoints:
        - web
      routes:
       #- match: Host(`traefik.localhost`) && (PathPrefix(`/dashboard`) || PathPrefix(`/api`))
        - match: Host(`k3s-traefik.local`) || (PathPrefix(`/dashboard`)
          kind: Rule
          services:
            - name: api@internal
              kind: TraefikService
    ```

  Save modification and exit editor.

  - Apply with kubectl
  
    ```sh
    kubectl apply -f traefik/dashboard.yaml
    ```  
  
    Open the dashboard in your browser http://k3s-traefik.local/dashboard/#/.  

---

## Install Rancher

Install Rancher on the HA cluster as described by Adrian Goins https://www.youtube.com/watch?v=9PLw1xalcYA&t=880s

---

## Resources

> **K3s, Rancher**
> - [Homepage](https://rancher.com/docs/k3s/latest/en)
> - [Installation / Uninstallation](https://rancher.com/docs/k3s/latest/en/installation/)

> **Adrian Goins on YouTube  / GitLab**
> - [HA K3s with etcd, kube-vip, MetalLB, and Rancher!](https://youtu.be/9PLw1xalcYA)
> - [Kubernetes 101: Why You Need To Use MetalLB](https://youtu.be/Ytc24Y0YrXE)
> - [GitLab](https://gitlab.com/monachus/channel)  

> **Helm**
> - [Homepage](https://helm.sh)
> - [Installation from apt debian/ubuntu](https://helm.sh/docs/intro/install/#from-apt-debianubuntu)
> - [Quickstart](https://helm.sh/docs/intro/quickstart/)

> **K3sup**
> - [GitHub - k3sup](https://github.com/alexellis/k3sup)

> **Kube-VIP**
> - [Homepage](https://kube-vip.io/)
> - [Using kube-vip in hybrid mode (load balancer)](https://kube-vip.io/hybrid/)

> **MetalLB**
> - [Homepage](https://metallb.universe.tf)
> - [Installation](https://metallb.universe.tf/installation/#installation-by-manifest)

>**Traefik**
>- [Homepage](https://traefik.io/)
>- [Install with Helm](https://traefik.io/blog/install-and-configure-traefik-with-helm/)
>- [Traefik Helm Chart on GitHub](https://github.com/traefik/traefik-helm-chart/tree/master/traefik)
