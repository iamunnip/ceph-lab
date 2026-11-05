# Enable and open the Ceph dashboard

Run Ceph commands on **node-01**, the administration VM. The dashboard is the management UI served by the active Ceph Manager (MGR). It does not require a fourth VM. The cluster must already be healthy.

## Enable HTTPS

If bootstrap already enabled the dashboard, check `ceph mgr services` first and skip completed steps. For a cluster bootstrapped with `--skip-dashboard`, run:

```sh
sudo cephadm shell -- ceph mgr module enable dashboard
sudo cephadm shell -- ceph dashboard create-self-signed-cert
sudo cephadm shell -- ceph config set mgr mgr/dashboard/server_addr 0.0.0.0
```

The first command enables the UI module. The second creates a lab TLS certificate; do not regenerate it on every visit. The third lets the dashboard listen on the VM's interfaces, including the interface used by iximiuz exposure. Keep browser access restricted appropriately by your platform's network rules. HTTPS remains enabled on port 8443.

## Create your login

Skip account creation if you already have a dashboard administrator. On node-01, enter the Ceph tools container:

```sh
sudo cephadm shell
```

Inside the container, read your chosen strong password without displaying it or putting it into shell history:

```sh
umask 077
read -s -p 'Choose a dashboard password: ' DASHBOARD_PASSWORD
printf '\n'
printf '%s' "$DASHBOARD_PASSWORD" > /tmp/ceph-dashboard-password
unset DASHBOARD_PASSWORD
```

Create the account, using the password file. The password must be changed on first login:

```sh
ceph dashboard ac-user-create admin -i /tmp/ceph-dashboard-password administrator --pwd_update_required
rm /tmp/ceph-dashboard-password
exit
```

If the password policy rejects it, choose a stronger password and repeat password entry and account creation. Do not commit passwords, hashes, keyrings or TLS private keys.

## Find the current dashboard address

Back on node-01:

```sh
sudo cephadm shell -- ceph mgr services
```

Expect a result like this, with your active manager's actual address:

```json
{
  "dashboard": "https://192.168.50.12:8443/"
}
```

Allow time for the module to start if the result is initially empty. Manager failover can change the address. Query this command again rather than assuming node-01 always serves the UI.

## VirtualBox and other reachable VMs

Open the address returned above in your browser. For example:

```text
https://192.168.50.12:8443/
```

Your browser host must be able to reach the VM. With VirtualBox, a correctly configured host-only or bridged network can provide that access. Permit TCP 8443 from your browser host in the VM firewall if necessary. No labctl command is required.

A browser warning is expected for the self-signed lab certificate. Confirm that you are opening your own VM before accepting it. Use a trusted certificate for a longer-lived shared deployment. Sign in as `admin` and change the initial password when prompted.

## iximiuz Labs

Use [labctl expose](iximiuz-access.md#expose-the-dashboard) on the playground running the **active manager**. The provider guide records a measured backend TLS incompatibility and the approved lab-only HTTP backend workaround. The public URL remains HTTPS and requires the dashboard login. Other VM platforms retain direct HTTPS. No local forwarding command or additional service is needed.

## Check the UI

After signing in, confirm that the dashboard shows three storage hosts, three OSDs up/in, and healthy cluster status. Cross-check on node-01:

```sh
sudo cephadm shell -- ceph -s
sudo cephadm shell -- ceph health detail
```

Monitoring charts can be unavailable because this lab skips the monitoring stack. That does not prevent using the Ceph hosts, OSDs, pools and health pages.

## Troubleshooting

If the browser cannot connect, check the address from `ceph mgr services`, confirm the active manager and inspect port 8443 **on that VM**:

```sh
sudo ss -lntp | grep 8443
curl -k -I https://127.0.0.1:8443/
```

The listener should include `0.0.0.0:8443`; the HTTPS request should return an HTTP response. `-k` bypasses certificate verification only for this local diagnostic with the self-signed lab certificate. An inaccessible page may indicate host routing/firewall issues or an exposure targeting the standby manager. Keep HTTPS for direct VM access; the separately documented iximiuz backend workaround is limited to that provider.

See the [Ceph dashboard documentation](https://docs.ceph.com/en/tentacle/mgr/dashboard/) for TLS, accounts and module settings.
