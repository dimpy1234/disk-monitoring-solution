# AWS Scalable Disk Monitoring Solution

Centralized disk utilization monitoring across multiple AWS accounts using **CloudWatch Agent**, **AWS Systems Manager (SSM)**, and **Ansible**.

---

## Problem

A large enterprise managing multiple AWS accounts (from acquisitions) needs to:

- Detect low disk space early across all EC2 instances
- Centralize visibility without managing SSH keys or bastion hosts
- Auto-enroll new instances as the fleet grows
- Alert the right team based on instance criticality

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           AWS Organizations                                   │
│                                                                               │
│  ┌─────────────────────┐   ┌─────────────────────┐   ┌──────────────────┐   │
│  │  Monitoring Account │   │ Workload Account A  │   │ Workload Account B│  │
│  │  (Central Hub)      │   │                     │   │                   │  │
│  │                     │   │  ┌───────────────┐  │   │ ┌─────────────┐  │  │
│  │  ┌───────────────┐  │   │  │ EC2 Instances │  │   │ │EC2 Instances│  │  │
│  │  │ CloudWatch    │◄─┼───┼──│ + CW Agent    │  │   │ │+ CW Agent   │  │  │
│  │  │ OAM Sink      │◄─┼───┼──│ + SSM Agent   │  │   │ │+ SSM Agent  │  │  │
│  │  └──────┬────────┘  │   │  └───────────────┘  │   │ └─────────────┘  │  │
│  │         │            │   │                     │   │                   │  │
│  │  ┌──────▼────────┐  │   └─────────────────────┘   └──────────────────┘  │
│  │  │ CloudWatch    │  │                                                     │
│  │  │ Alarms        │  │   ┌─────────────────────────────────────────────┐  │
│  │  └──────┬────────┘  │   │        Management Account                   │  │
│  │         │            │   │  ┌──────────────────────────────────────┐   │  │
│  │  ┌──────▼────────┐  │   │  │ Ansible Control Node                 │   │  │
│  │  │ SNS → Slack / │  │   │  │ - Dynamic Inventory (aws_ec2 plugin) │   │  │
│  │  │ PagerDuty /   │  │   │  │ - Deploys CW Agent via SSM           │   │  │
│  │  │ Email         │  │   │  │ - Cross-account AssumeRole            │   │  │
│  │  └───────────────┘  │   │  └──────────────────────────────────────┘   │  │
│  └─────────────────────┘   └─────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘

```

---

## Service Selection

For each requirement, we evaluated multiple options. Here's what we chose and why:

### 1. Access & Fleet Management

| Option | Verdict |
| --- | --- |
| **AWS Systems Manager (SSM)** ✅ | No SSH keys, no open ports, pre-installed on standard AMIs, IAM-based cross-account access |
| SSH via Bastion Hosts | Key rotation overhead, open port 22, bastion HA/patching burden |
| EC2 Instance Connect | Linux-only, interactive sessions only, not suited for automation |
| HashiCorp Boundary | Extra infra to manage, licensing, not native to AWS |

**Why SSM**: Eliminates SSH key management. The Ansible `aws_ssm` connection plugin runs playbooks through SSM without any network changes. Full CloudTrail audit trail included.

### 2. Metric Collection

| Option | Verdict |
| --- | --- |
| **CloudWatch Agent** ✅ | Native, pushes to CloudWatch directly, Linux + Windows, no license cost |
| Prometheus Node Exporter | Needs a Prometheus server, pull-based (network access to each instance) |
| Datadog Agent | $15-23/host/month, data leaves AWS, vendor lock-in |
| collectd + CW plugin | Extra layer, plugin maintenance lag |
| Custom cron + `put-metric-data` | Fragile, no retry/buffering, doesn't scale |

**Why CloudWatch Agent**: Zero middleware. Metrics land directly in CloudWatch for alarms, dashboards, and cross-account visibility. AWS maintains it. $0.30/metric/month is the only cost.

### 3. Metric Aggregation (Centralization)

| Option | Verdict |
| --- | --- |
| **CloudWatch Cross-Account Observability (OAM)** ✅ | No data duplication, deploy via StackSets, real-time queries |
| Metric Streams → S3/Firehose | Extra infra, latency, complex alerting setup |
| Amazon Managed Grafana | Great dashboards but doesn't solve alarm centralization |
| Lambda-based metric forwarding | Custom code, doubles storage cost |
| Third-party TSDB (InfluxDB, Thanos) | Major operational overhead |

**Why OAM**: Metrics stay in source accounts but the monitoring account queries them as if local. Single StackSet deployment. Zero data duplication = predictable costs.

### 4. Configuration Management

| Option | Verdict |
| --- | --- |
| **Ansible** ✅ | Already used by the company, agentless, SSM plugin, dynamic inventory |
| SSM State Manager (alone) | Scales well but limited templating/logic |
| Puppet/Chef | Needs agent + server infrastructure + licensing |
| Terraform | Not designed for runtime config management |

**Why Ansible**: Already adopted. Connects via SSM (no SSH). Dynamic inventory discovers instances by tag. Human-readable YAML. At 10k+ scale, we supplement with SSM State Manager for initial install.

### 5. Alerting & Notification

| Option | Verdict |
| --- | --- |
| **CloudWatch Alarms + SNS** ✅ | Native threshold evaluation, composite alarms, serverless |
| EventBridge + Lambda | Reinvents what Alarms already does |
| Prometheus Alertmanager | Needs Prometheus stack |
| PagerDuty (standalone) | Still needs a signal source (use as SNS subscriber instead) |

**Why CloudWatch Alarms + SNS**: Evaluates metrics directly. SNS fans out to Slack, PagerDuty, email. Composite alarms support rollup logic. No extra infrastructure.

---

## Key Components

### Access Management

- AWS Organizations with Workload OUs for acquired accounts
- Cross-account IAM roles scoped to Organization ID (`aws:PrincipalOrgID` condition)
- EC2 Instance Profiles with `CloudWatchAgentServerPolicy` + `AmazonSSMManagedInstanceCore`
- No SSH keys, no bastion hosts, no open inbound ports

### VM Discovery & Enrollment

```
New EC2 Instance Launched
    → EventBridge catches RunInstances API call
    → Lambda applies default monitoring tags
    → SSM Agent auto-registers
    → Next Ansible run picks it up via dynamic inventory
    → CW Agent deployed + alarms created
    → Metrics flowing within 5 minutes

