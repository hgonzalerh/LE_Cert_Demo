# LE_Cert_Demo
A demo for certificate maintenance using Let's Encrypt

## Playbooks

| Playbook | What it does |
|---|---|
| `apache_installed.yml` | Plain HTTP demo site (`apache_site` role). |
| `apache_tls_bootstrap.yml` | Prepares the TLS endpoint and, if no cert exists yet, installs a short-lived self-signed one. |
| `tls_renew.yml` | Audits every host's live cert on :443; for non-compliant hosts, gets a new one from Let's Encrypt (http-01) and installs it. |

Every variable has a default in its role's `defaults/main.yml`; override with `-e`.

`tls_renew.yml` additionally **requires**:

- `tls_min_days_remaining`: renew when the cert expires within this many days
  (no default; e.g. `30` for 90-day certs, `2` for ~6-day `shortlived` certs).
- `acme_account_key`: the Let's Encrypt
account private key as multiline PEM text, passed as an extra var. The easiest
way is a vars file (keep it out of git, or encrypt it with ansible-vault):

```yaml
# acme_account_key.yml
acme_account_key: |
  -----BEGIN PRIVATE KEY-----
  ...
  -----END PRIVATE KEY-----
```

```sh
ansible-navigator run tls_renew.yml -e @acme_account_key.yml -e tls_min_days_remaining=30
```

## Roles

| Role | Scope | Output facts |
|---|---|---|
| `tls_audit` | Checks the live endpoint from the controller. | `tls_compliant` |
| `acme_cert` | Let's Encrypt account (key from `acme_account_key`), domain key, CSR, order (`main`) and retrieval (`finalize`), all on the controller. | `cert_challenge`, `cert_files` |
| `selfsigned_cert` | Self-signed key + cert on the controller. | `cert_files` |
| `cert_target_apache` | Everything Apache-specific. | `cert_target_has_cert` |
| `apache_site` | Plain HTTP demo site. | — |

## Installing certs on something other than Apache

The server software is selected with `cert_target` (default `apache`), which
picks the role `cert_target_<cert_target>`:

```sh
ansible-playbook -i hosts tls_renew.yml -e cert_target=nginx
```

To add a target, create `roles/cert_target_<name>/` with these task files:

| Task file | Contract |
|---|---|
| `prepare.yml` | Make the host a TLS endpoint. Set fact `cert_target_has_cert` (bool). |
| `publish_challenge.yml` | Serve `cert_challenge.resource_value` at `http://<fqdn>/<cert_challenge.resource>`. Do nothing if `cert_challenge` is empty. |
| `cleanup_challenge.yml` | Remove it. Do nothing if `cert_challenge` is empty. |
| `install.yml` | Install `cert_files.key`, `.fullchain` and optionally `.chain` (controller-side paths) and reload the service. |
| `verify.yml` | Make sure the service is running with a valid config. |

`cert_target_apache` is the reference implementation.

## CI

`.github/workflows/ansible-lint.yml` runs `ansible-lint` on every pull request
(and on pushes to `main`) using the public
`ghcr.io/ansible/community-ansible-dev-tools` image. The `ansible-lint` check
must pass before a pull request can be merged into `main`.

## Running with ansible-navigator

`ansible-navigator.yml` runs everything in the AAP 2.7 supported EE
(`registry.redhat.io/ansible-automation-platform-27/ee-supported-rhel9`),
pulled only if missing. One-time setup:

```sh
podman login registry.redhat.io
ansible-galaxy collection install -r collections/requirements.yml -p collections
```

The EE doesn't ship `community.crypto`; the second command installs it into
`collections/` (gitignored), which is picked up automatically inside the EE.
Run artifacts and logs go to `.navigator/` (gitignored).

## Test infrastructure on DigitalOcean

`test_infra_create.yml` / `test_infra_delete.yml` build and tear down
throwaway droplets to run the playbooks against. Settings live in
`test_infra_vars.yml`.

Droplets use the custom RHEL 9 image in nyc3 at the smallest size there,
get a 2 GB swap file, and are registered with Red Hat by the
`redhat.rhel_system_roles.rhc` role. `test_infra_delete.yml` unregisters them
before deleting them.

One-time setup:
1. Put your DigitalOcean API token in `.do_api_token` (gitignored).
1. Put your Red Hat account in `rh_credentials.yml` (gitignored):
   ```yaml
   rhc_auth:
     login:
       username: "you@example.com"
       password: "..."
   ```
2. At your registrar, delegate the test zone to DigitalOcean: NS records
   for `test` → `ns1.digitalocean.com`, `ns2.digitalocean.com`, `ns3.digitalocean.com`.

```sh
ansible-navigator run test_infra_create.yml          # droplets + DNS + ./test_infra_hosts
ansible-navigator run apache_tls_bootstrap.yml -i test_infra_hosts
ansible-navigator run tls_renew.yml -i test_infra_hosts -e @acme_account_key.yml -e tls_min_days_remaining=2 -e tls_skip_chain_check=true
ansible-navigator run test_infra_delete.yml          # only removes droplets tagged le-cert-demo
```
