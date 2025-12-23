# AWS RHEL License Migration Framework (LI to BYOL)

## Overview
This project provides a a role to convert RHEL EC2 instances from **AWS License-Included (LI)** billing to **Bring Your Own License (BYOL)**. 

Because AWS billing codes are immutable on existing EBS volumes, this framework implements a **"Snapshot & Swap"** strategy. It decommissions the LI identity and launches a new BYOL identity while maintaining the **Private IP**, **Tags**, and **Data Volumes**.

---

## 📂 Architecture & Directory Structure
```text
rhel-migration-project/
├── ansible.cfg                # Project configuration (Paths & Plugins)
├── migrate.yml                # Main entry-point playbook
├── rollback.yml               # Emergency recovery playbook
├── inventories/
│   └── aws_ec2.yml            # Dynamic AWS fleet discovery plugin
├── group_vars/
│   └── all/
│       └── vault.yml          # Encrypted RH Credentials (AES-256)
└── roles/
    └── rhel_license_migration/# Modular migration logic
```

## 🔐 Security & Requirements
Ansible Vault: All Red Hat Customer Portal credentials are encrypted in group_vars/all/vault.yml.
* **[To Be Implemented] Ansible Vault:** All Red Hat Customer Portal credentials are encrypted in group_vars/all/vault.yml.
* **AWS Permissions:** The executing IAM role requires permissions for ec2:TerminateInstances, ec2:RunInstances (with IP assignment), and ec2:CreateSnapshot.
* **Network:** Instances must have egress access (via NAT or Proxy) to subscription.rhsm.redhat.com.

## 🎯 Target Group Filtering
To prevent accidental mass-migrations, this project uses Tag-Based Filtering. The playbook will only execute on instances where the tags match the values defined in roles/rhel_license_migration/defaults/main.yml.

Default Filter:
* **Key:** MigrationGroup
* **Value:** Alpha

## 🚀 Operations Guide
1. **Verification (Read-Only):** Check which instances are currently identified by the dynamic inventory:
```text
ansible-inventory --graph
```
2. **Execution (Migration Wave):** Execute the migration for the current group defined in defaults/main.yml. If you have your credentials stored in Vault, you must provide the Vault password.
```text
ansible-playbook migrate.yml
```

3. **Overriding the Migration Group:** To target a different wave (e.g., Beta) without changing the code:
```text
ansible-playbook migrate.yml -e "migration_filter_value=Beta" 
```

4. **Emergency Rollback:** If health checks fail, revert the target instance to its original state:
```text
ansible-playbook rollback.yml --limit <instance_id>
```

## 🛠 Project Lifecycle
* **Pre-Flight:** Snapshots are taken of all volumes. Original metadata (IP/fstab) is captured.
* **Swap:** Instance is terminated; New BYOL instance launched with original IP.
* **Restore:** Data volumes are re-attached and /etc/fstab is restored.
* **License:** System is registered via RHSM.
* **Validation:** Web and Database services are checked for health.
* **Cleanup:** Once verified, run with -e "run_cleanup=true" to remove safety snapshots.