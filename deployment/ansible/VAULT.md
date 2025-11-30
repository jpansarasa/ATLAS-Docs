# Ansible Vault

Sensitive credentials encrypted in `group_vars/vault.yml`. Vault password: `~/.ansible_vault_pass` (configured in `ansible.cfg`).

## Encrypted Variables

### Database
- `postgres_password` - PostgreSQL admin password
- `atlas_db_password` - ATLAS services database password

### API Keys
- `fred_api_key` - FRED API key (FredCollector)
- `alphavantage_api_key` - Alpha Vantage API key (AlphaVantageCollector)
- `finnhub_api_key` - Finnhub API key (FinnhubCollector)
- `nasdaq_api_key` - Nasdaq Data Link API key (NasdaqCollector)

### Notifications (Optional)
- `ntfy_password` - Ntfy push notification password
- `email_password` - SMTP email password

## Commands

```bash
# View encrypted values
ansible-vault view group_vars/vault.yml

# Edit encrypted values
ansible-vault edit group_vars/vault.yml

# Deploy (vault password auto-loaded from ansible.cfg)
ansible-playbook playbooks/site.yml

# Generate strong password/key
openssl rand -base64 32
```

## Adding New Secrets

```bash
# 1. Generate key
NEW_KEY=$(openssl rand -base64 32)
echo $NEW_KEY

# 2. Edit vault
ansible-vault edit group_vars/vault.yml

# 3. Add variable (e.g., new_api_key: <paste key>)

# 4. Reference in compose.yaml.j2
# - NEW_SERVICE_KEY={{ new_api_key }}
```

## Security

- Keep `~/.ansible_vault_pass` with `chmod 600`
- Never commit vault password file
- Commit `vault.yml` (it's encrypted)
- Use strong randomly-generated passwords
