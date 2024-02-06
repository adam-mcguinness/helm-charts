# Homelab

## Versioning

- Kubernetes - k3s v1.27.9+k3s1
- Rancher - V2.8


## Structure

Proxmox Hpyervisor -> Linux LXC Containers -> K3S -> Nginx Ingress Controller -> Rancher

## Prerequesits
- Linux Host or WSL
- kubectl
- helm
- vim

## Proxmox Hpyervisor

This is installed on Bare Metal, just download the .iso [here](https://www.proxmox.com/en/proxmox-virtual-environment/get-started) and make a bootable usb stick with RUFUS. Download rufus [here](https://rufus.ie/en/)

## Linux Containers

These can be set up in proxmox. Choose your linux flavor.
1. Uncheck Privlaged container
2. no not strat on create
3. ssh into proxmox
4. add the following to `/etc/pve/lxc/<conatinerId>.conf`
    ```
    lxc.apparmor.profile: unconfined
    lxc.cgroup.devices.allow: a
    lxc.cap.drop:
    lxc.mount.auto: "proc:rw sys:rw"
    ```
5. start the coantiners
6. run `pct push <container id> /boot/config-$(uname -r) /boot/config-$(uname -r)` for each conatinerId

### In Conatiner Config
This will need to be done in each container

you can get a shell in the conatiner with `lxc-attach <conatinerId>`

1. create this file `nano /usr/local/bin/conf-kmsg.sh`
2. Add the following
```
#!/bin/sh -e
if [ ! -e /dev/kmsg ]; then
        ln -s /dev/console /dev/kmsg
fi
mount --make-rshared /
```
3. Create this file `nano /etc/systemd/system/conf-kmsg.service`
4. add the following
```
[Unit]
Description=Make sure /dev/kmsg exists
[Service]
Type=simple
RemainAfterExit=yes
ExecStart=/usr/local/bin/conf-kmsg.sh
TimeoutStartSec=0
[Install]
WantedBy=default.target
```
5. run the following commands to start the service
```
chmod +x /usr/local/bin/conf-kmsg.sh
systemctl daemon-reload
systemctl enable --now conf-kmsg
```

Install the following packages:
- curl

## K3S Kubernetes Cluster

Followed most of the guide from [here](https://betterprogramming.pub/rancher-k3s-kubernetes-on-proxmox-containers-2228100e2d13)

### Control Node

Install k3s

`curl -fsL https://get.k3s.io | INSTALL_K3S_VERSION=v1.27.9+k3s1 sh -s - --disable traefik --node-name control.k8s`

Get kube config and save it locally under .kube/config (for linux) make sure to change the server to the ip address you gave your control.k8s

`cat /etc/rancher/k3s/k3s.yaml`

Get the Control Token (you will need this in the next step to setup the worker nodes)

`cat /var/lib/rancher/k3s/server/node-token`

save the config as a template
1. shut down the machaine.
2. right click on it an click clone
3. right click and click template
4. save the token and kube config in the notes

### Worker Nodes
Before you run the following command, shutdown the machine, clone, create a template and add the command below to the notes with the right tocket so that you can spin up more worker nodes easliy.
Install k3s

`curl -fsL https://get.k3s.io | INSTALL_K3S_VERSION=v1.27.9+k3s1 K3S_URL=https://<Ip_Of_Control_Node>:6443 K3S_TOKEN=<Token_From_Control_Node> sh -s - --node-name worker-1.k8s`

Do this for any additional nodes you need, just make sure to change the name of the nodes at the end of the command.

## Nginx Ingress Controller

`helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx`

`helm repo update`

`kubectl create namespace ingress`

`helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress --set controller.publishService.enabled=true`

## Cloudflare Certs

In order to sign you website there is a few setup things you need to do with Cloudflare and your K8s cluster.

1. Have a domain Registered and useing DNS on Cloudlfare.
2. Set an A record pointing to your home IP address, make sure to set is as proxied
3. Go to SSL/TLS on the left pane,
4. In SSL/TLS->Overview slect "Full (strict)"
5. In SSL/TLS->Origin Server you will need to generate an origin Certificate to load as a k8s secret. (default options are fine, but change if you want more sectuirty)
6. Use PEM format and copy the Origin Certificate to a tls.pem file.
7. Copy the Private key to tls.key file.
8. Download the Cloudflare Root CA Cert to the same directory as the above 2 files. `wget https://developers.cloudflare.com/ssl/static/origin_ca_rsa_root.pem`
9. Concatenate the tls.pem and origin_ca_rsa_root.pem `cat tls.pem origin_ca_rsa_root.pem > combined.crt`
10. Create the secret in the appropriate namespace `kubectl create secret tls my-tls-secret --cert=combined.crt --key=tls.key -n <namespace>` You will need a secret in each namespace for the ingress to pick up.

## Rancher

Add rancher latest repo

`helm repo add rancher-latest https://releases.rancher.com/server-charts/latest`

Create Namespace

`kubectl create namespace cattle-system`

Install Rancher using Certs from file option.

`helm install rancher rancher-latest/rancher --namespace cattle-system --set hostname=rancher.my.org --set bootstrapPassword=admin --set ingress.tls.source=secret`

Add Cloudflare Cert for your subdomain rancher.<your_domain>.com

`kubectl -n cattle-system create secret tls tls-rancher-ingress --cert=combined.crt --key=tls.key`

You will also need to add an ingress class to rancher or it will not work properly. Thanks to wrkode on github for this [fix](https://github.com/rancher/rancher/issues/37193#issuecomment-1114014095)

`kubectl edit ing -n cattle-system rancher`

add the following in annotations
```yaml
metedata:
    annotations:
        #exsiting annotations
        kubernetes.io/ingress.class: nginx
        #additional annotations
```

# Storage
## NFS Server

## Add PV to the Cluster
This is what will hold all of you media and config for the pods.

Add something like this to your k8s cluster:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
   name: nfs-pv
spec:
   capacity:
      storage: 100Gi
   accessModes:
      - ReadWriteMany
   nfs:
      path: /data
      server: nfs-server-ip
```

you will need to fill in the appropriate nfs server ip and path for your environment