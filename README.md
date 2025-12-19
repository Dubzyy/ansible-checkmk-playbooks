# Ansible CheckMK Playbooks

Fully automated CheckMK deployment and configuration using Ansible.

## Features

- 🚀 **Automated CheckMK Installation** - Deploy CheckMK Raw Edition from scratch
- 🤖 **Agent Distribution** - Install and configure agents across your infrastructure
- 🔌 **REST API Integration** - Manage hosts programmatically via CheckMK's REST API
- 🔐 **Ansible Vault Support** - Secure credential management
- ♻️ **Idempotent** - Safe to run multiple times
- ✅ **Production-Ready** - Battle-tested on real infrastructure

## Prerequisites

- **Target OS:** Ubuntu 22.04 LTS
- **Ansible:** 2.9 or higher
- **Access:** Root or sudo privileges on target hosts
- **Network:** HTTP access from Ansible control node to targets

## Quick Start

### Prerequisites Checklist

Before starting, ensure you have:
- ✅ A VM or server running **Ubuntu 22.04 LTS** for the CheckMK server
- ✅ Additional VMs/servers running **Ubuntu 22.04 LTS** for monitored hosts
- ✅ **SSH access** (root or sudo user) to all hosts
- ✅ **Ansible installed** on your control node
- ✅ Network connectivity between all hosts

**Example VM Specifications for CheckMK Server:**
- **Minimum:** 2 vCPU, 4GB RAM, 20GB disk
- **Recommended:** 4 vCPU, 8GB RAM, 50GB disk (for larger deployments)
- **OS:** Ubuntu 22.04 LTS (Server)
- **Networking:** Static IP address recommended

### 1. Prepare Your Environment

**On your Ansible control node** (can be your workstation or a dedicated server):
```bash
# Install Ansible if not already installed
sudo apt update
sudo apt install ansible -y

# Clone this repository
git clone https://github.com/Dubzyy/ansible-checkmk-playbooks.git
cd ansible-checkmk-playbooks
```

### 2. Configure Inventory

Create your inventory file with your actual VM/server IP addresses:
```bash
cp examples/inventory.example inventory/hosts
nano inventory/hosts
```

Update with your hosts:
```ini
[checkmk]
checkmk-server ansible_host=10.10.1.18 ansible_user=root

[monitored_hosts]
webserver01 ansible_host=10.10.1.10 ansible_user=root
dbserver01 ansible_host=10.10.1.11 ansible_user=root
appserver01 ansible_host=10.10.1.12 ansible_user=root
```

**Test connectivity:**
```bash
ansible all -i inventory/hosts -m ping
```

### 3. Create Vault for Admin Password
```bash
mkdir -p group_vars/all
ansible-vault create group_vars/all/vault.yml
```

Add to vault (automation password comes later):
```yaml
---
vault_checkmk_admin_password: "YourSecureAdminPassword"
```

### 4. Deploy CheckMK Server

This will install CheckMK on the VM you specified in the `[checkmk]` group:
```bash
ansible-playbook checkmk-deploy.yml -i inventory/hosts --ask-vault-pass
```

**Deployment takes 5-10 minutes.** After completion:
- Access CheckMK web UI at `http://YOUR_SERVER_IP/monitoring/`
- Login with username: `cmkadmin`
- Password: `[the password you set in vault]`

### 5. Get Automation Secret and Add to Vault

After CheckMK is deployed, retrieve the automation password:
```bash
ssh root@YOUR_CHECKMK_SERVER
cat /omd/sites/monitoring/var/check_mk/web/automation/automation.secret
```

Copy the secret and add it to your vault:
```bash
ansible-vault edit group_vars/all/vault.yml
```

Add this line:
```yaml
vault_checkmk_automation_password: "paste-the-secret-here"
```

### 6. Install Agents on Monitored Hosts

This installs the CheckMK agent on all servers in your `[monitored_hosts]` group:
```bash
ansible-playbook checkmk-install-agents.yml -i inventory/hosts \
  -e checkmk_server_ip=10.10.1.18 \
  -e checkmk_site_name=monitoring \
  -e target_group=monitored_hosts
```

### 7. Add Hosts to Monitoring

Edit the API playbook to define which hosts to monitor:
```bash
nano checkmk-add-hosts-api.yml
```

Update the `monitored_hosts` list:
```yaml
monitored_hosts:
  - { name: "webserver01", ip: "10.10.1.10" }
  - { name: "dbserver01", ip: "10.10.1.11" }
  - { name: "appserver01", ip: "10.10.1.12" }
```

Run:
```bash
ansible-playbook checkmk-add-hosts-api.yml \
  -e checkmk_server_ip=10.10.1.18 \
  -e checkmk_site_name=monitoring \
  --ask-vault-pass
```

**Done!** Check your CheckMK dashboard to see all monitored hosts and services.

## Playbook Reference

### checkmk-deploy.yml

Deploys CheckMK server with complete configuration.

