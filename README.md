# Shelly Pro 3EM energy monitoring with Ansible

An idempotent Ansible deployment for Fedora/RHEL that installs:

- Shelly Pro 3EM exporter (local polling)
- Prometheus with alert rules and 365-day retention
- Grafana with a provisioned datasource and dashboard
- nginx HTTPS reverse proxy
- Podman Quadlet system services
- firewalld and SELinux policy configuration

The repository contains no passwords, certificates, or private keys.

## Requirements

- Ansible Core 2.16 or newer on the controller
- SSH access with sudo to a Fedora/RHEL server
- DNS for the Grafana hostname
- A matching TLS certificate and private key

Install the required collection:

    ansible-galaxy collection install -r requirements.yml

## Configure

    cp inventory/hosts.example.yml inventory/hosts.yml
    cp /secure/path/grafana.crt files/tls/grafana.crt
    cp /secure/path/grafana.key files/tls/grafana.key

Edit `inventory/hosts.yml` and `inventory/group_vars/all/main.yml` as needed. Then create encrypted secrets:

    cat > /tmp/vault.yml <<'EOF'
    energy_monitoring_grafana_admin_password: "replace-with-a-strong-password"
    energy_monitoring_shelly_password: ""
    EOF
    ansible-vault encrypt /tmp/vault.yml --output inventory/group_vars/all/vault.yml
    rm -f /tmp/vault.yml

The real inventory, vault file, certificate, and private key are ignored by Git.

## Deploy

    ansible-playbook -i inventory/hosts.yml site.yml --ask-vault-pass

The playbook validates the exporter, Prometheus, and Grafana endpoints and fails if a Prometheus scrape target is down.

## Verify

    curl --cacert /path/to/ca.crt https://grafana.example.com/api/health
    ssh deployer@grafana.example.com 'systemctl is-active shelly-exporter prometheus grafana nginx'

No Shelly push endpoint, MQTT setting, or callback is required. The exporter polls the configured Shelly address every 10 seconds by default.

## Security and reproducibility

Never commit TLS private keys or plaintext passwords. Install a private CA on clients if one is used. For strict reproducibility, replace the configurable `latest` container tags with tested immutable tags or digests.
