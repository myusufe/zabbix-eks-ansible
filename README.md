# Zabbix on AWS EKS with MySQL, HAProxy & Grafana  
### Automated Deployment Using Ansible Roles + Optional Terraform Provisioning

**GitHub Description:**  
Deploy a highly available Zabbix monitoring platform on AWS EKS using modular Ansible roles. Includes MySQL StatefulSet with HAProxy, Zabbix Server, Zabbix Web, external HAProxy load balancing, Grafana with Zabbix plugin, dashboard provisioning, and optional Terraform automation for EKS cluster creation.

**GitHub Tags:**  
`zabbix` `grafana` `kubernetes` `eks` `aws` `ansible` `terraform` `haproxy` `monitoring` `observability` `mysql` `automation` `devops` `cloud` `infrastructure-as-code`

---

## 🏗 Project Overview

This repository provides a complete monitoring stack running on **AWS EKS**, managed with **Ansible roles**, with optional **Terraform** to provision the EKS cluster and network.

### Components

- **Zabbix Server** (Kubernetes Deployment)
- **Zabbix Web UI** (Nginx-based, Kubernetes Deployment)
- **MySQL** (Kubernetes StatefulSet)
- **Internal HAProxy** in front of MySQL
- **External HAProxy** (LoadBalancer Service) for Zabbix
- **Grafana** with:
  - Zabbix plugin preinstalled
  - Auto-provisioned Zabbix datasource
  - Starter dashboard for Zabbix, MySQL, and HAProxy metrics

---

## 📊 Architecture Diagram (ASCII)

```text
                        ┌──────────────────────────────┐
                        │          AWS EKS             │
                        │      (Kubernetes Cluster)    │
                        └──────────────┬───────────────┘
                                       │
                    ┌──────────────────┴──────────────────┐
                    │                                     │
         ┌──────────▼──────────┐              ┌───────────▼───────────┐
         │     MySQL Pod       │              │   Zabbix Server Pods  │
         │  StatefulSet (DB)   │              │ (Multiple Replicas)   │
         └──────────┬──────────┘              └───────────┬───────────┘
                    │                                     │
         ┌──────────▼──────────┐              ┌───────────▼───────────┐
         │   HAProxy (DB)      │              │   Zabbix Web UI Pods  │
         │ Internal LB :3307   │              │ (Multiple Replicas)   │
         └──────────┬──────────┘              └───────────┬───────────┘
                    │                                     │
                    └───────────────┬─────────────────────┘
                                    │
                     ┌──────────────▼───────────────┐
                     │   External HAProxy (LB)       │
                     │  Port 80 → Zabbix Web         │
                     │  Port 10051 → Zabbix Server   │
                     └──────────────┬───────────────┘
                                    │
                     ┌──────────────▼───────────────┐
                     │           Grafana             │
                     │ Zabbix Datasource + Dashboard │
                     └───────────────────────────────┘
```

---

## 📌 Prerequisites

### AWS

- AWS account
- EKS cluster (created manually or via Terraform in `terraform/`)
- Worker node group (e.g. `t3.micro` for testing)
- IAM permissions for:
  - EKS, EC2, ELB/NLB
  - IAM roles for nodes

### Local Machine (Mac / Windows / WSL)

- Python 3.8+
- Ansible
- `kubernetes.core` collection
- `kubectl` configured for EKS

Install requirements:

```bash
pip install ansible kubernetes
ansible-galaxy collection install kubernetes.core
```

Verify EKS connection:

```bash
kubectl get nodes
```

---

## ⚙️ Configuration

Environments are separated into inventories:

```text
inventories/
├─ dev/
│  ├─ hosts
│  └─ group_vars/all.yml
├─ stage/
│  ├─ hosts
│  └─ group_vars/all.yml
└─ prod/
   ├─ hosts
   └─ group_vars/all.yml
```

Example: `inventories/dev/group_vars/all.yml`:

```yaml
kubeconfig_path: "{{ lookup('env', 'KUBECONFIG') | default('~/.kube/config', true) }}"
kube_context: ""                 # leave empty to use current context
zabbix_namespace: "zabbix-dev"

# DB credentials (change in real use)
db_root_password: "DevRootPass123!"
db_zabbix_password: "DevZabbixPass123!"
db_user: "zabbix"
db_name: "zabbix"

# Images
zabbix_server_image: "zabbix/zabbix-server-mysql:alpine-7.4-latest"
zabbix_web_image: "zabbix/zabbix-web-nginx-mysql:alpine-7.4-latest"
mysql_image: "mysql:8.0"
haproxy_image: "haproxy:2.9"

# MySQL
mysql_replicas: 1
mysql_storage_size: "10Gi"
mysql_service_port: 3306
mysql_haproxy_port: 3307

# Zabbix replicas
zabbix_server_replicas: 2
zabbix_web_replicas: 2

# Grafana
grafana_image: "grafana/grafana:10.4.0"
grafana_admin_user: "admin"
grafana_admin_password: "DevGrafanaPass123!"
grafana_service_type: "LoadBalancer"
```

Stage/prod override passwords, namespace, replicas, and storage.

---

## 🧩 How It Works (Roles)

