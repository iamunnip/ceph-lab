# Access the lab through iximiuz Labs

This optional guide adapts the [provider-independent deployment guide](deploy-vm-cluster.md) to iximiuz Labs. Readers using other VMs can skip it.

Our observed environment has five separate playgrounds already linked by WireGuard, but the core setup uses only node-01–03. Nodes 04–05 now run the native and k3s client extensions; they remain unnecessary for the core cluster setup. See the [observed inventory](../../../inventory/iximiuz.json) for playground IDs and actual mesh addresses. Do not copy those IDs into your own environment; use your own running playgrounds. WireGuard setup is already complete in this environment and is not changed by these guides.

## Connect

From your computer, replace the ID below with your playground ID:

```sh
labctl ssh YOUR_PLAYGROUND_ID
```

If your playground contains multiple machines, select one explicitly:

```sh
labctl ssh YOUR_PLAYGROUND_ID --machine node-01
```

Our existing node-01 can be accessed with:

```sh
labctl ssh 6aec229fdd5e89ace3613e0f
```

After connecting, the Ceph commands are the same as in the main guide. Use `laborant` instead of its example `ubuntu` account and the actual 10.8.0.1–3 mesh addresses instead of its example 192.168.50.11–13 addresses. The actual hostnames are node-01–03.

## Provider-specific observations

- Different playgrounds have the same eth0 address; the existing mesh addresses are distinct. Ceph uses those mesh addresses.
- Additional disks were already ext4 filesystems mounted at `/data`. Nodes 01–03 were converted only after empty-content checks and explicit user approval. This is not a generic instruction to erase `/dev/vdb`.
- The existing authorized_keys file lacked a final newline. Always append a public key with a leading and trailing newline.
- Package installation did not start services automatically. Chrony and SSH were explicitly started.
- Playground reset/deletion durability remains unverified. Preserve important data outside the lab.

See the [finding](../../../knowledge/okf/findings/2026-11-05-vm-bootstrap.md) for evidence and known limitations.

## Check the cluster

After connecting to node-01, use the standard Ceph commands in the [main guide](deploy-vm-cluster.md#verification-and-recovery). No custom validator or inventory file is needed to run those commands.

## Expose the dashboard

First complete [dashboard enablement](dashboard.md), including creation of a login. On node-01, query the active manager:

```sh
sudo cephadm shell -- ceph mgr services
```

Match its IP to your playground inventory. In our observed cluster the active manager is node-01 at 10.8.0.1. Exposure must target that playground, not automatically node-01.

### Backend TLS compatibility

We tested `labctl expose port ... 8443 --https --public` against Ceph 20.2.5. The public endpoint returned 503: iximiuz ingress could not negotiate the dashboard's TLS-1.3-only backend. The dashboard itself returned 200 over TLS 1.3, and TLS 1.2 was rejected. This is a measured compatibility issue, not a reason to disable HTTPS on other VM platforms.

For this disposable iximiuz lab, the user approved an HTTP backend with HTTPS at the public endpoint. We use port 18080 because port 8080 is already occupied by wstunnel on node-01. Before choosing a port, check it on every manager host with `sudo ss -lntp`, including standby managers. On **node-01**, apply that provider-specific configuration:

```sh
sudo cephadm shell -- ceph config set mgr mgr/dashboard/ssl false
sudo cephadm shell -- ceph config set mgr mgr/dashboard/server_port 18080
sudo cephadm shell -- ceph mgr module disable dashboard
sudo cephadm shell -- ceph mgr module enable dashboard
sudo cephadm shell -- ceph mgr services
```

The last command should show an HTTP address such as `http://10.8.0.1:18080/`. Module restart briefly interrupts UI access. The dashboard backend is unencrypted inside the lab; keep it within the disposable lab network. The public URL uses HTTPS and the dashboard still requires its login. Do not copy this setting into the VirtualBox/direct-access walkthrough, which retains dashboard HTTPS.

### Publish with labctl expose

On **your own computer**, run:

```sh
labctl expose port YOUR_ACTIVE_MANAGER_PLAYGROUND_ID 18080 --public
```

For the existing deployment:

```sh
labctl expose port 6aec229fdd5e89ace3613e0f 18080 --public
```

Do not add `--https` here: that flag describes the backend protocol, which is now HTTP. The generated browser URL is still HTTPS. If a future iximiuz ingress supports the dashboard's backend TLS, retain Ceph HTTPS and expose 8443 with `--https` instead.

If your playground contains multiple machines, add `--machine` with the actual machine name. Our example uses a separate single-machine playground per host.

Open the URL printed by the command, sign in as `admin`, and change the initial password. Expose only the dashboard port, not MON or OSD ports. The service must listen on all interfaces or the playground's main interface; the dashboard guide sets `server_addr` to `0.0.0.0`.

After a manager failover, query the dashboard address again and expose the new active manager's playground. A standby may redirect to a private address your browser cannot reach. No local forwarding command or extra service is required.

These commands follow the [iximiuz HTTP/HTTPS exposure documentation](https://labs.iximiuz.com/docs/playgrounds/expose-http-ports). See our [dashboard finding](../../../knowledge/okf/findings/2026-11-05-dashboard.md) for observed outcomes.

### Login for the existing deployment

The existing lab has an `admin` account. Retrieve its initial password privately on your own terminal:

```sh
labctl ssh 6aec229fdd5e89ace3613e0f
sudo cat /root/ceph-dashboard-password
```

Open the [existing dashboard](https://6aec229fdd5e89ace3613e0f-8a248e.node-eu-10a1.iximiuz.com/) and change that password when prompted. The stored file is only the initial password and does not update when you change it through the UI. This file/account was created for our deployment; other readers should create their own login using the main dashboard guide.
