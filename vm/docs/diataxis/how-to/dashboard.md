# Enable and open the Ceph dashboard

Run Ceph commands on **node-01**, the administration VM. The dashboard is the management UI served by the active Ceph Manager (MGR). It does not need a fourth VM. The cluster must already be healthy.

A service called the **management gateway** (`mgmt-gateway`) sits in front of the dashboard and serves HTTPS on port 443. It sends requests to whichever manager is active, so your URL keeps working after a manager failover. See [ADR 0010](../../adr/0010-production-aligned-baseline.md).

## Enable the dashboard

Check whether it is already running. If the result lists a `dashboard` address, skip this command:

```sh
sudo cephadm shell -- ceph mgr services
```

Turn on the dashboard module:

```sh
sudo cephadm shell -- ceph mgr module enable dashboard
```

Create a certificate for the manager. The example is self-signed, which is fine for learning. Use a certificate from a trusted CA in production:

```sh
sudo cephadm shell -- ceph dashboard create-self-signed-cert
```

Expect `Self-signed certificate created`. Do not regenerate it on every visit.

## Create your login

Skip this if you already have a dashboard administrator. On node-01, generate a random password into a file only root can read:

```sh
sudo sh -c 'umask 077; openssl rand -base64 24 | tr -d "\n" > /root/ceph-dashboard-password'
```

Create the account. Output is discarded because it includes the password hash:

```sh
sudo cephadm shell --mount /root/ceph-dashboard-password:/pw -- ceph dashboard ac-user-create admin -i /pw administrator --pwd_update_required > /dev/null
```

Read the password privately when you are ready to sign in:

```sh
sudo cat /root/ceph-dashboard-password
```

You must choose a new password at first login. This file keeps only the initial one, so delete it afterwards. Never commit passwords, hashes, keyrings or TLS private keys.

## Deploy the gateway and monitoring

Follow [Monitoring and HTTPS gateway](deploy-vm-cluster.md#monitoring-and-https-gateway) in the deployment guide. Then check that the gateway is running:

```sh
sudo cephadm shell -- ceph orch ps --daemon-type mgmt-gateway
```

Expect one daemon with status `running` on the host labeled `gateway`, node-02 in the example.

## Open it from VirtualBox and other reachable VMs

Your browser host must be able to reach the gateway VM on TCP 443. With VirtualBox, a host-only or bridged network can provide that. Open the gateway host's address, for example:

```text
https://192.168.50.12/
```

A browser warning is expected for a self-signed certificate. Confirm you are opening your own VM before accepting it. Sign in as `admin` and change the password when prompted. No labctl command is needed.

## Open it from iximiuz Labs

Use [labctl expose](iximiuz-access.md#expose-the-dashboard) on the playground running the gateway.

## Check the UI

After signing in, confirm that the dashboard shows three storage hosts, three OSDs up and in, and a healthy status. Cross-check on node-01:

```sh
sudo cephadm shell -- ceph -s
```

Expect `HEALTH_OK`.

## Troubleshooting

On the **gateway VM**, check that something listens on port 443:

```sh
sudo ss -lntp | grep ':443 '
```

Expect `nginx` in the output. Then request the page locally. `-k` skips certificate checking for this one diagnostic:

```sh
curl -k -I https://127.0.0.1/
```

Expect `HTTP/1.1 200 OK`. A single `503` right after a manager failover is normal: in our test the next sample, 5 seconds later, returned 200. If the local request works but your browser fails, check routing and firewall rules between them.

See the [Ceph dashboard](https://docs.ceph.com/en/tentacle/mgr/dashboard/) and [management gateway](https://docs.ceph.com/en/tentacle/cephadm/services/mgmt-gateway/) documentation.
