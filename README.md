# RKE2 Cluster with Rancher and Fleet
This is a guide on how to setup a Kubernetes cluster with RKE2 and Rancher.
It is specifically designed for the use in the Machine Learning Cluster of the Ruhr-University Bochum.

RKE2 is a lightweight Kubernetes distribution that is easy to setup and maintain. It is a fully conformant Kubernetes distribution.
It inherits the usability and ease-of-operations and deployment model of K3s, while still maintaining full Kubernetes API compatibility.
RKE2 supports both single-node and high-availability (HA) multi-node setups, making it flexible for different deployment needs.
The installation process is straightforward, with a single binary installation. Upgrades are also simplified, which is crucial for maintaining a secure environment.
It does not rely on Docker (like RKE1), but launches containers directly with containerd.
RKE2 includes Canal (Calico and Flannel) for networking and supports popular Kubernetes storage options, simplifying cluster setup.
It comes with an out of the box NGINX Ingress controller, saving time and effort in setting up ingress resources.
Built-in support for deploying applications and services using Helm charts is also included.

## About Kubernetes
Kubernetes is a container orchestration system. It is used to manage containerized applications in a cluster.
It is used to deploy, scale and manage containerized applications.
In our case we use it to deploy our training jobs to the cluster.
It can also be used to deploy a development pod, where you can develop and debug your code before running a training (With our without GPU).

Pods are the smallest deployable units in k8s. They are used to run containers.
Jobs are used to run containers for a finite time. They are used to run training jobs.
The recommended way is to use **jobs for training** and _pods for development_.

Other resources inside the cluster are:
- Namespaces: Used to separate different users and projects
- Nodes (Worker): Used to run pods
- Services: Used to expose pods to the outside world
- ConfigMaps: Used to store configuration data
- Secrets: Used to store sensitive data, e.g. passwords, SSH keys, etc.
- Persistent Volumes: Used to store data persistently
- Persistent Volume Claims: Used to claim a persistent volume for a pod
- Pods: Used to run containers (e.g. development pod)
- Jobs: Used to run containers for a finite time (e.g. training jobs)
- CronJobs: Used to run jobs periodically (e.g. backups)
- Deployments: Used to deploy pods (e.g. for development)

## Contents
1. [Setting up NVME NFS Server](./docs/nfs.md)
1. [Setting up RKE2 on Premise](./docs/cluster.md)
1. [Deploy services](./docs/fleet.md)
1. [Creating a new User](./docs/user.md)
1. [Create Downstream Cluster](./docs/downsteam_cluster.md)

# Using the Cluster

## Getting Started
0. Install kubectl on your local machine.
1. Download the kubeconfig file from Rancher for all clusters you want to access.
    - Save them as `<Cluster Name>.kubeconfig` in `~/.kube/`
2. Merge the kubeconfig files into one file:
    ```bash
    cd ~/.kube/
    export KUBECONFIG=$(ls  *.kubeconfig|tr '\n' ':')
    kubectl config view --merge --flatten > config
    ```
3. Check if the kubeconfig file is working:
    ```bash
    # Get the clusters
    kubectl config get-contexts
    # Set which cluster to use
    kubectl config use-context local
    # Check the nodes of the cluster
    kubectl get nodes
    ```
4. Set your user namespace to use:
    ```bash
    kubectl config set-context local --namespace=<username>
    ```
5. Run the GPU test job and check the logs. It will also mount the NFS share.
    ```bash
    kubectl apply -f .k8s/test_gpu.yaml
    ```