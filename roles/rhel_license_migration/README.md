# RHEL License Conversion: License-Included (LI) to BYOL (AWS)

## Executive Summary
This Ansible role automates the large-scale conversion of RHEL EC2 instances from AWS License-Included (LI) billing to Bring Your Own License (BYOL). Because AWS does not support an in-place conversion for Linux, this role utilizes a **"Snapshot & Swap"** strategy to move the billing identity while preserving networking (Private IPs), storage (Data Volumes), and application state. This role is designed for a phased large-scale migration. Instead of migrating an entire AWS region, it uses **Tag-Based Filtering** to target specific "Migration Groups" (e.g., Alpha, Beta, Prod-Wave-1).

---

## 📂 Directory Structure
```
rhel_license_migration/
├── defaults/
│   └── main.yml           # Variable definitions & Toggles
├── inventories/
│   ├── aws_ec2.yml        # AWS dynamic inventory plugin config
├── tasks/
│   ├── main.yml           # Orchestration entry point
│   ├── 01_preflight.yml   # Snapshotting & State Capture
│   ├── 02_infra_swap.yml  # Termination & BYOL Relaunch
│   ├── 03_provision.yml   # Storage Restore & RHSM Registration
│   ├── 04_validation.yml  # Health Checks (Web/DB)
│   └── 05_cleanup.yml     # Snapshot Deletion
└── README.md              # Technical Documentation
```

## 🛠 Technical Workflow
### Phase 0: Identify Your Group
Ensure your target EC2 instances in AWS are tagged appropriately:
 * Key: MigrationGroup
 * Value: Alpha
### Phase 1: Pre-flight & Backup
 * State Capture: Records Private IP, Subnet, Security Groups, and IAM Profiles.
 * Storage Mapping: Scans /etc/fstab to preserve UUID-based mount points.
 * Snapshots: Creates EBS snapshots of all attached volumes (Root + Data) as a recovery point.
### Phase 2: Infrastructure Swap
 * Decommission: Terminates the original LI instance to release the Private IP.
 * Provisioning: Launches a new instance from a RHEL BYOL AMI.
 * Network Mapping: Explicitly re-assigns the original Private IP to the new instance.
 * Volume Restoration: Re-attaches data volumes created from the pre-flight snapshots.
### Phase 3: Provisioning & Entitlement
 * Storage Re-mount: Restores the fstab entries and mounts secondary volumes.
 * Licensing: Unregisters the AWS RHUI client and registers the system with the Red Hat Subscription Manager (RHSM).
### Phase 4: Validation & Health Checks
 * Verifies that Web (Nginx/Apache) and Database (Postgres/MySQL) ports are listening.
 * Confirms HTTP 200 responses and DB connectivity.

## ⚙️ Configuration (defaults/main.yml)
Update these variables before execution. Use Ansible Vault for sensitive credentials.
| Variable | Description |
|---|---|
| target_region | The AWS region where the fleet resides. |
| byol_ami_id | The AMI ID of your RHEL BYOL "Golden Image". |
| rh_username | Red Hat Customer Portal username. |
| rh_password | Red Hat Customer Portal password. |
| run_cleanup | Boolean. Set to true only to delete safety snapshots. |

## 🚀 Execution Guide
### 1. Standard Migration
Run the playbook against your dynamic inventory. It is recommended to use --limit for initial batches.
ansible-playbook -i aws_ec2.yml migrate.yml --limit <tag_or_id>

### 2. Emergency Rollback
If validation fails, use the standalone rollback playbook to restore the original LI instance from the snapshots.
ansible-playbook -i aws_ec2.yml rollback.yml --limit <instance_id>

### 3. Maintenance Cleanup
Once the fleet is confirmed stable (e.g., after 72 hours), remove the migration snapshots to optimize EBS costs.
ansible-playbook -i aws_ec2.yml migrate.yml -e "run_cleanup=true"

## ⚠️ Impact & Risk Assessment
 * Downtime: Mandatory outage of ~5–10 minutes during the IP handover window.
 * Root Volume: This process replaces the root volume. Ensure custom OS-level configs are either in your BYOL AMI or managed via Ansible post-tasks.
 * Storage Costs: You will incur temporary EBS snapshot costs until the cleanup task is executed.

## 📝 Rollback Procedure
In the event of a critical failure:
 * The failed BYOL instance is terminated.
 * A new instance is launched using the Original Root Snapshot as the image.
 * Original data snapshots are converted back to volumes and re-attached.
 * The system returns to the original AWS "License-Included" billing state.