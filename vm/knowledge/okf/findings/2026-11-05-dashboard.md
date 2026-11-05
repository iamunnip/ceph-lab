# Dashboard enablement and track reorganization, 2026-11-05

Status: dashboard enabled; public HTTPS page verified. Core cluster remains healthy.

## Scope and records

The user requested VM-specific documentation and knowledge under `vm/`, a core setup of three storage VMs using their second disks, and the Ceph UI. Existing nodes 04–05 remain reserved for optional extensions and were not modified. Moved root docs/knowledge into `vm/docs/` and `vm/knowledge/` and rebased Markdown links. The generic inventory template now has only three hosts; the observed iximiuz inventory retains all five actual VMs.

See [ADR 0007](../../../docs/adr/0007-vm-owned-docs-and-dashboard.md), the [dashboard guide](../../../docs/diataxis/how-to/dashboard.md), and the [provider guide](../../../docs/diataxis/how-to/iximiuz-access.md).

## Initial HTTPS enablement

On node-01, enabled the dashboard module, generated a self-signed certificate and set the bind address to `0.0.0.0`. The active manager was node-02. `ceph mgr services` advertised `https://10.8.0.2:8443/`, and the service returned HTTP 200 on loopback and the main interface.

Created dashboard account `admin` with administrator role and a random initial password stored only in root-readable `/root/ceph-dashboard-password` on node-01. Password change is required on first login. The API authentication test returned HTTP 201, accepted the administrator and confirmed the password-change flag. No password, password hash, session cookie or token is stored in this repository.

The user retrieves the initial password privately after connecting to node-01 using `sudo cat /root/ceph-dashboard-password`. The file contains the initial password only; it will not track a password changed through the UI. Remove the retained initial-password file after a successful password change if it is no longer needed.

## Measured iximiuz TLS incompatibility

The user explicitly requested `labctl expose`, without local port forwarding or an additional proxy. Initially ran:

```sh
labctl expose port 6aec22a0c4718fedbc76e5aa 8443 --https --public
```

The generated endpoint returned HTTP 503 from `iximiuz-labs-ingress`, reporting `TLSV1_ALERT_PROTOCOL_VERSION`. Direct dashboard testing confirmed TLS 1.3 returned 200 while TLS 1.2 failed. Inspection of the pinned dashboard module showed a TLS 1.3 minimum. Thus, local dashboard health did not imply compatibility with the platform's backend TLS connection.

## Approved provider-specific workaround

The user explicitly approved an HTTP backend inside this disposable lab while retaining public HTTPS. Set dashboard SSL to false, server port to 8080, and disabled/re-enabled the dashboard module to reload configuration. The final service address is `http://10.8.0.2:8080/`. Confirmed the listener belongs to `ceph-mgr` and local HTTP response identifies `Ceph-Dashboard` with status 200.

Exposed it using:

```sh
labctl expose port 6aec22a0c4718fedbc76e5aa 8080 --public
```

Working URL: [Ceph dashboard](https://6aec22a0c4718fedbc76e5aa-ac65c6.node-eu-d241.iximiuz.com/).

Public HTTPS verification returned HTTP 200 and downloaded the UI HTML. This checks the served application and transport, not an interactive browser session. Dashboard login remains required. The internal backend hop is HTTP; this is an explicitly approved iximiuz lab exception, not the recommended direct-VM configuration. VirtualBox and other direct-access instructions retain HTTPS on 8443.

The failed 8443 exposure was removed after the working 8080 exposure was verified. A second administrator authentication test against the final HTTP backend returned 201, accepted the account, and confirmed the required password change.

## Final cluster and limitations

After dashboard restart, `ceph -s` reported HEALTH_OK, three MONs in quorum, an active and standby MGR, three OSDs up/in and 33 active+clean PGs. Monitoring charts are deferred because no monitoring stack was deployed. Dashboard account authentication was tested; visual interaction with every dashboard page was not.

Manager failover can change the active dashboard host. Re-query `ceph mgr services` and expose the new host's playground. Other VM providers have not been live-tested. Existing storage data, root disks, client disks and WireGuard configuration were not changed by this task.

Documentation verification: all local links resolved across 26 Markdown files and `git diff --check` passed. No root-level docs/knowledge directories remain.

## Public access recheck after user report

The user reported the UI was not working. Rechecked the active node-02 backend, then ran `labctl expose port 6aec22a0c4718fedbc76e5aa 8080 --public` again. It generated a new [public dashboard URL](https://6aec22a0c4718fedbc76e5aa-a1e75a.node-eu-d241.iximiuz.com/), verified with HTTP 200. The previous URL also returned 200 during diagnosis. Public HTML and both entry JavaScript bundles returned 200, and public administrator authentication returned 201 without displaying credentials. No interactive browser session was tested, so the user's original browser symptom was not reproduced.

Current health has a RECENT_MGR_MODULE_CRASH warning for the standby node-01 dashboard at 07:22:31 UTC during the earlier module configuration change. All three MONs remain in quorum, three OSDs are up/in, and all 33 PGs are active+clean. The active node-02 dashboard serves normally. The crash record was not silently archived; standby dashboard recovery remains to inspect separately.

## Resolved standby port conflict

Inspected the retained crash report: `portend.Timeout: Port 8080 not free on 0.0.0.0`. The node-01 listener was the existing iximiuz `wstunnel` process, not Ceph. Choosing 8080 based only on the active node-02 had missed this standby conflict.

Verified 18080 was free on both manager hosts, changed `mgr/dashboard/server_port` to 18080, and restarted standby mgr.node-01.tkqxrk. Ceph correctly refused a direct active-manager restart; used a controlled `ceph mgr fail node-02.qhfpnw` after verifying the recovered standby. Node-01 became active and node-02 returned as standby.

Verified the active node-01 dashboard returned 200 on 18080, the standby node-02 dashboard listened on 18080 and returned its expected 303 redirect, and the new public HTTPS URL returned 200. Existing wstunnel on node-01:8080 was left untouched. Current exposure command:

```sh
labctl expose port 6aec229fdd5e89ace3613e0f 18080 --public
```

Current [dashboard URL](https://6aec229fdd5e89ace3613e0f-8a248e.node-eu-10a1.iximiuz.com/). After recovery, archived only the inspected crash ID to acknowledge it while retaining its report. No global warning suppression was used.

Disk verification with `ceph osd df` confirmed each of the three OSDs is 80 GiB, totaling 240 GiB raw. Each VM also has a separate 20 GiB root disk; that OS capacity is not part of the Ceph OSD capacity. Nominal three-replica capacity is about 80 GiB before overhead/headroom.

Final verification after the crash-health refresh: `ceph health detail` returned HEALTH_OK, `ceph crash ls-new` was empty, and `ceph crash stat` retained one recorded crash for history.
