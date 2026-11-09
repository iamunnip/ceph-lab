# Consume external Ceph storage from single-node k3s

Node-05 runs k3s and Ceph-CSI. Ceph itself remains on nodes 01 to 03; Rook is not used in this exercise. A 5 GiB PVC backs an nginx workload. node-05 has no extra disk; the storage comes from Ceph.

Examples use the Vagrant lab values: storage nodes 192.168.56.11 to 13 and k3s node-05 at 192.168.56.15. Start node-05 first from `vm/vagrant/podman/` on your computer with `vagrant up node-05`.

Status: planned. Not run on the Vagrant lab yet.

 The pinned [Ceph-CSI 3.18.2 matrix](https://raw.githubusercontent.com/ceph/ceph-csi/v3.18.2/README.md) includes Kubernetes 1.36. This walkthrough uses k3s v1.36.6+k3s1 and Helm v3.19.0.

## Prepare the pool on node-01

Run creation steps only once:

```sh
sudo cephadm shell -- ceph osd pool create k3s-rbd
```

```sh
sudo cephadm shell -- ceph osd pool application enable k3s-rbd rbd
```

```sh
sudo cephadm shell -- rbd pool init k3s-rbd
```

Tell the autoscaler to expect about half the cluster here:

```sh
sudo cephadm shell -- ceph osd pool set k3s-rbd target_size_ratio 0.5
```

```sh
sudo cephadm shell -- ceph osd pool set-quota k3s-rbd max_bytes 10737418240
```

The quota limits stored logical data to 10 GiB. It shares physical capacity with the native pool and other pools; it does not reserve extra disks.

Create the pool-scoped identity. Output is discarded so the key is not displayed:

```sh
sudo cephadm shell -- ceph auth get-or-create client.k3s mon 'profile rbd' osd 'profile rbd pool=k3s-rbd' mgr 'profile rbd pool=k3s-rbd' > /dev/null
```

The key moves to node-05 in a later step. Never transfer the admin keyring.

Find the cluster identifier on node-01:

```sh
sudo cephadm shell -- ceph fsid
```

## Install k3s on node-05

Skip installation if an appropriate k3s cluster already exists. Inspect its version, kubelet directory and networking before adapting the CSI settings.

Copy [k3s-config.example.yaml](../../../../configs/podman/clients/k3s/k3s-config.example.yaml) to node-05 as `/tmp/k3s-config.yaml`. From `vm/vagrant/podman/` on your computer:

```sh
vagrant upload ../../../configs/podman/clients/k3s/k3s-config.example.yaml /tmp/k3s-config.yaml node-05
```

 It already uses node-05's host-only address 192.168.56.15 and interface `enp0s8`. Without `flannel-iface`, k3s would pick the NAT card, whose address 10.0.2.15 is the same on every VM. The file turns on three things worth knowing:

- `secrets-encryption` encrypts Kubernetes Secrets at rest, including the Ceph key.
- `cluster-init` uses embedded etcd, which takes scheduled snapshots and lets you add servers later.
- Disabling `local-storage` avoids accidentally testing the local-path provisioner instead of Ceph.

On node-05, put the configuration in place:

```sh
sudo install -d -m 0755 /etc/rancher/k3s
```

```sh
sudo install -m 0600 /tmp/k3s-config.yaml /etc/rancher/k3s/config.yaml
```

Download the installer from the pinned release:

```sh
curl -fsSL https://raw.githubusercontent.com/k3s-io/k3s/v1.36.6%2Bk3s1/install.sh -o /tmp/install-k3s.sh
```

Run it:

```sh
sudo env INSTALL_K3S_VERSION=v1.36.6+k3s1 sh /tmp/install-k3s.sh
```

Check the node:

```sh
sudo k3s kubectl get nodes -o wide
```

Expect node-05 `Ready` with your chosen address. Check encryption:

```sh
sudo k3s secrets-encrypt status
```

Expect `Encryption Status: Enabled`.

Load NBD support and persist it across boots:

```sh
sudo modprobe nbd nbds_max=16
```

```sh
printf 'options nbd nbds_max=16\n' | sudo tee /etc/modprobe.d/ceph-nbd.conf
```

```sh
printf 'nbd\n' | sudo tee /etc/modules-load.d/ceph-nbd.conf
```

The kernel mounter is the production recommendation, but older kernels cannot load current Ceph keys, see the [native guide](native-rbd-client.md#install-a-compatible-mapper-on-node-04). The example StorageClass therefore sets `mounter: rbd-nbd`, which upstream labels [Alpha](https://raw.githubusercontent.com/ceph/ceph-csi/v3.18.2/docs/design/proposals/rbd-nbd.md). If kernel RBD worked on node-04 with the same kernel, remove that line from the StorageClass before creating volumes.

## Install pinned Helm and chart on node-05

```sh
curl -fsSL https://get.helm.sh/helm-v3.19.0-linux-amd64.tar.gz -o /tmp/helm-v3.19.0-linux-amd64.tar.gz
curl -fsSL https://get.helm.sh/helm-v3.19.0-linux-amd64.tar.gz.sha256sum -o /tmp/helm-v3.19.0.sha256sum
cd /tmp
sha256sum -c helm-v3.19.0.sha256sum
tar -xzf helm-v3.19.0-linux-amd64.tar.gz
sudo install -m 0755 linux-amd64/helm /usr/local/bin/helm
curl -fsSL https://ceph.github.io/csi-charts/rbd/ceph-csi-rbd-3.18.2.tgz -o /tmp/ceph-csi-rbd-3.18.2.tgz
sha256sum /tmp/ceph-csi-rbd-3.18.2.tgz
```

Helm checksum verification must say OK. The chart checksum must be `62e0c93615a63c7a4cc91211011f378c0318d00ba0e034be5c817c6f52dba897`, matching the [official chart index](https://ceph.github.io/csi-charts/index.yaml).

## Configure and install CSI

Copy the files in [clients/k3s](../../../../configs/podman/clients/k3s) to node-05, for example from `vm/vagrant/podman/` on your computer with `vagrant upload ../../../configs/podman/clients/k3s /home/vagrant/k3s node-05`. Edit `values.example.yaml`: replace the FSID placeholder. The monitor addresses already match the Vagrant lab. Confirm the actual kubelet directory is `/var/lib/kubelet` before retaining that value. Edit `storageclass.example.yaml` with the same FSID. No secrets belong in either file.

On node-05, create the namespace. CSI node plugins need privileged host access to mount volumes:

```sh
sudo k3s kubectl create namespace ceph-csi
```

```sh
sudo k3s kubectl label namespace ceph-csi pod-security.kubernetes.io/enforce=privileged
```

Create the Secret. Run this on **your computer**, from `vm/vagrant/podman/`. It pipes the key directly from node-01 into Kubernetes, so the key is never written to a file or displayed:

```sh
vagrant ssh node-01 -c 'sudo cephadm shell -- ceph auth get-key client.k3s 2>/dev/null' | vagrant ssh node-05 -c 'sudo k3s kubectl -n ceph-csi create secret generic ceph-rbd-client --from-literal=userID=k3s --from-file=userKey=/dev/stdin'
```

Expect `secret/ceph-rbd-client created`.

The client ID is `k3s`, without the `client.` prefix. CSI's node plugin needs privileged host access to mount volumes; it is not an ordinary application pod. Install the driver and StorageClass using the edited file paths:

```sh
sudo env KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm upgrade --install ceph-rbd /tmp/ceph-csi-rbd-3.18.2.tgz --namespace ceph-csi --values values.example.yaml
sudo k3s kubectl apply -f storageclass.example.yaml
sudo k3s kubectl get pods -n ceph-csi
```

Expect the provisioner and node plugin to become Running and fully Ready. This single-node lab uses one provisioner replica and OnDelete node-plugin updates to avoid automatically interrupting NBD processes.

Kubernetes Secrets still require access control, since anyone allowed to read the Secret through the API sees the key. Do not export it into Git.

## Create a PVC and workload

On node-05:

```sh
sudo k3s kubectl apply -f workload.yaml
sudo k3s kubectl -n ceph-demo get pvc,pods
sudo k3s kubectl -n ceph-demo rollout status deployment/ceph-web --timeout=120s
sudo k3s kubectl -n ceph-demo exec deployment/ceph-web -- wget -qO- http://127.0.0.1/
```

Expect a Bound 5 GiB PVC and a Ready pod. The response should say `Persistent Ceph storage from node-05 k3s.`. The explicit StorageClass is `ceph-rbd`; RWO means this filesystem volume is intended for one node at a time. The deployment uses Recreate to avoid overlapping consumers during updates.

## Prove persistence

On node-05, write a marker file into the volume. `exec -i` passes your input to `tee` inside the pod, which writes it to the file:

```sh
printf 'my-persistent-marker\n' | sudo k3s kubectl -n ceph-demo exec -i deployment/ceph-web -- tee /usr/share/nginx/html/persistence.txt
```

Expect `my-persistent-marker` echoed back. Record its checksum:

```sh
sudo k3s kubectl -n ceph-demo exec deployment/ceph-web -- sha256sum /usr/share/nginx/html/persistence.txt
```

Delete only the pod. The deployment creates a new one, which attaches the same volume:

```sh
sudo k3s kubectl -n ceph-demo delete pod -l app=ceph-web
```

```sh
sudo k3s kubectl -n ceph-demo rollout status deployment/ceph-web --timeout=120s
```

Check the file in the new pod:

```sh
sudo k3s kubectl -n ceph-demo exec deployment/ceph-web -- cat /usr/share/nginx/html/persistence.txt
```

```sh
sudo k3s kubectl -n ceph-demo exec deployment/ceph-web -- sha256sum /usr/share/nginx/html/persistence.txt
```

Expect the same marker and checksum. Keep the PVC: deleting a pod is different from deleting its storage claim. This class uses Retain, so deleting a PVC does not automatically destroy the underlying image; reclaiming it requires an explicit storage lifecycle procedure.

The test proves persistence across pod recreation, not Kubernetes node failover. With one k3s VM, application/node HA is outside this setup. Ceph-side HA and observed results are explained in the [capacity guide](../explanation/ha-and-capacity.md).

## Back up the Kubernetes datastore

k3s takes etcd snapshots on a schedule. To take one now, on node-05:

```sh
sudo k3s etcd-snapshot save --name manual
```

List them:

```sh
sudo ls /var/lib/rancher/k3s/server/db/snapshots
```

Snapshots stay on the same VM until you copy them elsewhere. Restoring also needs the server token from `/var/lib/rancher/k3s/server/token`, so store both securely outside the VM. This protects Kubernetes objects, not the data inside your volumes. This k3s has no VolumeSnapshot CRDs, so CSI snapshots are not available until you add them.
