# Install K8S
Taken from [here](https://alexellisuk.medium.com/walk-through-install-kubernetes-to-your-raspberry-pi-in-15-minutes-84a8492dc95a)
- On Pi:
    ```bash
    sudo vim /boot/cmdline.txt
    ```
    And add:
    ```cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory```
- Give the minimum RAM to the UI:
    ```bash
    sudo vim /boot/config.txt
    ```
    And add ```gpu_mem = 16```
- Restart.
- On your laptop:
    - get the tools:
    ```bash
    curl -sSL https://get.arkade.dev | sudo sh
    arkade get kubectl
    arkade get k3sup
    ```
    - Install:
    ```bash
    export IP=192.168.1.161
    k3sup install --ip $IP --user grifonas
    ```

- Disable Traefik:
    ```bash
    helm -n kube-system delete traefik traefik-crd
    kubectl -n kube-system delete helmchart traefik traefik-crd
    touch /var/lib/rancher/k3s/server/manifests/traefik.yaml.skip
    systemctl restart k3s```
- NGINX Ingress:
    ```bash
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    helm install nginx ingress-nginx/ingress-nginx -n kube-system
    ```
- Cert manager. See updated docs [here](https://cert-manager.io/docs/installation/helm/), At the time of writing this is up to date:
    ```bash
    helm repo add jetstack https://charts.jetstack.io
    helm repo update
    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.0/cert-manager.crds.yaml
    helm install   cert-manager jetstack/cert-manager   --namespace cert-manager   --create-namespace   --version v1.10.0
    ```
- Cluster Issuer:
    - Create a IAM user as described [here](https://cert-manager.io/docs/configuration/acme/dns01/route53/).
    - Get Creds.
    - Secret:
    ```bash
    kubectl -n cert-manager create secret generic prod-route53-credentials-secret --from-literal=secret-access-key=[Your secret key]
    ```
    - Issuer:
    ```bash
    kubectl -n cert-manager apply cluster-issuer.yaml
    ```
