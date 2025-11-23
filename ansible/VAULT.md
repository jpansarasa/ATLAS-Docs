# Ansible Vault

Sensitive credentials encrypted in `group_vars/vault.yml`. Vault password: `~/.ansible_vault_pass` (configured in `ansible.cfg`).

## Encrypted Variables

- `atlas_db_password` - FredCollector database
- `postgres_password` - PostgreSQL admin
- `fred_api_key` - FRED API key
- `api_key` - REST API authentication
- `api_key_enabled` - Enable REST API auth

## Common Commands

```bash
# View encrypted values
ansible-vault view group_vars/vault.yml

# Edit encrypted values
ansible-vault edit group_vars/vault.yml

# Deploy (vault password auto-loaded from ansible.cfg)
ansible-playbook playbooks/infrastructure.yml

# Generate strong password
openssl rand -base64 32
```

## Production Setup

```bash
# 1. Generate passwords
ATLAS_PASS=$(openssl rand -base64 32)
POSTGRES_PASS=$(openssl rand -base64 32)
API_KEY=$(openssl rand -base64 32)

# 2. Edit vault
ansible-vault edit group_vars/vault.yml
# Update: atlas_db_password, postgres_password, fred_api_key, api_key

# 3. Deploy
ansible-playbook playbooks/infrastructure.yml
```

## Security

✅ DO: Keep `~/.ansible_vault_pass` with `chmod 600`, use strong passwords, commit `vault.yml` (encrypted)  
❌ DON'T: Commit password file, use dev passwords in production

