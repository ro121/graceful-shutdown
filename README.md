# graceful-shutdown
# kubernetes graceful shutdown of custer

#!/bin/bash

# List of all nodes in the cluster
NODES=$(kubectl get nodes -o jsonpath='{.items[*].metadata.name}')

# Drain all nodes
echo "Draining all nodes..."
for NODE in $NODES; do
    echo "Draining node: $NODE"
    kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data --force
done

# Scale down all deployments
echo "Scaling down all deployments..."
DEPLOYMENTS=$(kubectl get deployments --all-namespaces -o jsonpath='{.items[*].metadata.name}')
for DEPLOYMENT in $DEPLOYMENTS; do
    echo "Scaling down deployment: $DEPLOYMENT"
    kubectl scale deployment $DEPLOYMENT --replicas=0 --all-namespaces
done

# Scale down all statefulsets
echo "Scaling down all statefulsets..."
STATEFULSETS=$(kubectl get statefulsets --all-namespaces -o jsonpath='{.items[*].metadata.name}')
for STATEFULSET in $STATEFULSETS; do
    echo "Scaling down statefulset: $STATEFULSET"
    kubectl scale statefulset $STATEFULSET --replicas=0 --all-namespaces
done

# Stop control plane components (only if managing control plane manually)
echo "Stopping control plane components..."
sudo systemctl stop kube-apiserver kube-controller-manager kube-scheduler

# Shutdown all nodes
echo "Shutting down all nodes..."
for NODE in $NODES; do
    echo "Shutting down node: $NODE"
    ssh $NODE "sudo shutdown -h now"
done

# Backup and delete PersistentVolumeClaims (PVCs) if required
echo "Backing up and deleting PVCs..."
PVCs=$(kubectl get pvc --all-namespaces -o jsonpath='{.items[*].metadata.name}')
for PVC in $PVCs; do
    echo "Backing up and deleting PVC: $PVC"
    # Add your backup logic here
    kubectl delete pvc $PVC --all-namespaces
done

echo "Kubernetes cluster shutdown completed."

