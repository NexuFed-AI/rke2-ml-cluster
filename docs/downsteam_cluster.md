# Create downstream cluster
This tutorial shows how to create a RKE2 cluster and register it with a Rancher server.

## Create local cluster
```bash	
kubectl apply -f clusters/cluster-hpc.yaml
```

## Register a node
Go to
- https://ml-cluster.domaine.de/dashboard/c/_/manager/provisioning.cattle.io.cluster
- Click on the new cluster
- Click on the `Registration` button
- Check the `Insecrue` checkbox
- Select if it should be: etcd, controlplane or worker (at least one of each must be deployed)
    - etcd: Backend storage for the cluster (master node)
    - controlplane: API server (master node)
    - worker: Worker node (worker node)
- Copy the command
- Run the command on the node

```bash
sudo systemctl status rancher-system-agent.service
sudo journalctl -u rancher-system-agent.service -f
# sudo systemctl restart rancher-system-agent.service
```

## Add correct cluster name for user rights
1. In the Rancher UI go to the new cluster
2. Copy the name of the cluster from the URL
    - e.g. `c-m-kcmfgsh7`
3. Replace or add the cluster name in all users `cluster_binding.yaml`

## Add fleet
```bash
kubectl get clusters.fleet.cattle.io -A
kubectl apply -f fleet-repo.yaml
kubectl get clusters.fleet.cattle.io -A
```

## Uninstall from node
If you want to remove the cluster and everything related to it, you can run the following commands:
```bash
kubectl delete -f clusters/cluster-hpc.yaml

# On each node
sudo rke2-killall.sh
sudo rke2-uninstall.sh
```