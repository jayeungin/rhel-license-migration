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