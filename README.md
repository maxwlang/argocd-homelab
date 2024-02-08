# Setup - K3s - LXC

## Initial cluster setup
1. Create cluster
    1. Install nvidia gpu driver on proxmox host
        1. Disable nouveau
            1. `echo "blacklist nouveau\noptions nouveau modeset=0" | sudo tee -a /etc/modprobe.d/disable-nouveau.conf`
            2. `sudo update-initramfs -u`
            3. `sudo reboot`
        2. Install nvidia driver
            1. `sudo apt install gcc build-essential -y && wget http://international.download.nvidia.com/XFree86/Linux-x86_64/545.29.06/NVIDIA-Linux-x86_64-545.29.06.run && sudo chmod +x NVIDIA-Linux-x86_64-545.29.06.run && sudo ./NVIDIA-Linux-x86_64-545.29.06.run`
        3. Configure kernel modules
            1. `echo -e '\n# load nvidia modules\nnvidia-drm\nnvidia-uvm' | sudo tee -a /etc/modules-load.d/modules.conf`
            2. Add the following to /etc/udev/rules.d/70-nvidia.rules
                ```bash
                KERNEL=="nvidia", RUN+="/bin/bash -c '/usr/bin/nvidia-smi -L && /bin/chmod 666 /dev/nvidia*'"
                KERNEL=="nvidia_uvm", RUN+="/bin/bash -c '/usr/bin/nvidia-modprobe -c0 -u && /bin/chmod 0666 /dev/nvidia-uvm*'"
                SUBSYSTEM=="module", ACTION=="add", DEVPATH=="/module/nvidia", RUN+="/usr/bin/nvidia-modprobe -m"
        4. Configure nvidia persistenced
            ```bash
            cp /usr/share/doc/NVIDIA_GLX-1.0/samples/nvidia-persistenced-init.tar.bz2 . && bunzip2 nvidia-persistenced-init.tar.bz2 && tar -xf nvidia-persistenced-init.tar && rm /etc/systemd/system/nvidia-persistenced.service && chmod +x nvidia-persistenced-init/install.sh && sudo ./nvidia-persistenced-init/install.sh && rm -rf nvidia-persistenced-init*
        5. Patch nvidia driver for unlimited encoding
            ```bash
            sudo mkdir /opt/nvidia && sudo chown -R $USER /opt/nvidia && cd /opt/nvidia && git clone https://github.com/keylase/nvidia-patch && cd nvidia-patch && sudo bash patch.sh && sudo bash patch-fbc.sh
        6. enable overlay and br_netfilter
            1. `sudo modprobe overlay`
            2. `sudo modprobe br_netfilter`
    2. Create 3 LXC containers with `bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/ct/ubuntu.sh)"`
    3. Configure LXC for gpu passthrough
        1. Ensure nesting is enabled
        2. nano each lxc configuration file in `/etc/pve/lxc/*.conf` to have the following:
            ```
            swap: 0
            lxc.mount.entry = /dev/kmsg dev/kmsg none defaults,bind,create=file
            lxc.apparmor.profile: unconfined
            lxc.mount.auto: proc:rw sys:rw
            lxc.cgroup2.devices.allow: a
            lxc.cap.drop:
            lxc.cgroup2.devices.allow: c 188:* rwm
            lxc.cgroup2.devices.allow: c 189:* rwm
            lxc.cgroup2.devices.allow: c 195:* rwm
            lxc.cgroup2.devices.allow: c 237:* rwm
            lxc.cgroup2.devices.allow: c 240:* rwm
            lxc.cgroup2.devices.allow: c 226:* rwm
            lxc.cgroup2.devices.allow: c 29:0 rwm

            lxc.mount.entry: /dev/serial/by-id  dev/serial/by-id  none bind,optional,create=dir
            lxc.mount.entry: /dev/ttyUSB0       dev/ttyUSB0       none bind,optional,create=file
            lxc.mount.entry: /dev/ttyUSB1       dev/ttyUSB1       none bind,optional,create=file
            lxc.mount.entry: /dev/ttyACM0       dev/ttyACM0       none bind,optional,create=file
            lxc.mount.entry: /dev/ttyACM1       dev/ttyACM1       none bind,optional,create=file
            lxc.mount.entry: /dev/fb0 dev/fb0 none bind,optional,create=file
            lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
            lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
            lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file

            lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
            lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
            lxc.mount.entry: /dev/nvidia-modeset dev/nvidia-modeset none bind,optional,create=file
            lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
            lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file
            lxc.mount.entry: /dev/nvidia-caps/nvidia-cap1 dev/nvidia-caps/nvidia-cap1 none bind,optional,create=file
            lxc.mount.entry: /dev/nvidia-caps/nvidia-cap2 dev/nvidia-caps/nvidia-cap2 none bind,optional,create=file
    4. Start containers and install the same nvidia driver without kernel modules:
        ```bash
        sudo apt install gcc build-essential -y && wget http://international.download.nvidia.com/XFree86/Linux-x86_64/545.29.06/NVIDIA-Linux-x86_64-545.29.06.run && sudo chmod +x NVIDIA-Linux-x86_64-545.29.06.run && sudo ./NVIDIA-Linux-x86_64-545.29.06.run --no-kernel-module
    5. Patch nvidia driver for unlimited encoding in each container
            ```bash
            sudo mkdir /opt/nvidia && sudo chown -R $USER /opt/nvidia && cd /opt/nvidia && git clone https://github.com/keylase/nvidia-patch && cd nvidia-patch && sudo bash patch.sh && sudo bash patch-fbc.sh
    6. Install nvidia container toolkit in each container
        ```bash
        curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
        && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
        sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
        sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list && sudo apt update && sudo apt install -y nvidia-container-toolkit
    7. Install open-iscsi on each container `sudo apt install open-iscsi`
    8. Install k3s on main container `curl -sfL https://get.k3s.io | sh -`
    9. Get k3s token `sudo cat /var/lib/rancher/k3s/server/node-token` from main container
    10. Install k3s on agent containers with token `curl -sfL https://get.k3s.io | K3S_URL=https://<main node>:6443 K3S_TOKEN=<token> sh -`
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
5. Test cuda on kubernetes
    ```bash
    cat <<EOF | kubectl create -f -
    apiVersion: v1
    kind: Pod
    metadata:
        name: gpu-pod
    spec:
        restartPolicy: Never
        runtimeClassName: nvidia
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