# satellite-aap-demo-artifacts

Ansible content for a Red Hat Satellite 6.19 + AAP SysAdmin demo: Day-0 content
lifecycle setup, a PXE-less boot-disk provisioning flow with AAP provisioning
callback, and OpenSCAP CIS compliance auditing.

This repo started from an AI-generated blueprint. It was reviewed against a
live Satellite instance (Foreman API + `ansible_variables`) and AAP before
being committed here; several parts of the original draft were fabricated or
wrong. Corrections are documented below so they aren't silently lost.

## Run order

1. `playbooks/00_sync_provisioning_templates.yml` — uploads the custom
   kickstart + snippet as Satellite Template records (the original blueprint
   never did this; the templates would have just been inert files in git).
2. `playbooks/01_satellite_setup_day0.yml` — lifecycle environments, content
   view, activation key.
3. `playbooks/02_satellite_hostgroup_callback_prep.yml` — host group +
   AAP callback parameters.
4. `playbooks/04_openscap_cis_audit_remediation.yml` (first play) — creates
   the CIS Level 1 and Level 2 compliance policies in Satellite.
5. Provision the host (see "Boot disk ISO" below), then AAP runs
   `03_aap_day1_post_install_hardening.yml` via the provisioning callback.
6. `04_openscap_cis_audit_remediation.yml` (second play) — deploys
   `foreman_scap_client` to the host.

## Corrections made to the original blueprint

Verified live against this Satellite instance's Foreman API and
`ansible_variables` endpoint:

- **`subscription_manager_registration` snippet doesn't exist.** The stock
  "Kickstart default" template registers RHEL9+ inline via `kickstart_rhsm`
  and RHEL8 in `%post` via `redhat_register`. `custom_kickstart_default.ks.erb`
  now branches on `os_major` and calls the real snippets.
- **`ansible_provisioning_callback` and `remote_execution_ssh_keys` are real**,
  stock, locked Satellite snippets — confirmed via `GET /api/provisioning_templates`.
  The host-group parameter names in `02_satellite_hostgroup_callback_prep.yml`
  (`ansible_tower_provisioning`, `ansible_tower_api_url`, `ansible_job_template_id`,
  `ansible_host_config_key`) are exactly what the real `ansible_tower_callback_script`
  snippet reads via `host_param(...)` — these were correct in the original draft.
- **`foreman_scap_client_policies` was fabricated.** Queried
  `ansible_variables` directly: the role's real default is
  `<%= @host.policies_enc %>`, an array Foreman renders from policies already
  assigned to the host — not a hand-authored list of
  `{profile_id, content_path, period, weekday}`. Those fields actually belong
  to Satellite's `POST /api/compliance/policies` resource. Policy
  creation/assignment was moved there; the role is now deployed without a
  policy override.
- **"CIS Level 1 & Level 2" only referenced one profile.**
  `xccdf_org.ssgproject.content_profile_cis` is Level 2 - Server only
  (confirmed via `GET /api/compliance/scap_contents/5`). Level 1 is the
  separate `xccdf_org.ssgproject.content_profile_cis_server_l1`. Both are now
  created as distinct policies in `group_vars/all.yml` / playbook 04.
- **Broken shell arithmetic in `dynamic_lvm_partitioning.erb`**:
  `SWAP_MEMORY=\(((\)MEMORY * 2))` is invalid syntax; the stray `\$` escapes
  elsewhere would have prevented variable interpolation inside the `cat <<EOF`
  heredoc, so `--size=$SWAP_MEMORY` would have rendered as the literal string
  `$SWAP_MEMORY` in the kickstart rather than a number. Fixed to plain
  `$((...))` / `$VAR`. The `#Dynamic` marker was also dropped — that marker
  only has meaning for a Foreman "Partition Table" resource, not a `kind:
  snippet` invoked from a hand-written `%pre` block, which is how this repo
  uses it.
- **`pxe_loader: "Grub2 UEFI HTTP"` contradicted the PXE-less requirement.**
  Removed from the hostgroup — it's an HTTP Boot (network-boot) setting,
  irrelevant to provisioning via a mounted boot-disk ISO over iDRAC virtual
  media.
- **`host_registration_insights` is a real parameter but only acts on
  `os_major < 9`** in the stock kickstart template (verified in the template
  source) — a no-op for this RHEL9 host group. Added an explicit
  `insights-client --register` fallback to
  `03_aap_day1_post_install_hardening.yml`.
- **`vault/vault.yml` held plaintext credentials** despite the name. Replaced
  with `vault/vault.yml.example`; `.gitignore` blocks the real `vault.yml` by
  name. Encrypt it with `ansible-vault encrypt` before use.

## Boot disk ISO (PXE-less provisioning)

```bash
hammer host create \
  --name "baremetal-node01.example.com" \
  --organization "Default Organization" \
  --location "Default Location" \
  --hostgroup "HG_RHEL9_SysAdmin_Prod" \
  --mac "52:54:00:12:34:56" \
  --ip "192.168.10.50" \
  --build true \
  --managed true

hammer bootdisk host --full true --host "baremetal-node01.example.com" --file /tmp/node01.iso
```

Mount `/tmp/node01.iso` via the Dell iDRAC Virtual Media console and boot.
Anaconda pulls installation media over HTTPS, runs the dynamic partitioning
`%pre`, registers via subscription-manager, and triggers the AAP provisioning
callback — no PXE/DHCP dependency. Verify hammer subcommand flags against
`hammer bootdisk host --help` on your Satellite version before running live;
this wasn't independently re-verified against the API in this review.

## Setup

```bash
ansible-galaxy collection install -r collections/requirements.yml
cp vault/vault.yml.example vault/vault.yml
# edit vault/vault.yml with real values, then:
ansible-vault encrypt vault/vault.yml
```
