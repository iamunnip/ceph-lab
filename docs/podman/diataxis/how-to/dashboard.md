# Enable and open the Ceph dashboard

Run Ceph commands on **node-01**, the administration VM. The dashboard is the management UI served by the active Ceph Manager (MGR). It does not need a fourth VM. The cluster must already be healthy.

A service called the **management gateway** (`mgmt-gateway`) sits in front of the dashboard and serves HTTPS on port 443. It sends requests to whichever manager is active, so your URL keeps working after a manager failover. See [ADR 0010](../../adr/0010-production-aligned-baseline.md).

## Enable the dashboard

Check whether it is already running. If the result lists a `dashboard` address, skip this command:

```sh
sudo cephadm shell -- ceph mgr services
```

On a new cluster bootstrapped with `--skip-dashboard`, expect `{}`. Turn on the dashboard module:

```sh
sudo cephadm shell -- ceph mgr module enable dashboard
```

It prints nothing. The module runs inside the active manager, and the standby managers load it too, so it moves with a failover.

### Give the dashboard a certificate from the cephadm CA

The dashboard needs an HTTPS certificate. **Do not use `ceph dashboard create-self-signed-cert` in this lab.** The dashboard also uses its certificate as a client certificate when it asks Prometheus and Alertmanager for data through the gateway's internal port 29443. The gateway accepts only client certificates signed by the cephadm root CA (`cephadm-root-<fsid>`). With a self-signed certificate, the gateway answers `400 No required SSL certificate was sent`, and the dashboard shows **Failed to execute prometheus: Error parsing Prometheus Alertmanager response: Expecting value**. That happened on 2026-11-08.

cephadm can issue a certificate from its own CA. The output contains a private key, so it goes into a folder only root can open and never onto the screen. On **node-01**, create the folder:

```sh
sudo install -d -m 0700 /root/dashboard-cert
```

Create an empty file only root can read:

```sh
sudo install -m 0600 /dev/null /root/dashboard-cert/certs.json
```

Ask cephadm for a dashboard certificate and write it into that file:

```sh
sudo cephadm shell -- ceph orch certmgr generate-certificates dashboard | sudo tee /root/dashboard-cert/certs.json > /dev/null
```

Check that it holds a certificate and a key, without printing them:

```sh
sudo jq -r 'keys[]' /root/dashboard-cert/certs.json
```

Expect `cert` and `key`. Create the two empty target files:

```sh
sudo install -m 0600 /dev/null /root/dashboard-cert/dashboard.crt
```

```sh
sudo install -m 0600 /dev/null /root/dashboard-cert/dashboard.key
```

Copy the certificate out:

```sh
sudo jq -r .cert /root/dashboard-cert/certs.json | sudo tee /root/dashboard-cert/dashboard.crt > /dev/null
```

Copy the key out:

```sh
sudo jq -r .key /root/dashboard-cert/certs.json | sudo tee /root/dashboard-cert/dashboard.key > /dev/null
```

Check the certificate. It is public, so printing its names is fine:

```sh
sudo openssl x509 -in /root/dashboard-cert/dashboard.crt -noout -issuer -subject -ext subjectAltName
```

Expect `issuer=CN=cephadm-root-<your fsid>` and the names `node-01`, `dashboard_servers` and `192.168.56.11`. Give it to the dashboard:

```sh
sudo cephadm shell --mount /root/dashboard-cert:/c -- ceph dashboard set-ssl-certificate -i /c/dashboard.crt
```

Expect `SSL certificate updated`.

```sh
sudo cephadm shell --mount /root/dashboard-cert:/c -- ceph dashboard set-ssl-certificate-key -i /c/dashboard.key
```

Expect `SSL certificate key updated`. Both are now stored in the cluster's configuration store, shared by all managers. Remove the working copies:

```sh
sudo rm -r /root/dashboard-cert
```

Your browser never sees this certificate: it talks to the gateway, which has its own certificate from the same CA. The browser still warns, because your computer does not trust the cephadm CA. Use a certificate from a trusted CA in production.

If the dashboard was already running with another certificate, restart it so it loads the new one and drops cached connection details. Fail over the active manager, using the name from `ceph mgr stat`:

