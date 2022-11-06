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
- Join another node:
    ```bash
    k3sup join --user grifonas --ip 192.168.1.4 --server-ip 192.168.1.161 --server-user grifonas
    ```

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
- NFS Share :
    ```bash
    sudo vim /etc/exports
    # Add: /media/grifonas/2TBHDD-2 *(rw,all_squash,insecure,async,no_subtree_check,anonuid=1000,anongid=1000,no_root_squash)
    sudo exportfs -ra
    sudo systemctl restart nfs-kernel-server
    sudo systemctl status nfs-server
    ```
- Monitoring:
    ```bash
    op signin --account my.1password.com
    export GRAFANA_PRIVATE_PASSWORD='op://Private/Grafana-Private/password'
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update

    op run -- helm upgrade --install --version 41.7.0 kube-prometheus-stack prometheus-community/kube-prometheus-stack \
     --namespace monitoring \
     --set grafana.persistence.enabled=true \
     --set grafana.ingress.enabled=true \
     --set grafana.ingress.hosts[0]=grafana.gkon.link \
     --set grafana.ingress.tls[0].hosts[0]=grafana.gkon.link \
     --set grafana.ingress.tls[0].secretName=grafana.gkon.link-auto-created \
     --set grafana.ingress.annotations."cert-manager\.io/cluster-issuer=letsencrypt-prod"  \
     --set grafana.ingress.annotations."kubernetes\.io/ingress\.class=nginx"  \
     --set grafana.plugins[0]=grafana-piechart-panel \
     --set defaultRules.rules.kubeControllerManager=false \
     --set defaultRules.rules.kubeScheduler=false \
     --set defaultRules.rules.kubeSchedulerAlerting=false \
     --set grafana.adminPassword=${GRAFANA_PRIVATE_PASSWORD}
    ```

- Nextcloud (not worth it though)
    ```bash
    kubectl apply -f nextcloud-data-pvand-pvc.yaml -n nextcloud
    ```
    ```bash
    #export NEXTCLOUD_PRIVATE_POSTGRES_PASSWORD=$(op item get Nextcloud-Private --fields=postgres-password)
    export NEXTCLOUD_PRIVATE_PASSWORD=$(op item get Nextcloud-Private --fields=password)

    helm repo add nextcloud https://nextcloud.github.io/helm/
    helm repo update
    helm upgrade --install -n nextcloud nextcloud nextcloud/nextcloud \
    --set nextcloud.username=admin \
    --set nextcloud.password="${NEXTCLOUD_PRIVATE_PASSWORD}" \
    --set persistence.enabled=true \
    --set ingress.enabled=true \
    --set ingress.className=nginx \
    --set ingress.annotations."cert-manager\.io/cluster-issuer=letsencrypt-prod" \
    --set ingress.tls[0].hosts[0]=nextcloud.gkon.link \
    --set ingress.tls[0].secretName=nextcloud.gkon.link-auto-created \
    --set nextcloud.host=nextcloud.gkon.link \
    --values nextcloud-values.yaml
    ```
