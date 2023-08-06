Title: Installing longhorn on microk8s nodes running Rocky Linux/Alma/RHEL
Date: 2023-08-06 16:42
Category: Guides

## Introduction
In one of my previous blog posts, i discussed how to setup a bare metal kubernetes cluster using microk8s. However, one crucial part was still missing: Storage. 

In the past i have used [longhorn]() for this, as it provides a really powerful way of handling persistent storage in bare metal environments. 

I recently switched all of my kubernetes nodes to Rocky Linux, hoping to get better performance and requiring less maintenance than ubuntu server. This however poses some challenges, as longhorn relies on iSCSI for storage, which works straight out of the box in ubuntu, but thats not the case for RHEL. After some tinkering around, i finally got this to work though and i am documenting the process here.

## Preparation

First, we need to install the `iscsi-initiator-utils`:
```bash
yum install iscsi-initiator-utils
```
After this we need to configure the initiator name for `iscsi`:
```bash
echo "InitiatorName=$(/sbin/iscsi-iname)" > /etc/iscsi/initiatorname.iscsi
```
Finally, we can enable and start the `iscsi service`:
```bash
sudo modprobe iscsi_tcp
systemctl enable iscsid
systemctl start iscsid
```
Now, we're ready to install longhorn via `helm`.
## Installation
In theory, the installation is pretty straight forward and you can follow the official documentation, except for one crucial part: If you installed microk8s via snap, the kubelet dir will be different, so we have to manually tell longhorn the correct path.

Add the helm repo:
```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
```

And install longhorn:
```bash
 helm install longhorn longhorn/longhorn --namespace longhorn-system --set csi.kubeletRootDir="/var/snap/microk8s/common/var/lib/kubelet" --create-namespace --version 1.5.1
```
You can see here that we changed the kubelet dir to `/var/snap/microk8s/common/var/lib/kubelet`, which is very important.


## Configuration

In theory, you should be good to go know. However, in some situations, some further configuration is necessary, especially in regards to accessing (and securing!) the rancher UI. As this process varies a lot from deployment to deployment, i wont go into too much detail here.

One more important note, if you have less than three nodes in your cluster: By default, longhorn sets the replica count for each volume to 3, which will cause all volumes to be degraded, as longhorn cant find enough nodes for the replicas. To change this, edit the configmap called `longhorn-storageclass`:
```bash
kubectl edit configmaps longhorn-storageclass
```
Just set `numberOfReplicas` to the amount of nodes you have in the yaml section. Updating this configmap will also update the storageclass, which will cause all future volumes to use this new number of replicas. 
For existing volumes, you can change the amount of replicas directly from the ui. 

Note, that if you decrease the number, longhorn wont automatically delete the excess replicas, which might cause the volume to stay degraded, even if you decreases the replica amount, so you might have to delete them manually, which you can also do from the UI.