# LE_Cert_Demo
A demo for certificate maintenance using Let's Encrypt

## Playbooks

| Playbook | What it does |
|---|---|
| `apache_installed.yml` | Plain HTTP demo site (`apache_site` role). |
| `apache_tls_bootstrap.yml` | Prepares the TLS endpoint and, if no cert exists yet, installs a short-lived self-signed one. |
| `tls_renew.yml` | Audits every host's live cert on :443; for non-compliant hosts, gets a new one from Let's Encrypt (http-01) and installs it. |

Every variable has a default in its role's `defaults/main.yml`; override with `-e`.

## Roles

| Role | Scope | Output facts |
|---|---|---|
| `tls_audit` | Checks the live endpoint from the controller. | `tls_compliant` |
| `acme_cert` | Let's Encrypt account, key, CSR, order (`main`) and retrieval (`finalize`), all on the controller. | `cert_challenge`, `cert_files` |
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

## Test infrastructure on DigitalOcean

`test_infra_create.yml` / `test_infra_delete.yml` build and tear down
throwaway droplets to run the playbooks against. Settings live in
`test_infra_vars.yml`.

One-time setup:
1. Put your DigitalOcean API token in `.do_api_token` (gitignored).
2. At your registrar, delegate the test zone to DigitalOcean: NS records
   for `test` → `ns1.digitalocean.com`, `ns2.digitalocean.com`, `ns3.digitalocean.com`.

```sh
ansible-playbook test_infra_create.yml          # droplets + DNS + ./test_infra_hosts
ansible-playbook -i test_infra_hosts apache_tls_bootstrap.yml
ansible-playbook -i test_infra_hosts tls_renew.yml -e tls_skip_chain_check=true
ansible-playbook test_infra_delete.yml          # only removes droplets tagged le-cert-demo
```
