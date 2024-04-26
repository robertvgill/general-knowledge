# Creating and Using an etcd Cluster

# Table of Contents

1. [Introduction to etcd and the Raft Consensus Algorithm](#introduction-to-etcd-and-the-raft-consensus-algorithm)
2. [Prerequisites](#prerequisites)
3. [Steps](#steps)
    - [Running a 3 node etcd cluster](#running-a-3-node-etcd-cluster)
    - [Testing the etcd Cluster](#testing-the-etcd-cluster)
    - [Cleanup](#cleanup)
4. [Conclusion](#conclusion)

## Introduction to etcd and the Raft Consensus Algorithm

**etcd** is a distributed key-value store used for shared configuration and service discovery in distributed systems. It uses the **Raft consensus algorithm** to manage a highly available replicated log. The Raft algorithm ensures that all nodes in the etcd cluster agree on the state of the distributed log, even in the presence of node failures or network partitions.

In a Raft-based system like etcd, each node in the cluster can be in one of three states: leader, follower, or candidate. The leader is responsible for coordinating updates to the distributed log and replicating these updates to followers. Followers simply replicate the log entries sent by the leader. In the event of leader failure, Raft elects a new leader from the available candidates.

## Prerequisites

Before starting, make sure you have the following:
- Podman installed (for running etcd in containers)
- etcdctl installed by following the instructions provided on the official etcd website: [etcdctl Installation Guide](https://etcd.io/docs/v3.5/op-guide/install/#installing-etcd)

## Steps

You can start an etcd cluster with three servers using the following command-lines:

### Running a 3 node etcd cluster

**1. Start the first etcd server**
```sh
podman run -d --name etcd0 \
  --net=host \
  --privileged \
  quay.io/coreos/etcd:v3.5.0 \
  /usr/local/bin/etcd \
  --name etcd0 \
  --initial-advertise-peer-urls http://localhost:2380 \
  --listen-peer-urls http://localhost:2380 \
  --advertise-client-urls http://localhost:2360 \
  --listen-client-urls http://localhost:2360 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster etcd0=http://localhost:2380,etcd1=http://localhost:2381,etcd2=http://localhost:2382 \
  --initial-cluster-state new
```
**2. Start the second etcd server**
```sh
podman run -d --name etcd1 \
  --net=host \
  --privileged \
  quay.io/coreos/etcd:v3.5.0 \
  /usr/local/bin/etcd \
  --name etcd1 \
  --initial-advertise-peer-urls http://localhost:2381 \
  --listen-peer-urls http://localhost:2381 \
  --advertise-client-urls http://localhost:2361 \
  --listen-client-urls http://localhost:2361 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster etcd0=http://localhost:2380,etcd1=http://localhost:2381,etcd2=http://localhost:2382 \
  --initial-cluster-state new
```
**3. Start the third etcd server**
```sh
podman run -d --name etcd2 \
  --net=host \
  --privileged \
  quay.io/coreos/etcd:v3.5.0 \
  /usr/local/bin/etcd \
  --name etcd2 \
  --initial-advertise-peer-urls http://localhost:2382 \
  --listen-peer-urls http://localhost:2382 \
  --advertise-client-urls http://localhost:2362 \
  --listen-client-urls http://localhost:2362 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster etcd0=http://localhost:2380,etcd1=http://localhost:2381,etcd2=http://localhost:2382 \
  --initial-cluster-state new
```

### Testing the etcd Cluster

You can now test the etcd cluster by performing read and write operations.

**Writing Data**

To write data to etcd, you can use either the `etcdctl` tool or `podman exec` command. For example, to write a key-value pair:

```sh
etcdctl --endpoints=http://localhost:2360 put myKey myValue
```
```sh
podman exec etcd0 etcdctl --endpoints=http://localhost:2360 put myKey myValue
```
**Reading Data**

To read data from etcd, you can again use the `etcdctl` tool or `podman exec` command. For example, to read the value of the key `mykey`:
```sh
etcdctl --endpoints=http://localhost:2360 get myKey
```
```sh
podman exec etcd0 etcdctl --endpoints=http://localhost:2360 get myKey
```
### Cleanup

When you're done testing, you can stop and remove the etcd servers:
```sh
# Stop etcd servers
podman stop etcd0 etcd1 etcd2

# Remove etcd servers
podman rm etcd0 etcd1 etcd2
```

## Conclusion
You've successfully set up an etcd cluster with three servers using Podman containers and performed read and write operations on the cluster.