```

New accounts: deploy baseline IAM/OAM via CloudFormation StackSets, then run the Ansible playbook.

### Data Collection

- CloudWatch Agent collects: `disk_used_percent`, `disk_free`, `disk_inodes_used`
- Collection interval: 60 seconds
- Mount points configurable per-instance via `monitoring:mounts` tag
- Centralized via OAM (monitoring account sees all accounts, no data movement)

### Scalability

| Fleet Size | Approach |
| --- | --- |
| < 100 instances | Ansible alone (forks=20) |
| 100-1,000 | Ansible with rolling batches |
| 1,000-10,000 | SSM State Manager for install, Ansible for config |
| 10,000+ | SSM State Manager + StackSets, Ansible for drift checks |

### Alerting (Three Tiers)

| Tier | Warning | Critical | Routing |
| --- | --- | --- | --- |
| Critical (prod DBs) | 70% | 85% | Page on-call + auto-remediate |
| Standard (app servers) | 80% | 90% | Slack team channel |
| Low (dev/test) | 90% | 95% | Email digest |

Tier assigned by `monitoring:tier` tag on each instance.

---

## Cost Estimate

| Component | Cost |
| --- | --- |
| CloudWatch custom metrics | ~$0.30/metric/month per instance |
| CloudWatch Alarms | ~$0.10/alarm/month |
| SSM | Free (included with EC2) |
| SNS notifications | Negligible (~$0.50/million) |
| **Total per instance** | **~$3-5/month** |

---

## Quick Start

```bash
# Install Ansible prerequisites
ansible-galaxy collection install amazon.aws community.general
pip install boto3

# Dry run
cd ansible/
ansible-playbook playbooks/deploy_monitoring.yml --check

# Deploy to all monitored instances
ansible-playbook playbooks/deploy_monitoring.yml

# Ad-hoc disk check across fleet
ansible-playbook playbooks/check_disk_space.yml

```

---

## Repository Structure

```
.
├── README.md                          ← This file (full solution documentation)
├── ansible/
│   ├── ansible.cfg
│   ├── inventory/aws_ec2.yml          ← Dynamic cross-account EC2 inventory
│   ├── group_vars/all.yml             ← Variables (thresholds, SNS topics, config)
│   ├── playbooks/
│   │   ├── deploy_monitoring.yml      ← Main deployment playbook
│   │   └── check_disk_space.yml       ← Ad-hoc fleet disk check
│   └── roles/
│       ├── cloudwatch_agent/          ← Install + configure CW Agent
│       └── ssm_setup/                 ← Install + configure SSM Agent
└── cloudformation/
    └── workload-account-baseline.yml  ← StackSet for new account onboarding

```

