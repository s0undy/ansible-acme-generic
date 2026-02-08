# ansible-acme-automation

Ansible role for SSL certificate generation via ACME enrollment. Currently supports Let's Encrypt as ACME and Cloudflare as DNS provider.

## Usage

```bash
cp example-vars.yaml vars.yaml
ansible-vault create vault.yaml
ansible-playbook playbook.yaml --ask-vault-pass
```

Put secrets in `vault.yaml`:

```yaml
letsencrypt_cloudflare_api_token: "your-token"
letsencrypt_cloudflare_zone: "example.com"
```

Playbook:

```yaml
- hosts: localhost
  connection: local
  gather_facts: true
  become: false
  vars_files:
    - vars.yaml
    - vault.yaml
  roles:
    - role: letsencrypt
```

## Required variables

| Variable | Description |
|---|---|
| `letsencrypt_domain` | Primary domain |
| `letsencrypt_account_email` | Account email |
| `letsencrypt_cloudflare_api_token` | Cloudflare API token (when using Cloudflare) |
| `letsencrypt_cloudflare_zone` | Cloudflare DNS zone (when using Cloudflare) |

## Optional variables

| Variable | Default | Description |
|---|---|---|
| `letsencrypt_environment` | `staging` | `staging` or `production` |
| `letsencrypt_san_domains` | `[]` | Additional domains or wildcards |
| `letsencrypt_cert_name` | `default` | Certificate directory name |
| `letsencrypt_certs_base_dir` | `{{ playbook_dir }}/certs` | Output directory |
| `letsencrypt_private_key_type` | `ECC` | `RSA` or `ECC` |
| `letsencrypt_cert_ecc_curve` | `secp384r1` | ECC curve for certificate key |
| `letsencrypt_cert_rsa_key_size` | `4096` | RSA size for certificate key |
| `letsencrypt_account_key_type` | `ECC` | `RSA` or `ECC` |
| `letsencrypt_account_ecc_curve` | `secp384r1` | ECC curve for account key |
| `letsencrypt_account_rsa_key_size` | `4096` | RSA size for account key |
| `letsencrypt_account_key_path` | auto | Override account key path |
| `letsencrypt_renewal_days` | `30` | Days before expiry to renew |
| `letsencrypt_dns_propagation_wait` | `60` | Seconds to wait for DNS |
| `letsencrypt_pfx_export` | `true` | Export PFX |
| `letsencrypt_pfx_legacy_export` | `true` | Export legacy PFX |
| `letsencrypt_debug` | `false` | Verbose output |

See [example-vars.yaml](example-vars.yaml) for the full list with descriptions.

## Output

Certificates are written to `certs/{domain}/{name}/`:

```
cert.pem              Server certificate
chain.pem             Intermediate chain
fullchain.pem         Certificate + chain
privatekey.pem        Private key (encrypted)
passphrase.txt        Key passphrase
certificate.pfx       PKCS#12
certificate-legacy.pfx   PKCS#12 (legacy)
```

## DNS providers

### Cloudflare (built-in)

Used by default. Requires `letsencrypt_cloudflare_api_token` and `letsencrypt_cloudflare_zone`.

### Custom provider

Set `letsencrypt_dns_provider: custom` and point `letsencrypt_dns_provider_tasks_dir` to a directory containing `create.yaml` and `cleanup.yaml`.

```yaml
letsencrypt_dns_provider: custom
letsencrypt_dns_provider_tasks_dir: "{{ playbook_dir }}/my_provider"
```

`create.yaml` receives `_letsencrypt_acme_challenge`, `_letsencrypt_all_domains`, and `letsencrypt_challenge_type`. It must register `_letsencrypt_dns_records_created`.

`cleanup.yaml` receives the same variables. Use `failed_when: false` so cleanup never breaks the run.

Example:

```yaml
# my_provider/create.yaml
- name: Create TXT records
  my_dns_module:
    record: "{{ _letsencrypt_acme_challenge['challenge_data'][item][letsencrypt_challenge_type].record }}"
    type: TXT
    value: "{{ _letsencrypt_acme_challenge['challenge_data'][item][letsencrypt_challenge_type].resource_value }}"
    state: present
  loop: "{{ _letsencrypt_all_domains }}"
  register: _letsencrypt_dns_records_created
  when: item in _letsencrypt_acme_challenge.get('challenge_data', {})
```

## Requirements

- `community.crypto` collection
- `community.general` collection

## License

AGPL-3.0
