---
title: "Replacing a Failed Control Plane Node"
date: 2021-06-19T15:48:00Z
draft: true
tags:
  - etcd
categories:
  - hardware
  - kubernetes
---

In my case, I'm replacing a "healthy" node that has a bad HDD, however I did it
in such a way that it could be done if the node had actually failed and the disk
had completely died.

In the example below, I'm going to be replacing `cubert` which had an IP of
`192.168.10.24` with the exact same node.

If you're going to add a new control plane (as a 4th node), and then remove one,
I have a different note file for that...

## Topology

Just so we're all on the same page here is the local topology of hosts I'm
talking about. I have 3 control plane nodes named `larry`, `walt` and `igner`.
Here are their IP addresses. Many of the `etcd` commands/logs only reference
nodes by IP address.

| Host    | IP              |
|---------|-----------------|
| `larry` | `192.168.10.24` |
| `walt`  | `192.168.10.24` |
| `igner` | `192.168.10.24` |

## Backup

Before you do anything, make a backup of the etcd cluster. I'm not responsible
for anything you do to your cluster (or mine) so I always make sure I have
backups.
<!--more-->

```text
kubectl exec -it -n kube-system etcd-larry -- sh -c 'ETCDCTL_API=3 etcdctl \
  --endpoints https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key /etc/kubernetes/pki/etcd/healthcheck-client.key \
  snapshot save /var/lib/etcd/snapshot.db'

{"level":"info","ts":1623779176.1537933,"caller":"snapshot/v3_snapshot.go:119","msg":"created temporary db file","path":"/var/lib/etcd/snapshot.db.part"}
{"level":"info","ts":"2021-06-15T17:46:16.153Z","caller":"clientv3/maintenance.go:200","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":1623779176.1539862,"caller":"snapshot/v3_snapshot.go:127","msg":"fetching snapshot","endpoint":"https://127.0.0.1:2379"}
{"level":"info","ts":"2021-06-15T17:46:18.588Z","caller":"clientv3/maintenance.go:208","msg":"completed snapshot read; closing"}
{"level":"info","ts":1623779184.1301796,"caller":"snapshot/v3_snapshot.go:142","msg":"fetched snapshot","endpoint":"https://127.0.0.1:2379","size":"163 MB","took":7.97631882}
{"level":"info","ts":1623779184.130351,"caller":"snapshot/v3_snapshot.go:152","msg":"saved","path":"/var/lib/etcd/snapshot.db"}
Snapshot saved at /var/lib/etcd/snapshot.db
```

**NOTE:** You must use `/var/lib/etcd/snapshot.db` as the path since
`/var/lib/etcd` is one of the few mounted paths in the `etcd` pod. Since this
pod doesn't have the `tar` executable, you cannot use the `kubectl cp` command
to grab it out of there.

You can now copy your backup straight from `/var/lib/etcd/snapshot.db` on the host (`larry` in my case).

### Prep `etcd`

In order for things to go the smoothest, let's ensure the node we're replacing
isn't the etcd leader:

```shell
export ETCD="ETCDCTL_API=3 etcdctl --endpoints https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key /etc/kubernetes/pki/etcd/healthcheck-client.key"
```

```text
$ kubectl exec -it -n kube-system etcd-$HOST -- \
  sh -c "$ETCD endpoint status --cluster"
https://192.168.10.24:2379, b91fe73334be058, 3.4.13, 163 MB, false, false, 120478, 327026147, 327026147,
https://192.168.10.25:2379, 979324e546bccc76, 3.4.13, 163 MB, false, false, 120478, 327026146, 327026145,
https://192.168.10.26:2379, f97426ee61bf534c, 3.4.13, 163 MB, true, false, 120478, 327026149, 327026149,
```

Here we can see that `192.168.10.26:2379` is the leader. If you'd prefer a more
verbose output to the above command you can add `-w table` to get the longer
table form which includes headers:

```text
$ kubectl exec -it -n kube-system etcd-$HOST -- \
  sh -c "$ETCD endpoint status --cluster -w table"
+----------------------------+------------------+---------+---------+-----------+...
|          ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER |...
+----------------------------+------------------+---------+---------+-----------+...
| https://192.168.10.24:2379 |  b91fe73334be058 |  3.4.13 |  163 MB |     false |...
| https://192.168.10.25:2379 | 979324e546bccc76 |  3.4.13 |  163 MB |     false |...
| https://192.168.10.26:2379 | f97426ee61bf534c |  3.4.13 |  163 MB |      true |...
+----------------------------+------------------+---------+---------+-----------+...
```