Project structure:

```text
.
├─ ansible.cfg
├─ deploy-zabbix-eks.yml
├─ inventories/
├─ roles/
│  ├─ mysql/
│  ├─ zabbix/
│  ├─ haproxy/
│  └─ grafana/
├─ .github/workflows/deploy.yml
└─ terraform/
```

### Role: `mysql`

- Creates `mysql-secret` with DB credentials
- Deploys MySQL StatefulSet + Service from Jinja2 template
- Deploys internal HAProxy for MySQL (`mysql-ha:3307`)

### Role: `zabbix`

- Deploys Zabbix Server (Deployment + Service) from template
- Deploys Zabbix Web (Deployment + Service)
- Zabbix connects to MySQL through `mysql-ha`

### Role: `haproxy` (external)

- Deploys HAProxy for Zabbix with Service type `LoadBalancer`
- Routes:
  - HTTP `:80` → Zabbix Web
  - TCP `:10051` → Zabbix Server

### Role: `grafana`

- Deploys Grafana
- Installs Zabbix plugin (`alexanderzobnin-zabbix-app`)
- Auto-provisions Zabbix datasource
- Loads starter dashboard: **“Zabbix / MySQL / HA Cluster Overview”**

---

## 🚀 Usage

From repo root:

### Dev

```bash
ansible-playbook -i inventories/dev/hosts deploy-zabbix-eks.yml
```

### Stage

```bash
ansible-playbook -i inventories/stage/hosts deploy-zabbix-eks.yml
```

### Prod

```bash
ansible-playbook -i inventories/prod/hosts deploy-zabbix-eks.yml
```

The playbook:

```yaml
# deploy-zabbix-eks.yml
- name: Deploy Zabbix stack on EKS
  hosts: eks_control
  gather_facts: false

  pre_tasks:
    - name: Ensure namespace exists
      kubernetes.core.k8s:
        kubeconfig: "{{ kubeconfig_path }}"
        context: "{{ kube_context | default(omit) }}"
        state: present
        definition:
          apiVersion: v1
          kind: Namespace
          metadata:
            name: "{{ zabbix_namespace }}"

  roles:
    - role: mysql
    - role: zabbix
    - role: haproxy
    - role: grafana
```

---

## 🔍 Verifying the Deployment

Check pods and services in your namespace (example: dev):

```bash
kubectl get pods,svc -n zabbix-dev
```

You should see:

- Pods: `mysql`, `mysql-haproxy`, `zabbix-server`, `zabbix-web`, `zabbix-haproxy`, `grafana`
- Services:
  - `mysql`, `mysql-ha`
  - `zabbix-server`, `zabbix-web`, `zabbix-ha`
  - `grafana`

---

## 🌐 Accessing Zabbix and Grafana

### Zabbix Web UI

```bash
kubectl get svc zabbix-ha -n zabbix-dev
```

Open in browser:

```text
http://<zabbix-ha-external-hostname>/
```

Default credentials (if unchanged):

- Username: `Admin`
- Password: `zabbix`

> Change these immediately in production.

### Grafana

```bash
kubectl get svc grafana -n zabbix-dev
```

Open:

```text
http://<grafana-external-hostname>:3000
```

Login with credentials from `group_vars/all.yml` (e.g. `admin / DevGrafanaPass123!`).

The starter dashboard appears as:

- **“Zabbix / MySQL / HA Cluster Overview”**

---

## 🛠 Customization Ideas

- Replace in-cluster MySQL with **Amazon RDS MySQL (Multi-AZ)**  
- Add **TLS termination** using Ingress or HAProxy SSL
- Enable **HPA (Horizontal Pod Autoscaler)** for Zabbix, HAProxy, Grafana
- Integrate **Prometheus & exporters** alongside Zabbix
- Add **NetworkPolicies** to restrict traffic
- Use **AWS Secrets Manager** instead of Kubernetes Secrets

---

## ⚠️ Disclaimer

This project is a **reference implementation / starter kit**, not a fully hardened enterprise solution.

Before using in production, you should:

- Use a proper HA database (RDS Multi-AZ or MySQL cluster)
- Enforce TLS everywhere
- Implement backup and restore strategies
- Harden RBAC, IAM, and network policies
- Review resource requests/limits and autoscaling policies

---

## 🔁 GitHub Actions Workflow

A sample CI/CD workflow is provided in:

```text
.github/workflows/deploy.yml
```

It:

- Checks out repo
- Installs Python + Ansible
- Configures AWS credentials
- Updates kubeconfig
- Runs Ansible playbook for `prod`

Adjust for your needs before using in production.

---

## 🌍 Terraform Version (Optional)

A minimal Terraform example is provided in `terraform/`:

- `main.tf` – provider and base config
- `vpc.tf` – VPC and subnets using a community module
- `eks.tf` – EKS cluster and managed node group
- `outputs.tf` – key outputs

You can:

1. `cd terraform`
2. `terraform init`
3. `terraform apply`
4. Then run the Ansible playbook pointing to the new EKS cluster.

This keeps infra (Terraform) and app deployment (Ansible) separate but coordinated.

---

Happy monitoring! 🎯