**Variables:**
- `checkmk_version`: CheckMK version (default: 2.2.0p17)
- `checkmk_site_name`: Site name (default: monitoring)
- `checkmk_admin_password`: Admin password from vault
- `checkmk_force_password_reset`: Force password reset (default: false)

**Example:**
```bash
ansible-playbook checkmk-deploy.yml -i inventory/hosts --ask-vault-pass
```

### checkmk-install-agents.yml

Installs CheckMK agents on target hosts.

**Required Variables:**
- `checkmk_server_ip`: IP address of CheckMK server
- `checkmk_site_name`: CheckMK site name

**Optional Variables:**
- `target_group`: Inventory group to target (default: monitored_hosts)
- `checkmk_version`: Agent version (default: 2.2.0p17)

**Example:**
```bash
ansible-playbook checkmk-install-agents.yml -i inventory/hosts \
  -e checkmk_server_ip=10.10.1.18 \
  -e checkmk_site_name=monitoring
```

### checkmk-add-hosts-api.yml

Adds hosts to CheckMK via REST API and triggers service discovery.

**Required Variables:**
- `checkmk_server_ip`: CheckMK server IP
- `checkmk_site_name`: Site name
- `vault_checkmk_automation_password`: Automation user password from vault
- `monitored_hosts`: List of hosts to add

**Example:**
```bash
ansible-playbook checkmk-add-hosts-api.yml \
  -e checkmk_server_ip=10.10.1.18 \
  -e checkmk_site_name=monitoring \
  --ask-vault-pass
```

## Getting Automation Secret

The automation user password is required for API operations:
```bash
ssh root@YOUR_CHECKMK_SERVER
cat /omd/sites/monitoring/var/check_mk/web/automation/automation.secret
```

Add this to your vault as `vault_checkmk_automation_password`.

## Advanced Usage

### Force Password Reset
```bash
ansible-playbook checkmk-deploy.yml -i inventory/hosts \
  --ask-vault-pass \
  -e checkmk_force_password_reset=true
```

### Target Specific Hosts
```bash
ansible-playbook checkmk-install-agents.yml -i inventory/hosts \
  -e checkmk_server_ip=10.10.1.18 \
  -e checkmk_site_name=monitoring \
  --limit webserver01
```

### Different Site Name
```bash
ansible-playbook checkmk-deploy.yml -i inventory/hosts \
  --ask-vault-pass \
  -e checkmk_site_name=production
```

## Troubleshooting

### Agent Not Responding

Check systemd socket status:
```bash
systemctl status check-mk-agent.socket
```

Test agent manually:
```bash
check_mk_agent
```

Verify network connectivity:
```bash
telnet CHECKMK_SERVER 6556
```

### API Authentication Fails

1. Verify automation secret:
```bash
cat /omd/sites/monitoring/var/check_mk/web/automation/automation.secret
```

2. Ensure it matches your vault
3. Check URL is correct (http://SERVER/SITE_NAME)

### Site Won't Start

Check site status:
```bash
omd status monitoring
```

View logs:
```bash
omd su monitoring
tail -f var/log/cmc.log
```

## Security Best Practices

- ✅ Always use Ansible Vault for passwords
- ✅ Never commit vault files or plaintext credentials
- ✅ Use automation user (not cmkadmin) for API operations
- ✅ Restrict CheckMK agent access via firewall
- ✅ Use SSH keys instead of passwords where possible
- ✅ Regularly rotate automation secrets

## Project Structure
```
ansible-checkmk-playbooks/
├── checkmk-deploy.yml              # Deploy CheckMK server
├── checkmk-install-agents.yml      # Install agents on hosts
├── checkmk-add-hosts-api.yml       # Add hosts via API
├── README.md                       # This file
├── LICENSE                         # MIT License
├── .gitignore                      # Git ignore rules
├── examples/
│   ├── inventory.example           # Example inventory
│   └── vault.example               # Example vault structure
├── group_vars/
│   └── all/
│       └── vault.yml              # Encrypted secrets (gitignored)
└── inventory/
    └── hosts                       # Your inventory (gitignored)
```

## Contributing

Contributions welcome! Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

Please ensure:
- Playbooks are idempotent
- Variables are well-documented
- Code follows existing style
- README is updated if needed

## License

MIT License - see [LICENSE](LICENSE) file for details.

## Author

**Hunter Wilson**  
- GitHub: [@Dubzyy](https://github.com/Dubzyy)
- Website: [vrhost.org](https://vrhost.org)

## Acknowledgments

- CheckMK community for excellent documentation
- Ansible community for best practices
- Everyone who contributed feedback and testing

## Support

- 📖 [CheckMK Documentation](https://docs.checkmk.com/)
- 🐛 [Report Issues](https://github.com/YOUR_USERNAME/ansible-checkmk-playbooks/issues)
- 💬 [Discussions](https://github.com/YOUR_USERNAME/ansible-checkmk-playbooks/discussions)

---

⭐ If you find this project useful, please consider giving it a star on GitHub!