```sh
sudo cephadm shell -- ceph mgr fail node-01.bsllkm
```

## Create your login

Skip this if you already have a dashboard administrator. On node-01, create an empty file that only root can read:

```sh
sudo install -m 0600 /dev/null /root/ceph-dashboard-password
```

Generate a random password into it. `tee` writes into the existing file and keeps its permissions, and `> /dev/null` keeps the password off the screen:

```sh
openssl rand -base64 24 | tr -d '\n' | sudo tee /root/ceph-dashboard-password > /dev/null
```

Check the permissions:

```sh
sudo ls -l /root/ceph-dashboard-password
```

Expect `-rw-------`, owner `root` and size `32`.

Create the account. `-i /pw` reads the password from the mounted file, `administrator` is the role, and `--pwd_update_required` forces a new password at first login. Output is discarded because it includes the password hash:

```sh
sudo cephadm shell --mount /root/ceph-dashboard-password:/pw -- ceph dashboard ac-user-create admin -i /pw administrator --pwd_update_required > /dev/null
```

Check the account exists:

```sh
sudo cephadm shell -- ceph dashboard ac-user-show
```

Expect `["admin"]`. Check where the dashboard now listens:

```sh
sudo cephadm shell -- ceph mgr services
```

Expect `"dashboard": "https://192.168.56.11:8443/"`: the active manager's own address. Do not use this URL: it changes when another manager becomes active. The gateway below gives one fixed URL.

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

Expect three daemons, `mgmt-gateway.node-01`, `.node-02` and `.node-03`, each `running` on port 443: one on every host labeled `gateway`.

## Open it from your computer

The VMs sit on the VirtualBox host-only network 192.168.56.0/24, which your computer reaches directly. No port forwarding is needed. Every storage VM runs a gateway, so any of these opens the same dashboard:

```text
https://192.168.56.11/
```

```text
https://192.168.56.12/
```

```text
https://192.168.56.13/
```

Each gateway forwards to whichever manager is active, so none of these addresses changes on a manager failover. If one VM is down, use another address. There is no shared virtual IP, so switching is manual.

A browser warning is expected, because your computer does not trust the gateway's certificate. Confirm you are opening your own VM before accepting it. Sign in as `admin` and change the password when prompted.

### Give the gateways one certificate for all three addresses

cephadm creates the gateway certificate once, for the host of the first gateway, and reuses it on every gateway. It covers only that host: on 2026-11-09 the first gateway was node-01, so the certificate named only `DNS:node-01, IP:192.168.56.11`, and `.12` and `.13` showed a name mismatch. Which host comes first can differ between installs. A certificate stored in cephadm's cert store as `mgmt_gw_cert` and `mgmt_gw_key` replaces it on all gateways.

On **node-01**, create a folder only root can open:

```sh
sudo install -d -m 0700 /root/gw-cert
```

Create a self-signed certificate and key that name all three hosts, valid for 825 days:

```sh
sudo openssl req -x509 -newkey rsa:3072 -sha256 -days 825 -nodes -keyout /root/gw-cert/gw.key -out /root/gw-cert/gw.crt -subj "/CN=ceph-lab-gateway" -addext "subjectAltName=DNS:node-01,DNS:node-02,DNS:node-03,IP:192.168.56.11,IP:192.168.56.12,IP:192.168.56.13"
```

It prints a line of dots and `-----`. Check the names:

```sh
sudo openssl x509 -in /root/gw-cert/gw.crt -noout -subject -enddate -ext subjectAltName
```

Expect `CN=ceph-lab-gateway` and the six names and addresses. Store the certificate:

```sh
sudo cephadm shell --mount /root/gw-cert:/c -- ceph orch certmgr cert set mgmt_gw_cert -i /c/gw.crt
```

Expect `Certificate for mgmt_gw_cert set correctly`. Store the key:

```sh
sudo cephadm shell --mount /root/gw-cert:/c -- ceph orch certmgr key set mgmt_gw_key -i /c/gw.key
```

Expect `Key for mgmt_gw_key set correctly`. Write the new files to every gateway:

