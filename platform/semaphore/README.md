# Semaphore

Semaphore runs the `bts.onprem` collection's playbooks against the on-prem
inventory. Three things live here:

- `values.yaml` — chart values, including the `incus` client setup below.
- `resources/` — extra manifests Argo CD applies (the sealed client certificate).
- `incus-client-sealed-secret.yaml` — sealed copy of the `incus` client config.

## Task template

The playbooks belong to the collection, so the template points there and uses
the inventory repo only as its inventory source.

| Field | Value |
| :--- | :--- |
| Repository | `backstage-technical-services/on-prem-ansible` |
| Playbook | `playbooks/wazuh_agent.yml`, `playbooks/patchmon_agent.yml` |
| Inventory | File, repository `Aspect102/on-prem_ansible_inventory`, path `inventory/incus.yml` |
| Environment | `ANSIBLE_INVENTORY_ENABLED` containing `community.general.incus` |
| Vault | vault password for the collection's encrypted PatchMon token |

The environment variable is required: a collection inventory plugin is never
auto-detected, and a task runs with the collection as its working directory, so
the inventory repo's own `ansible.cfg` is not read.

## The incus client

`community.general.incus` shells out to the `incus` CLI for both inventory and
connection, so the pod needs it. It gets there at pod start:

1. An init container unpacks `incus-client` from Alpine's repository into an
   emptyDir. It is musl-native, matching the Semaphore image, and there is no
   registry to build a derived image from.
2. The main container mounts that emptyDir at `/usr/local/sbin`, which is on the
   image's `PATH` and does not exist inside it, so nothing is shadowed. The
   libraries are alongside it, hence `LD_LIBRARY_PATH`.
3. The client certificate for the `bts` remote is mounted read-only at
   `/etc/incus` and pointed at with `INCUS_CONF`.

The certificate is a copy of the one on the workstation that already trusts the
remote. To rotate it, create the secret and reseal:

```bash
kubectl -n semaphore create secret generic semaphore-incus-client \
  --from-file=config.yml --from-file=client.crt --from-file=client.key \
  --from-file=bts.crt --dry-run=client -o yaml > /tmp/incus-client.yaml

kubeseal --controller-name sealed-secrets --controller-namespace kube-system \
  --format yaml < /tmp/incus-client.yaml \
  > platform/semaphore/resources/incus-client-sealed-secret.yaml
```

`bts.crt` is mounted as `servercerts/bts.crt` by the volume's `items`, because
Incus looks for the remote's certificate under that name.
