# Setup - K3s - LXC

## Initial cluster setup
1. Create cluster
    1. Create 3 virtual machines
    2. Install k3s on main vm `curl -sfL https://get.k3s.io | sh -`
    3. Get k3s token `sudo cat /var/lib/rancher/k3s/server/node-token` from main vm
    4. Install k3s on agent vms with token `curl -sfL https://get.k3s.io | K3S_URL=https://<main node>:6443 K3S_TOKEN=<token> sh -`
2. Install ArgoCD
    1. `cd install`
    2. `kubectl create namespace argocd`
    3. Apply secrets
        1. `kubectl apply -f ../secrets/repo-secret.yaml`
        2. `kubectl apply -f ../secrets/sops-age-secret.yaml`
    4. Install helm
    5. Run as root:
        1. `export KUBECONFIG=/etc/rancher/k3s/k3s.yaml`
        2. `helm install argo-cd argo-cd/ --namespace argocd --values values-override.yaml`
3. Login to ArgoCD
    1. `kubectl port-forward svc/argocd-server -n argocd 8080:443`
    2. Get password from `argocd-initial-admin-secret` secret
    3. [Login](https://localhost:8080) with admin:<password>
    4. Change password
    5. Delete `argocd-initial-admin-secret` secret