```sh
sudo cephadm shell -- ceph orch reconfig mgmt-gateway
```

`reconfig` writes the files but the running nginx keeps the old certificate. Restart the gateways; the UI is away for a few seconds:

```sh
sudo cephadm shell -- ceph orch restart mgmt-gateway
```

Expect `Scheduled to restart mgmt-gateway.node-01 ...` for all three. The gateways restart one after another, so wait about a minute; on 2026-11-09 node-03 still served the old certificate right after the command. From your computer, check one address:

```sh
echo | openssl s_client -connect 192.168.56.11:443 2>/dev/null | openssl x509 -noout -subject -ext subjectAltName
```

Expect `CN=ceph-lab-gateway` and all six names. Check all three gateways in full: the name, the end date, every address, that it stays valid for at least 30 more days (2592000 seconds), and the fingerprint. Read the fingerprint in its own command: on 2026-11-09, OpenSSL 4.1.0 on the host printed a wrong SHA-256 fingerprint when `-ext subjectAltName` was in the same command.

```sh
echo | openssl s_client -connect 192.168.56.11:443 2>/dev/null | openssl x509 -noout -subject -enddate -ext subjectAltName -checkend 2592000
```

```sh
echo | openssl s_client -connect 192.168.56.11:443 2>/dev/null | openssl x509 -noout -fingerprint -sha256
```

```sh
echo | openssl s_client -connect 192.168.56.12:443 2>/dev/null | openssl x509 -noout -subject -enddate -ext subjectAltName -checkend 2592000
```

```sh
echo | openssl s_client -connect 192.168.56.12:443 2>/dev/null | openssl x509 -noout -fingerprint -sha256
```

```sh
echo | openssl s_client -connect 192.168.56.13:443 2>/dev/null | openssl x509 -noout -subject -enddate -ext subjectAltName -checkend 2592000
```

```sh
echo | openssl s_client -connect 192.168.56.13:443 2>/dev/null | openssl x509 -noout -fingerprint -sha256
```

Expect, for each address, `subject=CN=ceph-lab-gateway`, a `notAfter=` date about 825 days ahead, the line `DNS:node-01, DNS:node-02, DNS:node-03, IP Address:192.168.56.11, IP Address:192.168.56.12, IP Address:192.168.56.13`, and `Certificate will not expire` from the first command of each pair. The `sha256 Fingerprint=` from the second command must be the same on all three: one certificate, shared by every gateway. The Ansible playbook runs the same checks and makes a new certificate when one fails.

Remove the working copies on node-01; the cert store keeps both:

```sh
sudo rm -r /root/gw-cert
```

The browser still warns once, because the certificate is self-signed. To stop the warning, import this certificate into your browser's trusted certificates. Save it from the browser's certificate viewer, or from your computer:

```sh
echo | openssl s_client -connect 192.168.56.11:443 2>/dev/null | openssl x509 > ceph-lab-gateway.crt
```

Only trust it in a lab browser profile. It expires after 825 days; repeat these steps before then.

To run the gateway on every storage VM, the `gateway` label is on all three hosts in the [cluster spec](../../../../configs/podman/cluster-spec.example.yaml), and the gateway entry in the [monitoring spec](../../../../configs/podman/monitoring-spec.yaml) has no `count`. On a cluster that already runs one gateway, add the label on **node-01**:

```sh
sudo cephadm shell -- ceph orch host label add node-01 gateway
```

```sh
sudo cephadm shell -- ceph orch host label add node-03 gateway
```

Expect `Added label gateway to host node-01` and the same for node-03. Then upload and apply the monitoring spec again as in the deployment guide. Expect `mgmt-gateway 3/3` in `ceph orch ls` after node-01 and node-03 have downloaded the nginx image, about a minute on 2026-11-08.

## Open Prometheus, Alertmanager and Grafana

The gateway also serves the monitoring tools under the same address:

| Tool | URL | Login |
| --- | --- | --- |
| Grafana | `https://192.168.56.12/grafana/` | none extra |
| Prometheus | `https://192.168.56.12/prometheus/` | Prometheus user and password |
| Alertmanager | `https://192.168.56.12/alertmanager/` | same user and password |

