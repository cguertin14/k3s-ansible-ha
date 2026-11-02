# Confined CloudNativePG AppArmor profile

## Purpose and status

After the scheduled GitHub Actions APT upgrade and kured reboots on 2026-11-02,
PostgreSQL processes encountered AppArmor signal denials. Audit records identified
`cri-containerd.apparmor.d` as the denying profile and
`cri-containerd.apparmor.d//&unconfined` as the peer. Immich failed with
`could not signal for checkpoint: Permission denied`; Grafana stalled in recovery
and Paperless exhausted connection slots with sessions stuck in authentication.
Setting the database pods to `Unconfined` restored service.

This role provides a durable **local compatibility policy**, not a claim that the
underlying Ubuntu bug has been fixed. The precise upstream regression is still
unidentified. The initial draft has been compiled with the affected host's parser;
its runtime behavior still needs verification under CNPG before calling it fixed.

The profile is based on containerd v2.3.4's default AppArmor template (Apache-2.0,
see the repository LICENSE). It retains its filesystem, mount, and kernel-interface
restrictions, while allowing signals and ptrace between peers labelled either
`cnpg-postgres` or `cnpg-postgres//&unconfined`. It grants receive-only signal
permissions from unconfined runtime processes, runc, and crun. These rules assume
the runtime is unconfined; revisit its actual label if that changes.

Peer rules apply to processes sharing this profile, not to an individual pod.
PID namespaces and the other Linux permission checks still apply. This is a
runtime-default-style container policy, not a PostgreSQL file allowlist.

The role owns `/etc/apparmor.d/cnpg-postgres`; it does not replace containerd's
generated default profile. Ubuntu's AppArmor service loads the policy at boot.
Ansible validates templates before installation and reloads changed or missing
profiles in enforce mode. Kernel upgrades remain included in the scheduled APT
upgrade, and kured retains responsibility for reboots.

## Deploy the host profile

From the repository root, using the normal SSH/become credentials:

```bash
ansible-playbook -i inventory/cluster/hosts.ini playbooks/apparmor.yml --check --diff
ansible-playbook -i inventory/cluster/hosts.ini playbooks/apparmor.yml
```

The second command changes host policy. The install and updates playbooks also
include this role. A second run should report no changes.

On each eligible node, verify:

```bash
sudo apparmor_status
sudo cat /sys/kernel/security/apparmor/profiles
systemctl is-enabled apparmor
```

The list must contain `cnpg-postgres (enforce)` before scheduling a pod that uses
it. Kubernetes will reject containers requesting a missing Localhost profile.

## Switch the CNPG manifests

In the infrastructure GitOps repository, replace the temporary pod security
context with the following on each affected Cluster:

```yaml
spec:
  podSecurityContext:
    runAsUser: 26
    runAsGroup: 26
    runAsNonRoot: true
    fsGroup: 26
    seccompProfile:
      type: RuntimeDefault
    appArmorProfile:
      type: Localhost
      localhostProfile: cnpg-postgres
```

Preserve the other fields: CNPG treats `podSecurityContext` as an override.
Allow Argo CD to reconcile the manifests. Existing pods may require recreation
by deleting only the database pods, with explicit operator authorization. Keep
the Cluster resources and PVCs. The profile cannot change on an existing
container without recreating it.

## Verify runtime behavior

Check the pod-level configuration:

```bash
kubectl get pods -A -l cnpg.io/cluster \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,TYPE:.spec.securityContext.appArmorProfile.type,PROFILE:.spec.securityContext.appArmorProfile.localhostProfile'
```

Each new pod should show `Localhost` and `cnpg-postgres`. Check the actual process
label as well, for example:

```bash
kubectl exec -n immich immich-postgres-1 -c postgres -- cat /proc/1/attr/current
```

It should show the `cnpg-postgres` profile (possibly stacked with `unconfined`)
in enforce mode. Verify that:

- All three database pods become Ready and their restart counts stabilize.
- PostgreSQL completes recovery and accepts real application connections.
- Immich, Grafana, and Paperless resume normal operation; Paperless no longer
  accumulates authentication sessions or reports exhausted connections.
- Host kernel logs show no new AppArmor denials for the new profile:

```bash
sudo journalctl -k --since '10 minutes ago' --grep 'apparmor="DENIED".*cnpg-postgres'
```

After the next normal kured reboot, repeat the profile-loaded and database-health
checks to confirm boot persistence. Parser success alone does not establish that
the kernel's stacked-label signal mediation works with this policy.

## Rollback and eventual removal

If the confined profile still causes denials, revert the GitOps profile type to
`Unconfined`, remove `localhostProfile`, reconcile, and recreate the affected
pods. Keep the audit evidence for refining the policy. No data-volume changes
are needed.

When a verified Ubuntu fix allows the runtime default profile to work again,
restore `RuntimeDefault` (or the operator defaults) in GitOps and recreate the
pods. Only remove the Ansible-managed profile after no containers use it.
