# Provisioning CoreOS, etcd, fleet, and flannel on Google Compute Engine

* Provision a control node on GCE
* Configure client SSL tunnels for remote access to etcd and fleet
* Provision 5 Kubernetes nodes on GCE

## Workspace

```
cd intro-to-kubernetes-workshop/cloud-configs
```

## Control Node

The control node will be used to bootstrap the k8s cluster. It will also serve as the host for the client applications.

The control node will run the following services:

  * etcd
  * fleet
  * flannel-route-manager

```
gcloud compute instances create kcontrol \
--image-project coreos-cloud \
--image coreos-alpha-472-0-0-v20141017 \
--boot-disk-size 200GB \
--machine-type g1-small \
--can-ip-forward \
--scopes compute-rw \
--metadata-from-file user-data=kcontrol.yaml \
--zone us-central1-a
```

## Setup Client SSH Tunnels

```
gcloud compute instances list
```

### etcd

```
ssh -i ~/.ssh/google_compute_engine -f -nNT -L 4001:127.0.0.1:4001 core@${KCONTROL_EXTERNAL_IP}
```

### fleet

```
export FLEETCTL_TUNNEL="${KCONTROL_EXTERNAL_IP}"
```

## Cluster Configuration

Configure the cluster using the etcdctl command.

### flannel configuration

The flannel subnet manager requires the following settings in etcd:

```
etcdctl --no-sync set /coreos.com/network/config '{"Network":"10.244.0.0/16", "Backend":{"Type": "alloc"}}'
```


## Kubernetes Node

The node will run the following services:

  * docker
  * fleet
  * flannel
  * kubelet
  * proxy

One of the nodes will be elected master by fleet and will run the additional services:

  * apiserver
  * scheduler
  * replication controller
  * kube-register

### Edit knode.yaml cloud config

Grab the `INTERNAL_IP` for the kcontrol machine

```
gcloud compute instances list
```

Edit the knode.yaml and update the following lines:

#### Update knode.yaml with sed

```
sed -i "" -e 's/CONTROL-NODE-INTERNAL-IP/${KCONTROL_INTERNAL_IP}/g' knode.yaml
```

### Start knodes with gcloud

```
gcloud compute instances create knode1 knode2 knode3 knode4 knode5 \
--image-project coreos-cloud \
--image coreos-alpha-472-0-0-v20141017 \
--boot-disk-size 200GB \
--machine-type g1-small \
--can-ip-forward \
--metadata-from-file user-data=knode.yaml \
--zone us-central1-a
```

## Validation

### GCE Instances

```
gcloud compute instances list
```

### CoreOS Services

#### flannel

```
etcdctl --no-sync ls / -recursive
```

#### fleet

```
fleetctl list-machines
```

Note: if you see an 'unable to authenticate' error when you run `fleetctl`, then try first running:

```
ssh-add ~/.ssh/google_compute_engine
```

(Substitute the filename of your ssh keys as necessary).
These keys are [generated](https://cloud.google.com/compute/docs/console#sshkeys) the first time you run `gcloud compute ssh`.


## Delete all the nodes

Only do this if you need to start over or clean up after you are done.

```
gcloud compute instances delete knode1 knode2 knode3 knode4 knode5 kcontrol \
--zone us-central1-a --delete-disks all
```
