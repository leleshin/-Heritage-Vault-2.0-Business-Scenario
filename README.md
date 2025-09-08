Heritage Automation — Ubuntu VM Quickstart

This guide explains how to install and run heritage_automation.sh on an Ubuntu VM to automate updates, backups, deployments (via Ansible), and monitoring.

Requirements:

Ubuntu 20.04+ (recommended 22.04+)

sudo privileges

1. Install Prerequisites
sudo apt-get update
sudo apt-get install -y ansible rsync mailutils curl


Note: Python3 is included in modern Ubuntu versions.

2. Get the Files onto the VM

Option A — Clone from GitHub

git clone https://github.com/<your-user>/<your-repo>.git
cd <your-repo>


Option B — Copy from Local Machine

scp -r heritage_automation.sh ansible notify systemd <your-user>@<vm-ip>:~/

3. Place Files in Standard Locations
# Main script
sudo install -m 0755 ./heritage_automation.sh /usr/local/sbin/

# Ansible project (inventory + playbooks)
sudo mkdir -p /opt/heritage/ansible
sudo cp -r ./ansible/* /opt/heritage/ansible/

# Notifier (optional)
sudo install -m 0755 ./notify/heritage_notify.sh /usr/local/lib/

# Inventory (default path for Ansible)
sudo mkdir -p /etc/ansible
sudo cp ./ansible/inventory/hosts.ini /etc/ansible/hosts

# Log file
sudo touch /var/log/heritage_automation.log
sudo chmod 640 /var/log/heritage_automation.log


Adjust script variables if your repo uses different paths (ANSIBLE_INVENTORY, ANSIBLE_PLAYBOOK, BACKUP_SRC, BACKUP_DEST).

4. (Optional) Add Quick Test Data
sudo mkdir -p /srv/heritage/data
echo "hello" | sudo tee /srv/heritage/data/example.txt

5. Run the Script

Safe read-only check (monitor only):

sudo heritage_automation.sh monitor


Full run (update + backup + deploy + monitor):

sudo heritage_automation.sh --all


View logs:

sudo tail -n 200 /var/log/heritage_automation.log

6. Schedule Automation

Option A — Systemd Timers (Recommended)

sudo cp ./systemd/heritage_automation.* /etc/systemd/system/
sudo cp ./systemd/heritage_monitor.* /etc/systemd/system/
sudo systemctl daemon-reload

# Enable timers
sudo systemctl enable --now heritage_automation.timer   # daily at 02:10
sudo systemctl enable --now heritage_monitor.timer      # every 5 minutes

# Verify
systemctl list-timers | grep heritage


Option B — Cron

sudo crontab -e

# Example entries:
# 10 2 * * * /usr/local/sbin/heritage_automation.sh --all
# */5 * * * * /usr/local/sbin/heritage_automation.sh monitor

7. (Optional) Configure Alerts (Email / Slack)

The script automatically loads /usr/local/lib/heritage_notify.sh if present. Configure environment variables:

echo 'EMAIL_TO=ops@example.org' | sudo tee -a /etc/environment
echo 'EMAIL_SUBJECT_PREFIX=[Heritage Automation]' | sudo tee -a /etc/environment
echo 'SLACK_WEBHOOK_URL=https://hooks.slack.com/services/XXX/YYY/ZZZ' | sudo tee -a /etc/environment

sudo systemctl daemon-reload
sudo systemctl restart heritage_automation.timer heritage_monitor.timer


Note: Fresh Ubuntu VMs may require an MTA or SMTP relay for email. You can use Slack notifications only if preferred.

8. Using the Sample Ansible Playbook

[web] hosts: installs NGINX and serves a simple page

[db] hosts: installs and enables PostgreSQL

Test locally by adding the VM to [web] in /etc/ansible/hosts. For NGINX testing with UFW:

sudo ufw allow 80/tcp

9. Troubleshooting

Permission denied: Run with sudo.

Ansible not found: sudo apt-get install -y ansible

No backups: Verify BACKUP_SRC exists and has files

Playbook errors: ansible -i /etc/ansible/hosts all -m ping to test SSH connectivity

Timers not firing: systemctl status heritage_automation.timer and journalctl -u heritage_automation.service

10. Disable / Uninstall
# Disable timers
sudo systemctl disable --now heritage_automation.timer heritage_monitor.timer

# Remove script and notifier
sudo rm -f /usr/local/sbin/heritage_automation.sh /usr/local/lib/heritage_notify.sh

# Optional: remove Ansible project and logs
# sudo rm -rf /opt/heritage/ansible /var/log/heritage_automation.log

Security Notes

Review scripts and playbooks before running

Restrict access to /usr/local/sbin/heritage_automation.sh and /var/log/heritage_automation.log

Store SSH private keys securely (e.g., root keychain with correct permissions)

Use system environment variables or systemd drop-ins for Slack/email secrets instead of storing them in the repo
