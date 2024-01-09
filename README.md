# Homelab

## Versioning

Kubernetes - k3s v1.27.9+k3s1
Rancher - V2.8


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

Install the following packages:
- curl


## K3S Kubernetes Cluster

Followed most of the guide from [here](https://betterprogramming.pub/rancher-k3s-kubernetes-on-proxmox-containers-2228100e2d13)

### Control Node

Install k3s

`curl -fsL https://get.k3s.io | INSTALL_K3S_VERSION=v1.27.9+k3s1 sh -s - --disable traefik --node-name control.k8s`

Get kube config and save it locally under .kube/config (for linux)

`cat /etc/rancher/k3s/k3s.yaml`

Get the Control Token (you will need this in the next step to setup the worker nodes)

`cat /var/lib/rancher/k3s/server/node-token`

### Worker Nodes

Install k3s

`curl -fsL https://get.k3s.io | INSTALL_K3S_VERSION=v1.27.9+k3s1 K3S_URL=https://<Ip_Of_Control_Node>:6443 K3S_TOKEN=<Token_From_Control_Node> sh -s - --node-name worker-1.k8`

Do this for any additional nodes you need, just make sure to change the name of the nodes at the end of the command.

## Nginx Ingress Controller

`helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx`

`helm repo update`

`helm install nginx-ingress ingress-nginx/ingress-nginx --set controller.publishService.enabled=true`

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
10. Create the secret in the appropriate namespace `kubectl create secret tls my-tls-secret --cert=combined.pem --key=tls.key -n <namespace>` You will need a secret in each namespace for the ingress to pick up.

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