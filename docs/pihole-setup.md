# Pihole Setup

## Install Helm
[Helm | Installing Helm](https://helm.sh/docs/intro/install/)
```shell
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

## Install MetalLB
[MetalLB, bare metal load-balancer for Kubernetes](https://metallb.universe.tf/)
In order for the pihole services to expose themselves to the internal network, a load balancer needs to used as the service type. On bare metal installs of Kubernetes the load balancer functionality doesn’t exist out of the box so it is required to make use of one that will work. This is where MetalLB comes in to play.

There are several methods available for the install. I used the following method. Run the following two commands to setup MetalLB.
```shell
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/metallb.yaml
```

To get MetalLB to begin assigning external IPs to LoadBalancer service types, copy the following into a yaml file and modify the network range to match your local network. Ensure the addresses are not apart of your routers DHCP lease. I configured my router to only use up .80 for DHCP leases.
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
      - 192.168.1.200-192.168.0.250
```

Apply the config map using the following command.
```shell
kubectl apply -f <YOUR_CONFIG_YAML>
```

## Deploy Pihole
### Create load balancer config
For the load balancer IP address, I used the first available address in the MetalLB address pool. Modify the IP to match your MetalLB address pool.
```yaml
ingress:
  enabled: true
serviceTCP:
  loadBalancerIP: '192.168.0.200'
  type: LoadBalancer
serviceUDP:
  loadBalancerIP: '192.168.0.200'
  type: LoadBalancer
```

### Deploy the Pihole helm chart
Create namespace called Pihole and deploy the Pihole helm chart to the Pihole namespace.
```shell
kubectl create ns pihole

helm repo add mojo2600 https://mojo2600.github.io/pihole-kubernetes/
helm repo update
helm install pihole mojo2600/pihole --namespace pihole --values <YOUR_PIHOLE_CONFIG>
```

## Optionally, create in ingress route using nip.io domain name.
Nip.io provides the ability to use a domain name for resolving the web interface locally. It makes it easier to remember the address.
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: pihole-ingress
  namespace: pihole
spec:
  rules:
  - host: pihole.192.168.0.200.nip.io
    http:
      paths:
      - backend:
          serviceName: pihole-web
          servicePort: http
        path: /
  tls:
  - hosts:
    - pihole.192.168.0.200.nip.io
```

You can then navigate to Pihole in a browser on your local network using either the load balancer IP or the nip.io ingress IP.

![1054C5DD-65E4-406C-9C8E-7DE6F5C78454](https://user-images.githubusercontent.com/26353407/122658271-322bbb80-d139-11eb-9495-7c241e5abe92.png)

## Uninstall
```shell
helm uninstall pihole
```
