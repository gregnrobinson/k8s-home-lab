# Cluster Monitoring Setup
Shoutout to [carlosedp (Carlos Eduardo) · GitHub](https://github.com/carlosedp) for providing a quick and easy way to deploy Kubernetes monitoring components for ARM builds.

Clone the following repo to a machine that can execute `kubectl`  commands against your Kubernetes environment.
```
git clone https://github.com/carlosedp/cluster-monitoring.git
cd cluster-monitoring
```

## Update Ingress URL
I am using the first worker node address which is `192.168.0.82` in my environment. Any worker node address will suffice. Also using the nip.io domain for the ability to have many different urls for the same IP. 

```
make change_suffix suffix=192.168.0.82.nip.io
```

## Deploy
To customize the manifests, edit `vars.jsonnet` and rebuild the manifests.
```
make vendor
make
make deploy

# Or manually:

make vendor
make
kubectl apply -f manifests/setup/
kubectl apply -f manifests/
```

Once deployed.   The following urls will be exposed. Grafana is where all metrics dashboards will be stored and by default there are tons of dashboards available for use immediately.
```
http://grafana.192.168.0.82.nip.io/
http://prometheus.192.168.0.82.nip.io/
http://alertmanager.192.168.0.82.nip.io/
```

For more detailed documentation, check out Carlos’s repo: [GitHub - carlosedp/cluster-monitoring: Cluster monitoring stack for clusters based on Prometheus Operator](https://github.com/carlosedp/cluster-monitoring)