Prometheus and Alertmanager answer `401 Unauthorized` with `Www-Authenticate: Basic`, so the browser shows a login prompt. That is expected, not an error. cephadm stores the login. On **node-01**, read it privately, in your own terminal:

```sh
sudo cephadm shell -- ceph orch prometheus get-credentials
```

It prints a `user` and a `password`. Do not paste them into chats, tickets or Git. To set your own instead, replace the example values and run:

```sh
sudo cephadm shell -- ceph orch prometheus set-credentials YOUR_USER YOUR_PASSWORD
```

The ports 9095 (Prometheus) and 9093 (Alertmanager) on node-02 accept only TLS connections with a certificate from the cephadm CA, so a browser cannot use them directly. Use the gateway URLs.

The host-only network is reachable only from your computer, not from your LAN. Do not forward port 443 or any Ceph port to other machines.

## Check the UI

On the first login, after the password change, Ceph 20.2.5 shows a **Welcome to Ceph Dashboard** page that says "Add storage to begin using your cluster" and "The cluster cannot store data until storage is added". It shows this even when OSDs already exist. **Do not click "Add storage".** It opens the cluster expansion wizard, which can create OSDs on every available disk, including the reserve disks `/dev/sdc`. Choose to skip, or continue to the cluster overview.

After that, confirm that the dashboard shows three storage hosts, three OSDs up and in, and a healthy status. Cross-check on node-01:

```sh
sudo cephadm shell -- ceph -s
```

Expect `HEALTH_OK`.

## Troubleshooting

On the **gateway VM**, check that something listens on port 443:

```sh
sudo ss -lntp | grep ':443 '
```

Expect `nginx` in the output. If nothing listens, and `ceph health detail` on node-01 shows `CEPHADM_FAILED_DAEMON` for `mgmt-gateway.node-02`, read the gateway's log on node-02. Replace the fsid with yours from `ceph fsid`:

```sh
sudo journalctl -u ceph-81c36e54-db79-11f1-9a8b-08002739f937@mgmt-gateway.node-02.service --no-pager -n 40
```

If it shows `host not found in upstream "node-01:8443"`, the VMs cannot resolve each other's names. Do [make the host names resolvable](deploy-vm-cluster.md#make-the-host-names-resolvable) on all three storage VMs, then restart the gateway on node-01:

```sh
sudo cephadm shell -- ceph orch daemon restart mgmt-gateway.node-02
```

If logins or actions fail at random with `503 Service Unavailable` and `"detail": "Keep on looking"`, the gateway sent a POST to a standby manager and did not retry. This is a [known issue](deploy-vm-cluster.md#known-issue-dashboard-post-requests-can-fail-with-503) in this lab. Open the active manager's address from `ceph mgr services` instead.

If the dashboard shows **Error parsing Prometheus Alertmanager response: Expecting value**, check who issued the dashboard's certificate. On node-01:

```sh
sudo cephadm shell -- ceph config-key get mgr/dashboard/crt | openssl x509 -noout -issuer
```

If it says `CN=ceph-dashboard` instead of `CN=cephadm-root-<fsid>`, the gateway rejects it. Replace it as in [give the dashboard a certificate from the cephadm CA](#give-the-dashboard-a-certificate-from-the-cephadm-ca), then fail over the manager.

The gateway runs as a Podman container started by a systemd unit, like every Ceph daemon. systemd starts it at boot and restarts it after a failure; Podman itself has no background service to do that.

Then request the page locally. `-k` skips certificate checking for this one diagnostic:

```sh
curl -k -I https://127.0.0.1/
```

Expect `HTTP/1.1 200 OK`. A short run of `503` right after a manager failover is normal while the gateway finds the new active manager. If the local request works but your browser fails, check that your computer reaches the VM with `ping 192.168.56.12`, and that `vboxnet0` has the address 192.168.56.1.

See the [Ceph dashboard](https://docs.ceph.com/en/tentacle/mgr/dashboard/) and [management gateway](https://docs.ceph.com/en/tentacle/cephadm/services/mgmt-gateway/) documentation.
