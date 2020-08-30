---
title: Example Scenarios
weight: 4
---

These example scenarios for backup and restore are different based on your version of RKE.

{{% tabs %}}
{{% tab "RKE v0.2.0+" %}}

This walkthrough will demonstrate how to restore an etcd cluster from a local snapshot with the following steps:

1. [Back up the cluster](#1-back-up-the-cluster)
1. [Simulate a node failure](#2-simulate-a-node-failure)
1. [Add a new etcd node to the cluster](#3-add-a-new-etcd-node-to-the-kubernetes-cluster)
1. [Restore etcd on the new node from the backup](#4-restore-etcd-on-the-new-node-from-the-backup)
1. [Confirm that cluster operations are restored](#5-confirm-that-cluster-operations-are-restored)

In this example, the Kubernetes cluster was deployed on two AWS nodes.

|  Name |    IP    |          Role          |
|:-----:|:--------:|:----------------------:|
| node1 | 10.0.0.1 | [controlplane, worker] |
| node2 | 10.0.0.2 | [etcd]                 |


### 1. Back Up the Cluster

Take a local snapshot of the Kubernetes cluster.

You can upload this snapshot directly to an S3 backend with the [S3 options]({{<baseurl>}}/rke/latest/en/etcd-snapshots/one-time-snapshots/#options-for-rke-etcd-snapshot-save).

```
$ rke etcd snapshot-save --name snapshot.db --config cluster.yml
```

### 2. Simulate a Node Failure

To simulate the failure, let's power down `node2`.

```
root@node2:~# poweroff
```

|  Name |    IP    |          Role          |
|:-----:|:--------:|:----------------------:|
| node1 | 10.0.0.1 | [controlplane, worker] |
| ~~node2~~ | ~~10.0.0.2~~ | ~~[etcd]~~     |

###  3. Add a New etcd Node to the Kubernetes Cluster

Before updating and restoring etcd, you will need to add the new node into the Kubernetes cluster with the `etcd` role. In the `cluster.yml`, comment out the old node and add in the new node.

```yaml
nodes:
    - address: 10.0.0.1
      hostname_override: node1
      user: ubuntu
      role:
        - controlplane
        - worker
#    - address: 10.0.0.2
#      hostname_override: node2
#      user: ubuntu
#      role:
#       - etcd
    - address: 10.0.0.3
      hostname_override: node3
      user: ubuntu
      role:
        - etcd
```

###  4. Restore etcd on the New Node from the Backup

> **Prerequisite:** Ensure your `cluster.rkestate` is present before starting the restore, because this contains your certificate data for the cluster.

After the new node is added to the `cluster.yml`, run the `rke etcd snapshot-restore` to launch `etcd` from the backup:

```
$ rke etcd snapshot-restore --name snapshot.db --config cluster.yml
```

The snapshot is expected to be saved at `/opt/rke/etcd-snapshots`.

If you want to directly retrieve the snapshot from S3, add in the [S3 options](#options-for-rke-etcd-snapshot-restore).

> **Note:** As of v0.2.0, the file `pki.bundle.tar.gz` is no longer required for the restore process because the certificates required to restore are preserved within the `cluster.rkestate`.

### 5. Confirm that Cluster Operations are Restored

The `rke etcd snapshot-restore` command triggers `rke up` using the new `cluster.yml`. Confirm that your Kubernetes cluster is functional by checking the pods on your cluster.

```
> kubectl get pods                                                    
NAME                     READY     STATUS    RESTARTS   AGE
nginx-65899c769f-kcdpr   1/1       Running   0          17s
nginx-65899c769f-pc45c   1/1       Running   0          17s
nginx-65899c769f-qkhml   1/1       Running   0          17s
```

{{% /tab %}}
{{% tab "RKE prior to v0.2.0" %}}

This walkthrough will demonstrate how to restore an etcd cluster from a local snapshot with the following steps:

1. [Take a local snapshot of the cluster](#take-a-local-snapshot-of-the-cluster-rke-prior-to-v0.2.0)
1. [Store the snapshot externally](#store-the-snapshot-externally-rke-prior-to-v0.2.0)
1. [Simulate a node failure](#simulate-a-node-failure-rke-prior-to-v0.2.0)
1. [Remove the Kubernetes cluster and clean the nodes](#remove-the-kubernetes-cluster-and-clean-the-nodes-rke-prior-to-v0.2.0)
1. [Retrieve the backup and place it on a new node](#retrieve-the-backup-and-place-it-on-a-new-node-rke-prior-to-v0.2.0)
1. [Add a new etcd node to the Kubernetes cluster](#add-a-new-etcd-node-to-the-kubernetes-cluster-rke-prior-to-v0.2.0)
1. [Restore etcd on the new node from the backup](#restore-etcd-on-the-new-node-from-the-backup-rke-prior-to-v0.2.0)
1. [Restore Operations on the Cluster](#restore-operations-on-the-cluster-rke-prior-to-v0.2.0)

### Example Scenario of restoring from a Local Snapshot

In this example, the Kubernetes cluster was deployed on two AWS nodes.

|  Name |    IP    |          Role          |
|:-----:|:--------:|:----------------------:|
| node1 | 10.0.0.1 | [controlplane, worker] |
| node2 | 10.0.0.2 | [etcd]                 |

<a id="take-a-local-snapshot-of-the-cluster-rke-prior-to-v0.2.0"></a>
### 1. Take a Local Snapshot of the Cluster

Back up the Kubernetes cluster by taking a local snapshot:

```
$ rke etcd snapshot-save --name snapshot.db --config cluster.yml
```

<a id="store-the-snapshot-externally-rke-prior-to-v0.2.0"></a>
### 2. Store the Snapshot Externally

After taking the etcd snapshot on `node2`, we recommend saving this backup in a persistent place. One of the options is to save the backup and `pki.bundle.tar.gz` file on an S3 bucket or tape backup.

```
# If you're using an AWS host and have the ability to connect to S3
root@node2:~# s3cmd mb s3://rke-etcd-backup
root@node2:~# s3cmd \
  /opt/rke/etcd-snapshots/snapshot.db \
  /opt/rke/etcd-snapshots/pki.bundle.tar.gz \
  s3://rke-etcd-backup/
```

<a id="simulate-a-node-failure-rke-prior-to-v0.2.0"></a>
### 3. Simulate a Node Failure

To simulate the failure, let's power down `node2`.

```
root@node2:~# poweroff
```

|  Name |    IP    |          Role          |
|:-----:|:--------:|:----------------------:|
| node1 | 10.0.0.1 | [controlplane, worker] |
| ~~node2~~ | ~~10.0.0.2~~ | ~~[etcd]~~     |

<a id="remove-the-kubernetes-cluster-and-clean-the-nodes-rke-prior-to-v0.2.0"></a>
### 4. Remove the Kubernetes Cluster and Clean the Nodes

The following command removes your cluster and cleans the nodes so that the cluster can be restored without any conflicts:

```
rke remove --config rancher-cluster.yml
```

<a id="retrieve-the-backup-and-place-it-on-a-new-node-rke-prior-to-v0.2.0"></a>
### 5. Retrieve the Backup and Place it On a New Node

Before restoring etcd and running `rke up`, we need to retrieve the backup saved on S3 to a new node, e.g. `node3`. 

```
# Make a Directory
root@node3:~# mkdir -p /opt/rke/etcdbackup

# Get the Backup from S3
root@node3:~# s3cmd get \
  s3://rke-etcd-backup/snapshot.db \
  /opt/rke/etcd-snapshots/snapshot.db

# Get the pki bundle from S3
root@node3:~# s3cmd get \
  s3://rke-etcd-backup/pki.bundle.tar.gz \
  /opt/rke/etcd-snapshots/pki.bundle.tar.gz
```

> **Note:** If you had multiple etcd nodes, you would have to manually sync the snapshot and `pki.bundle.tar.gz` across all of the etcd nodes in the cluster.

<a id="add-a-new-etcd-node-to-the-kubernetes-cluster-rke-prior-to-v0.2.0"></a>
###  6. Add a New etcd Node to the Kubernetes Cluster

Before updating and restoring etcd, you will need to add the new node into the Kubernetes cluster with the `etcd` role. In the `cluster.yml`, comment out the old node and add in the new node. `

```yaml
nodes:
    - address: 10.0.0.1
      hostname_override: node1
      user: ubuntu
      role:
        - controlplane
        - worker
#    - address: 10.0.0.2
#      hostname_override: node2
#      user: ubuntu
#      role:
#       - etcd
    - address: 10.0.0.3
      hostname_override: node3
      user: ubuntu
      role:
        - etcd
```

<a id="restore-etcd-on-the-new-node-from-the-backup-rke-prior-to-v0.2.0"></a>
###  7. Restore etcd on the New Node from the Backup

After the new node is added to the `cluster.yml`, run the `rke etcd snapshot-restore` command to launch `etcd` from the backup:

```
$ rke etcd snapshot-restore --name snapshot.db --config cluster.yml
```

The snapshot and `pki.bundle.tar.gz` file are expected to be saved at `/opt/rke/etcd-snapshots` on each etcd node.

<a id="restore-operations-on-the-cluster-rke-prior-to-v0.2.0"></a>
### 8. Restore Operations on the Cluster

Finally, we need to restore the operations on the cluster. We will make the Kubernetes API point to the new `etcd` by running `rke up` again using the new `cluster.yml`.

```
$ rke up --config cluster.yml
```

Confirm that your Kubernetes cluster is functional by checking the pods on your cluster.

```
> kubectl get pods                                                    
NAME                     READY     STATUS    RESTARTS   AGE
nginx-65899c769f-kcdpr   1/1       Running   0          17s
nginx-65899c769f-pc45c   1/1       Running   0          17s
nginx-65899c769f-qkhml   1/1       Running   0          17s
```

{{% /tab %}}
{{% /tabs %}}
