# How to setup a k0s cluster on an oracle VM

Oracle offers you a free Ampera based VM with up to four cores and 24GB of RAM backed by a 200GB boot drive.
Therefore I want to use this as a kubernetes machine. k0s is currently my distribution of choice since it's not as ugly as microk8s (snap data location is baaaaaad) and not as minimalistic as k3s. Though, I do not like mirantis either, but thats for the future ...

## Preparation (within the VM)

Since the default images provided by oracle are strange, I need to alter some iptables settings.

```sh
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT
sudo iptables -t nat -F
sudo iptables -t mangle -F
sudo iptables -F
sudo iptables -X
```

To test: `sudo apt pruge -y iptables; sudo apt -y autoremove; sudo reboot` might work as well, need to check.

## Operate (from your local working device)

Since I'm working with k0s I use `k0sctl` to do the livecycle of the cluster.

```sh
./k0sctl apply -c k0s_config.yaml
./k0sctl kubeconfig -c k0s_config.yaml
./k0sctl reset -c k0s_config.yaml
```

## Cleanup left over directories (within your VM)

If you have to reset something during testing, keep in mind to cleanup left over directories.

```sh
sudo rm -rf /etc/k0s
sudo rm -rf /var/lib/k0s
sudo rm -rf /var/lib/kubelet
```

## Cluster configuration (from your local working device)

To be able to install things right, I use metallb with only one public IP address with an nginx ingress controller that handles all the traffic for the services. We have to use `hostNetwork` for the ingress controller as it would otherwise not be possible to allocate port `80` or `443` for it.
To test if things really work, I use my dummy helmchart that deploys a simple nginx webserver so I can check if it is reachable from the internet via the given URL. (You need to create a URL upfront that points to the public IP of your VM)

```sh
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/main/config/manifests/metallb-native.yaml
kubectl apply -f metallb_config.yaml
# helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx --set hostNetwork="true"
helm install foo tibeer/default-helmchart --set ingress.host="test.tibeer.de" 
```
