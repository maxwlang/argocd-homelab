# Setup

## Initial cluster setup
1. Create cluster
2. Install ArgoCD
    1. `cd install`
    2. `kubectl create namespace argocd`
    3. Apply secrets
        1. `kubectl apply -f ../secrets/repo-secret.yaml`
        2. `kubectl apply -f ../secrets/sops-age-secret.yaml`
    4. `helm install argo-cd argo-cd/ --namespace argocd --values values-override.yaml`
3. Expose ArgoCD to loadbalencer
    1. `kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'`
4. Login to ArgoCD
    1. `kubectl port-forward svc/argocd-server -n argocd 8080:443`
    2. `argocd admin initial-password -n argocd`
    3. [Login](https://localhost:8080) with admin:<password>
    4. Change password
5. Allow ArgoCD to manage itself
    1. `cd install`
    2. `helm upgrade argocd ./argo-cd --namespace=argocd --create-namespace -f values-override.yaml`

## Setup nvidia plugin
1. Install nvidia driver on agent 1
    1. `sudo apt install gcc build-essential -y && wget http://international.download.nvidia.com/XFree86/Linux-x86_64/545.29.06/NVIDIA-Linux-x86_64-545.29.06.run && sudo chmod +x NVIDIA-Linux-x86_64-545.29.06.run && sudo ./NVIDIA-Linux-x86_64-545.29.06.run`
    2. Let the installer disable nouveau, patch initramfs, then reboot
    3. `sudo ./NVIDIA-Linux-x86_64-545.29.06.run`
    4. reboot 
2. Apply nvidia driver patches
    1. `sudo mkdir /opt/nvidia && sudo chown -R $USER /opt/nvidia && cd /opt/nvidia && git clone https://github.com/keylase/nvidia-patch && cd nvidia-patch && sudo bash patch.sh && sudo bash patch-fbc.sh`
3. Install nvidia container toolkit
    1. Follow https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#installing-with-apt
    2. Or just run this:
        ```bash
        curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
        && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
        sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
        sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list && sudo apt update && sudo apt install -y nvidia-container-toolkit
        
4. Install nvidia plugin
    1. Follow https://github.com/NVIDIA/k8s-device-plugin#configure-containerd
5. Test cuda
    1. Pull image `sudo ctr image pull docker.io/nvidia/cuda:12.3.1-base-ubuntu20.04`
    2. Run image `sudo ctr run --rm --gpus 0 -t docker.io/nvidia/cuda:12.3.1-base-ubuntu20.04 nvidia-smi`
6. Reboot agent 1
7. Test cuda on kubernetes
    ```bash
    cat <<EOF | kubectl create -f -
    apiVersion: v1
    kind: Pod
    metadata:
        name: gpu-pod
    spec:
        restartPolicy: Never
        containers:
            - name: cuda-container
              image: nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda10.2
              command: [ "/bin/bash", "-c", "--" ]
              args: [ "while true; do sleep 30; done;" ]
              env:
              - name: NVIDIA_VISIBLE_DEVICES
                value: "all"
              - name: NVIDIA_DRIVER_CAPABILITIES
                value: "all"
              # Try to avoid using resource limits, so we can use the gpu on multiple containers
              #resources:
              #  limits:
              #      nvidia.com/gpu: 1 # requesting 1 GPU
        tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
    EOF

# Cluster Services & Plugins / Patches
| Service | Namespace | Description | URL |
| ------- | ------- | ----------- | --- |
| ArgoCD | argocd | GitOps | https://localhost:8080 (Proxy) |
| Ksops | N/A | Sops wrapper for kubernetes secrets | N/A |
| Plex (Published) | media | Media server | https://app.plex.tv |
| Plex (Local) | media | Media server | https://app.plex.tv |
| Longhorn | longhorn-system | Replicating local storage | N/A |
| local-storage-provisioner | local-path-storage | Local storage provisioner | N/A |
| [Accomplice-V2](https://github.com/maxwlang/accomplice-v2) | discord | Home grown Discord leaderboard bot | N/A |
| MetalLB | metallb-system | Loadbalencer | N/A |
| Nvidia Device Plugin | system | Nvidia GPU support | N/A |
| Traefik | traefik | Ingress controller | N/A |