### Shutdown the node

First let's shutdown `cubert` we don't really need any of the data on it:

```
ssh cubert
sudo shutdown -h now
```

Now re-kickstart the node. Make sure it has the exact same IP addr it had
before.

In my case, i'm re-kicking `cubert` which had `192.168.10.24`

## Configure IP Address with netplan

```yaml
```

## Run the Ansible plays

```console
$ cd ~/ansible

$ ansible-playbook -DKk ansible_user.yml -u sam -i 192.168.10.24,

$ ansible-playbook netdata.yml -i 192.168.10.24,

$ ansible-playbook setup_users.yml -i 192.168.10.24,

$ ansible-playbook docker.yml -i 192.168.10.24,

$ sudo apt-key adv --keyserver keyserver.ubuntu.com --recv FEEA9169307EA071

$ ansible-playbook setup_kubernetes.yml -i 192.168.10.24,

$ ansible-playbook kubernetes_resolvconf.yml -i 192.168.10.24,

$ sudo swapoff -a
$ sudo vim /etc/fstab
* comment out the swap line *
```

I'll need to write a play for my CA or find out where it is... the apiserver was
crashing without the stokesnet ca, so I manually put the ca contents in (via
vim).

```console
sudo mkdir /usr/share/ca-certificates/extra
sudo vim /usr/share/ca-certificates/extra/stokesnet_root_ca_2.crt
sudo update-ca-certificates
```

## Rejoin the node

Once the node has kubernetes on it, we can remove the old `etcd` reference and
perform our join

Here's a shortcut that makes sure we can quickly run `etcd` commands in pods:

```shell
export ETCD="ETCDCTL_API=3 etcdctl --endpoints https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key /etc/kubernetes/pki/etcd/healthcheck-client.key"
export ETCDHOST=cubert
```

First dump a live status just to have. This is run from my MacBook (across the
VPN) so you don't have to be on the cluster to do it. As long as `haproxy` is
configured correctly this should work even with the node down.

```console
$ kubectl exec -it -n kube-system etcd-$ETCDHOST -- \
  sh -c "$ETCD endpoint status --cluster"
Failed to get the status of endpoint https://192.168.10.24:2379 (context deadline exceeded)
https://192.168.10.25:2379, 979324e546bccc76, 3.4.13, 163 MB, true, false, 120477, 323884422, 323884422,
https://192.168.10.26:2379, f97426ee61bf534c, 3.4.13, 163 MB, false, false, 120477, 323884422, 323884422,
```

**TODO:** It seems like it'd be a good idea to elect another leader if we're
going to be nuking the leader... I should investigate next host.

Now dump all the node ids. If you've already shutdown the node you won't get a
nice status for the previous

```console
$ kubectl exec -it -n kube-system etcd-$ETCDHOST -- sh -c "$ETCD member list"
21168bfb73d2ddaa, started, cubert, https://192.168.10.24:2380, https://192.168.10.24:2379, false
979324e546bccc76, started, igner, https://192.168.10.25:2380, https://192.168.10.25:2379, false
f97426ee61bf534c, started, larry, https://192.168.10.26:2380, https://192.168.10.26:2379, false
```

Find the id of `cubert` as `21168bfb73d2ddaa`. Let's remove it so we can let k8s
re-initialize when the control plane comes up.

```console
$ kubectl exec -it -n kube-system etcd-$ETCDHOST -- \
  sh -c "$ETCD member remove 21168bfb73d2ddaa"
```

Now re-add it. It will be in a state where etcd recognizes it as a new member
and k8s will be able to join!

```console
$ kubectl exec -it -n kube-system etcd-$ETCDHOST -- \
  sh -c "$ETCD member add cubert --peer-urls=https://192.168.10.24:2380"
```

Now lets generate a new control-plane cert:

```console
$ sudo kubeadm init phase upload-certs --upload-certs
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
XXX
```

That gets uploaded to the cluster, cool, let's get a new join command as a
control plane... and...

```console
$ sudo kubeadm token create --print-join-command --certificate-key XXX
kubeadm join kubernetes.stokes.nc:8443 --token yyyyyy.YYY --discovery-token-ca-cert-hash sha256:zzz --control-plane --certificate-key XXX
```

VOILA! It works! and it's added as a new control plane node!