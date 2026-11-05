# Consume external Ceph storage from single-node k3s

Node-05 runs k3s and Ceph-CSI. Ceph itself remains on nodes 01–03; Rook is not used in this exercise. A 5 GiB PVC backs an nginx workload. Node-05's local second disk is untouched and is not added to Ceph.

Examples use storage IPs 192.168.50.11–13 and k3s IP 192.168.50.15. Replace them with your addresses. The pinned [Ceph-CSI 3.18.2 matrix](https://raw.githubusercontent.com/ceph/ceph-csi/v3.18.2/README.md) includes Kubernetes 1.36. This walkthrough uses k3s v1.36.6+k3s1 and Helm v3.19.0.

## Prepare the pool on node-01

Run creation steps only once:

```sh
sudo cephadm shell -- ceph osd pool create k3s-rbd 8
sudo cephadm shell -- ceph osd pool application enable k3s-rbd rbd
sudo cephadm shell -- rbd pool init k3s-rbd
sudo cephadm shell -- ceph osd pool set k3s-rbd size 3
sudo cephadm shell -- ceph osd pool set k3s-rbd min_size 2
sudo cephadm shell -- ceph osd pool set-quota k3s-rbd max_bytes 42949672960
```

The quota limits stored logical data to 40 GiB. It shares physical capacity with the native pool and other pools; it does not reserve extra disks.

Create the pool-scoped identity, placing its output in a temporary file inside the tools container rather than printing it:

```sh
sudo cephadm shell -- ceph auth get-or-create client.k3s mon 'profile rbd' osd 'profile rbd pool=k3s-rbd' mgr 'profile rbd pool=k3s-rbd' -o /tmp/k3s.keyring
```

Export its key into a private directory on node-01, then transfer it using your authenticated SSH access:

```sh
umask 077
mkdir -p ~/ceph-k3s-transfer
sudo cephadm shell -- ceph auth get-key client.k3s > ~/ceph-k3s-transfer/userKey
scp ~/ceph-k3s-transfer/userKey ubuntu@192.168.50.15:~/ceph-k3s.key
```

Never transfer the admin keyring. For iximiuz, use authenticated playground file transfer appropriate to your access setup. The Secret is created on node-05 below; remove temporary key files after verifying it.

Find the cluster identifier on node-01:

```sh
sudo cephadm shell -- ceph fsid
```

## Install k3s on node-05

Skip installation if an appropriate k3s cluster already exists. Inspect its version, kubelet directory and networking before adapting the CSI settings.

For a new node-05, fetch the installer from the pinned release and install:

```sh
curl -fsSL https://raw.githubusercontent.com/k3s-io/k3s/v1.36.6%2Bk3s1/install.sh -o /tmp/install-k3s.sh
sudo env INSTALL_K3S_VERSION=v1.36.6+k3s1 sh /tmp/install-k3s.sh server --node-name node-05 --node-ip 192.168.50.15 --advertise-address 192.168.50.15 --disable traefik --disable servicelb --disable local-storage
sudo k3s kubectl get nodes -o wide
```

Expect node-05 to become Ready. When multiple VM interfaces exist, select the appropriate interface with `--flannel-iface YOUR_INTERFACE` if needed. The observed iximiuz install selected its existing mesh0 interface; networking was not reconfigured. Disabling local-storage avoids accidentally testing the local-path provisioner instead of Ceph.

Load NBD support and persist it across boots:

```sh
sudo modprobe nbd nbds_max=16
printf 'nbd\n' | sudo tee /etc/modules-load.d/ceph-csi.conf
```

This lab uses the RBD-NBD mounter with Ceph-CSI's newer userspace tools because of the observed client/kernel credential compatibility issue. Upstream labels [RBD-NBD support Alpha](https://raw.githubusercontent.com/ceph/ceph-csi/v3.18.2/docs/design/proposals/rbd-nbd.md); it is a lab choice, not a production recommendation.

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

Copy the three files in [clients/k3s](../../../clients/k3s/) to node-05. Edit `values.example.yaml`: replace the FSID placeholder and all three monitor addresses. Confirm the actual kubelet directory is `/var/lib/kubelet` before retaining that value. Edit `storageclass.example.yaml` with the same FSID. No secrets belong in either file.

On node-05, create the namespace and Secret from the transferred private key:

```sh
sudo k3s kubectl create namespace ceph-csi
sudo k3s kubectl label namespace ceph-csi pod-security.kubernetes.io/enforce=privileged
sudo k3s kubectl -n ceph-csi create secret generic ceph-rbd-client --from-literal=userID=k3s --from-file=userKey="$HOME/ceph-k3s.key"
```

The client ID is `k3s`, without the `client.` prefix. CSI's node plugin needs privileged host access to mount volumes; it is not an ordinary application pod. Install the driver and StorageClass using the edited file paths:

```sh
sudo env KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm upgrade --install ceph-rbd /tmp/ceph-csi-rbd-3.18.2.tgz --namespace ceph-csi --values values.example.yaml
sudo k3s kubectl apply -f storageclass.example.yaml
sudo k3s kubectl get pods -n ceph-csi
```

Expect the provisioner and node plugin to become Running and fully Ready. This single-node lab uses one provisioner replica and OnDelete node-plugin updates to avoid automatically interrupting NBD processes.

Remove `~/ceph-k3s.key` and the private key staging copy on node-01 once access is verified. Kubernetes Secrets require access control; base64 encoding is not encryption. Do not export the Secret into Git.

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

Create a unique marker, delete only the pod, and check the replacement pod:

```sh
sudo k3s kubectl -n ceph-demo exec deployment/ceph-web -- sh -c 'printf "my-persistent-marker\n" > /usr/share/nginx/html/persistence.txt; sha256sum /usr/share/nginx/html/persistence.txt'
sudo k3s kubectl -n ceph-demo delete pod -l app=ceph-web
sudo k3s kubectl -n ceph-demo rollout status deployment/ceph-web --timeout=120s
sudo k3s kubectl -n ceph-demo exec deployment/ceph-web -- cat /usr/share/nginx/html/persistence.txt
sudo k3s kubectl -n ceph-demo exec deployment/ceph-web -- sha256sum /usr/share/nginx/html/persistence.txt
```

Expect the same marker and checksum. Keep the PVC: deleting a pod is different from deleting its storage claim. This class uses Retain, so deleting a PVC does not automatically destroy the underlying image; reclaiming it requires an explicit storage lifecycle procedure.

The test proves persistence across pod recreation, not Kubernetes node failover. With one k3s VM, application/node HA is outside this setup. Ceph-side HA and observed results are explained in the [capacity guide](../explanation/ha-and-capacity.md) and [client finding](../../../knowledge/okf/findings/2026-11-05-client-storage.md).
